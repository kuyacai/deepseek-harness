# DeepSeek Harness 插件开发笔记

## 一、官方插件目录结构简介

`packages/` 目录下的子目录是 DeepSeek Harness 按**功能领域分组**的 workspace 包，每个分组代表一个插件家族。这些分组本身只是组织方式，不是强制性的插件放置规则。

主要分组及其职责（来自官方 `packages/README.md`）：

| 分组 | 职责 |
|------|------|
| `core/` | 产品核心：会话、提示词、工具、Agent 服务及主循环 |
| `api/` | 远程 BFF 组装和 Typert RPC 网关 |
| `llm/` | LLM 能力家族：抽象服务 + 提供商适配器 |
| `subprocess/` | 子进程能力家族 |
| `shell/` | Bash 能力家族 |
| `terminal/` | 持久化 PTY 能力家族 |
| `sandbox/` | 进程隔离层 |
| `fs/` | 文件系统能力家族 |
| `skill/` | 技能能力家族 |
| `subagent/` | 子代理能力家族 |
| `jobs/` | 通用后台任务运行时 |
| `web/` | Web 能力家族：搜索/获取提供商 |
| `client/` | Web UI 相关插件（`dsh-client-ui-*` 系列） |
| `extensions/` | Agent 运行时自修改：动态插件挂载/卸载 |
| `bundle/` | 可安装的 `dsh --profile` patch 层 |

**关键理解**：整个 DSH 的核心理念是 **“一切皆插件”**。模型适配器、工具注册表、会话日志、Agent Loop、甚至前端 UI 组件本身都是 Cordis 插件。扩展 DSH 的方式是把插件挂载到其他插件旁边，**不需要修改所谓的“核心”**。


## 二、插件的两种运行面

在编写任何插件之前，必须先判断它运行在**哪一面**：

| 运行面 | 适用场景 | 入口 |
|--------|----------|------|
| **Host（Node 端）** | 工具、系统提示词、HTTP 路由、持久化、provider | `src/index.ts` |
| **Client（浏览器端）** | Slot 注册、Conversation Node、浏览器状态、UI 浮层 | `src/client/index.tsx` |
| **Host + Client** | 需要 Web 可视化的宿主能力 | 两者都有 |

**没有 Web 需求就不要声明 `dsh.client`，也不要构建 client bundle**。


## 三、最小插件骨架

### 3.1 纯函数形式（最常用）

```typescript
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-plugin'
export const inject = ['tools']  // 声明依赖

export function apply(ctx: Context) {
  console.log('[my-plugin] loaded!')
  // 通过 ctx 注册能力
}
```

**要点**：
- `name` 是插件唯一标识
- `apply(ctx)` 是框架加载时调用的入口
- `inject` 声明依赖的服务，框架会**等到这些服务就绪后才执行 apply**
- 通过 `ctx` 注册的任何东西（事件监听、工具、定时器）在插件卸载时**自动清理**

### 3.2 带配置的插件

```typescript
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export interface Config {
  greeting: string
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
})

export function apply(ctx: Context, config: Config) {
  console.log(config.greeting)
}
```

**原则**：可调参数一律进 Config，不硬编码；配置非法要让加载响亮失败。

### 3.3 对象形式与类形式

```typescript
// 对象形式
export default {
  name: 'my-plugin',
  inject: ['tools'],
  apply(ctx: Context) { /* ... */ },
}

// 类形式（当需要向其他插件提供服务时）
import { Service } from '@deepseek-ai/cordis'

export default class MyService extends Service {
  static inject = ['tools']
  constructor(ctx: Context) {
    super(ctx, 'myService')
  }
}
```

大多数情况下**函数形式足够了**。类形式仅在向其他插件提供服务时使用。


## 四、注册工具（最常用的扩展方式）

工具是 Agent 可调用的能力，通过 `ctx.tools.register()` 注册：

```typescript
import { defineTool } from '@deepseek-ai/dsh-tools'

ctx.tools.register(defineTool({
  name: 'greet',
  description: 'Greet someone by name.',
  parameters: {
    name: { type: 'string', required: true, description: 'Who to greet.' },
  },
  output: {
    schema: { type: 'string' },
    render: (_args, value) => [{ type: 'text', text: value }],
  },
  async execute(args, exec) {
    return `Hello, ${args.name}!`
  },
}))
```

**关键约定**：
- `parameters` 推断并校验参数
- `output.schema` 校验返回值，`output.render` 把返回值变成模型看到的内容
- object 输出 schema **必须写 `additionalProperties: true`**，否则注册直接失败
- `execute` 抛异常 = `isError`；`exec.signal` / `exec.agent` 可用


## 五、调试：本地插件通过 `--patch` 覆盖层加载

这是开发阶段**最快**的验证方式，不需要打包、不需要安装：

### 5.1 创建覆盖层文件

```yaml
# scratch-plugin/cordis.yml
- insert:
    - id: hello
      name: '/absolute/path/to/deepseek-harness/scratch-plugin/src/my-plugin.ts'
```

**插件路径必须是绝对路径**。

### 5.2 启动

```bash
pnpm dsh web --patch ./scratch-plugin/cordis.yml
```

打开 `http://127.0.0.1:3080`。启动时终端会打印 `[hello-plugin] plugin loaded!`。

### 5.3 验证要点

- 终端出现 `plugin loaded` 只证明**插件被加载并执行了 apply()**
- 对话出现 `Tool call · text_stats` 才证明**能力注册成功并可被模型使用**
- 如果插件只打印日志，检查日志即可；如果提供 Service，调用 Service 验证


## 六、发布为可安装 Bundle

当插件需要对外分发时，打包成 **bundle**：

### 6.1 目录结构

```
hello-plugin/
├── package.json       # 声明 dsh.bundle
├── cordis.patch.yml   # 安装时应用的层
└── index.js           # 插件入口
```

### 6.2 package.json

```json
{
  "name": "dsh-hello-plugin",
  "type": "module",
  "main": "index.js",
  "files": ["index.js", "cordis.patch.yml"],
  "dsh": {
    "bundle": {
      "patch": "./cordis.patch.yml"
    }
  }
}
```

### 6.3 cordis.patch.yml

```yaml
- insert:
    - id: hello
      name: 'dsh-hello-plugin'
```

**关键区别**：bundle 的 patch 按**包名**引用插件，而不是源码路径，这样 Node 的模块解析才能找到已安装的代码。

### 6.4 安装到 profile

```bash
dsh plugin --profile web add ./hello-plugin
```

卸载：

```bash
dsh plugin --profile web remove dsh-hello-plugin
```


## 七、插件形态判断速查

| 你想要… | 写什么 | 关键 API |
|---------|--------|----------|
| 增加一个模型可调用动作 | 工具插件 | `ctx.tools.register(defineTool({...}))` |
| 接入新模型提供商 | LLM Adapter | `ctx.llm.registerAdapter()` |
| 提供 HTTP 端点 | Host Service | `ctx.webServer.register()` |
| 向其他插件提供服务 | 类形式（extends Service） | `super(ctx, 'serviceName')` |
| 添加设置卡片 | 设置插件 | `installSettingsSection()` |
| 在 Web UI 插入组件 | Client 插件 | Slot 注册 |

**判断标准**：只增加一个模型可调用动作，优先写工具插件；需要接入新的模型提供商，写 LLM Adapter；需要把多个插件和默认配置一起交付，再封装成 Bundle。


## 八、参考资料

- 官方开发教程：`docs/user/develop/basic/index.zh.md`
- 工具开发：`docs/user/develop/basic/tool.zh.md`
- 插件配置：`docs/user/develop/basic/config.zh.md`
- 打包与安装：`docs/user/develop/basic/publish.zh.md`
- 官方包目录说明：`packages/README.md`
- 架构文档：`docs/architecture.md`
- 服务与依赖：`docs/user/develop/framework/service.zh.md`

---
