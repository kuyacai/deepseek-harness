# DeepSeek Harness 插件机制：从装载到调用

本文记录 DSH（DeepSeek Harness）插件的**装配链路与调用链路**：一次用户输入之后，插件究竟是怎么被"用到"的；以及多个插件并存时的冲突规则。
注意：本文所有路径与行号基于**本机实测**；源码锚点以本地仓库 `webnovel-writer-8`（v8 分支，`0.1.0-preview.6`）为准，换版本行号可能移动（见 §8）。

配套阅读：`session-logs.md`（会话事件日志与「轨迹」）。本文只讲机制，**不重复**日志格式与查阅命令。

---

## 1. 一句话结论

DSH 是**「注册表 + 循环驱动器」**，不是调度器。

- 插件**不被路由**：进程启动时它就已经装好了，不存在"DSH 判断该调用哪个插件"这一步。
- 插件做的事是**把自己放进模型的可见面**（工具清单、技能清单、系统提示词）。
- 模型在运行时从可见面里**自己选**要用的能力；DSH 只按返回的名字**查表派发**。

---

## 2. 启动装载链（插件如何进场）

以本机 `scriptor` profile 装载 V8 插件为例，实测链路如下：

```
~/.dsh/profiles/scriptor/package.json
  └─ dsh.profile.bundles: ["@deepseek-ai/dsh-base", "@deepseek-ai/dsh-web-app",
                           "@linfengqaqtat/dsh-scriptor"]
       └─ 该包自己的 package.json
            ├─ dsh.bundle.patch: ./cordis.patch.yml
            │    └─ insert: { id: webnovel, name: '@linfengqaqtat/dsh-scriptor' }
            └─ main: ./lib/index.js        ← 真正被加载的入口文件
                 └─ export function apply(ctx)   ← 入口函数
```

三段证据：

| 环节 | 实测内容 |
| --- | --- |
| profile 入口表 | `~/.dsh/profiles/scriptor/cordis.yml` 的内容就是一个 `[]`；注释写明：树由 patches 组合而成——先各 bundle，再 `cordis.patch.yml`，再 `--patch` 覆盖层 |
| bundle 清单 | `~/.dsh/profiles/scriptor/package.json` 的 `dsh.profile.bundles` 三个条目（本机装的是 `0.1.0-preview.7`） |
| 包自述 | `packages/bundle/package.json` 的 `main` / `exports` / `dsh.bundle.patch` / `dsh.client`；`packages/bundle/cordis.patch.yml` 全文只有 3 行：`insert: [{ id: webnovel, name: '@linfengqaqtat/dsh-scriptor' }]` |

> 别把两份 `cordis.patch.yml` 混为一谈：profile 目录里那份是**用户的**补丁层（本机只覆盖了一条欢迎语版本）；插件包里那份是**包自带的**补丁层，负责把插件自己插进 Cordis 入口表。

入口文件 `packages/bundle/src/index.ts` 只导出两样关键东西：

- `export const name = 'webnovel-bundle'`（第 38 行）
- `export function apply(ctx: Context)`（第 309 行）

`apply()` 的第一行就是 `ctx.logger.info('工作台插件已加载（webnovel bundle）')`（第 310 行）——启动日志里看到这句，说明装载成功。

---

## 3. apply() 装了什么（四条路）

`apply` 里**没有一行处理用户输入**，它全部在做"往宿主的各个注册表里塞东西"：

| 路 | 源码锚点 | 塞进哪 | 效果 |
| --- | --- | --- | --- |
| 工作范围 | `index.ts:451–470` | `workspaceRegistry`（经 inject 声明，不是启动时快照） | 认领工作区根；headless 下退回进程 cwd |
| 技能 | `index.ts:311–324` | `ctx.skills` 注册 provider（`providerName: 'webnovel-bundled'`、`bundledSkillDir: ../skills/`、`includeDefaultRoots: false`） | 随包 10 个 `novel-*/SKILL.md` 进入技能清单 |
| 工具 | `index.ts:357–407` | 主 Agent scope 的 `tools.register`（逐个注册，注销函数收进 `agentToolOffs`） | 27 个 `novel_*` 工具出现在模型的工具清单里 |
| 提示词 | `index.ts:518–528` | `systemPrompt.section`（persona 走官方槽位 `deployment:persona-prefix`）+ `systemPrompt.context`（书架状态） | 中文适配约定与书架总览常驻系统提示词 |

另有客户端半边：`packages/bundle/package.json` 的 `dsh.client`（`platform: web` + 4 个 inject 依赖）指向 `lib/client.js`，浏览器加载后注册右侧栏面板与工作区里的"书房"入口。**它不经模型，与调用链无关。**

工具注册是**按 agent** 的（`installFor`，`index.ts:496–540`）：

- **主 Agent**：注册全部 `novel_*` + persona + 书架状态；
- **子 Agent**：反向操作——`tools.restrict({ deny: [全部 novel_*] })` 收掉继承来的工具面（`index.ts:430–444`），并用 `approval.setPolicy(agent, 'never')` 让授权请求自动拒绝。

---

## 4. 一次请求的完整循环

```
你按下发送
  │
  ├─① 组装一次请求：系统提示词（persona / 书架状态已在其中）+ 历史消息 + 工具清单
  │        ↑ 插件的能力此刻已经"在场"，无需任何决策
  │
  ├─② 发给 LLM
  │
  ├─③ LLM 返回 assistant 消息：可能带 tool_call（名字 + 参数），也可能要求加载某个技能
  │
  ├─④ DSH 拿名字去工具注册表查执行体 → 找到就执行
  │        ↑ "按名查表"，不是"路由到插件"：
  │          novel_get_story_status 这个名字是插件早先注册进去的，所以落回插件代码
  │
  ├─⑤ 结果作为新消息回填 → 回到 ②（多轮 step 循环）
  │
  └─⑥ LLM 不再要工具、只返回文本 → 本轮结束
```

这套循环在会话事件日志里的投影，是成对的 `step/start` / `step/end` 与 `tool/call` / `tool/result`（事件名清单与实测计数见 `session-logs.md` §4；本机一个 1186 行的会话里 `tool/call` 出现 209 次，即可理解为 209 次工具往返）。

两个容易漏掉的点：

- **不是所有"插件生效"都长成工具调用。** persona 与书架状态靠系统提示词生效，全程零 `tool_call` 也照样起作用。
- **选择是逐轮重算的。** 每轮都从当前可见面重新决策，历史里没有硬约束把上一轮的选择锁住——这是 §6 全部风险的前提。

---

## 5. 模型看到的可见面

工具注册时只交了五个字段：`name` / `description` / `parameters` / `output` / `execute`（`index.ts:395–401`）。**没有"我来自哪个插件"这一项**，技能清单同理。

宿主侧工具解析的产物是一个 `Map<名字, 定义>`（`packages/core/tools/src/index.ts:1192–1208`）。也就是说：

> 模型眼里是一堆**平铺的名字 + 描述**，插件归属不可见；它只能靠描述措辞判断该用谁。

这是 §6 全部风险的根源。

---

## 6. 多插件并存时的三类撞车

### 6.1 工具同名 → 静默覆盖，不会误调用

可见面是 `Map<name, definition>`，同一个名字只会有一个条目：

- **近层覆盖远层**：scope 自己的注册最后写入，覆盖任何继承来的同名条目（`tools/src/index.ts:1184–1209`）；
- **同层内后注册覆盖先注册**（**由 Map 组装顺序推演，未实跑验证**，见 §8）。

结果比"误调用"更隐蔽——**你以为装了两个，其实有一个的工具压根没露面**。

### 6.2 技能同名 → 确定性去重 + warning

技能服务按层取近者；同一层内：

- **rank 小的赢**（`rank` 语义见 `packages/skill/skill/src/index.ts:78–79`）；
- rank 相同则按**注册顺序 first-wins**；
- 落败者**打一条 warning 并拿到 no-op disposer**（同上 `:432–435`），因此它卸载时不会误删赢家。

本机实测的 rank 常量：`RUNTIME_RANK = 250`、`BUNDLED_SKILL_RANK = 600`（`skill/src/index.ts:25`、`:28`）。两个插件都用 `bundledSkillDir` 挂载，**同属 bundled 层、rank 都是 600**，于是胜负由注册顺序决定——谁先被装载谁赢。

### 6.3 名字不同但语义相近 → 这才真会误调用

这时两个候选**并列出现在可见面上**，宿主不排序、不投票，纯看模型对描述的理解。两种表现：

- **逐轮漂移**：turn 1 用 A、turn 2 用 B（选择逐轮重算，见 §4）；
- **试挫振荡**：A 报错 → 模型改投 B → B 也报错 → 来回横跳；轨迹里会看到一堆意图相同、名字不同的 `tool/call`。

### 6.4 本机两个插件的实测对比

| 层 | `@dsh-novel/plugin`（自研） | `@linfengqaqtat/dsh-scriptor`（V8） | 结论 |
| --- | --- | --- | --- |
| 工具 | 1 个：`novel_check_growth` | 27 个：`novel_create_book`、`novel_settle_chapter`… | **无重名**，但共用 `novel_` 前缀、同域 |
| 技能 | `novel-drafting`、`novel-review`、`novel-prep`、`novel-volume-outline`、`novel-chapter-beats`、`novel-growth-check` | `novel-drafting`、`novel-review`、`novel-director`、`novel-design`… | **`novel-drafting`、`novel-review` 重名**（走 §6.2，静默覆盖） |
| 书仓口径 | `大纲/成长线.json`、`定稿/卷NN/NNNN-标题.md` | `构想` / `作品契约` / `大纲` / `世界书` / `定稿` | 互不认识——工具名不撞，但**可能被拿错去操作对方的书仓**，触发 §6.3 振荡 |

---

## 7. 三条对策

| 手段 | 做法 | 效果 |
| --- | --- | --- |
| **分 profile 装** | 每个 profile 只装一套（本机现状） | 插件互不可见，三类风险一并归零 |
| **名字加前缀** | 工具与技能名带插件标识（如 `dnv_*` vs `novel_*`） | 把"语义相近"变成"一眼可辨" |
| **按 agent 收面** | `tools.restrict({ deny: [...] })` | V8 已用此招对子 Agent 全封小说工具，同法可关掉某一方 |

---

## 8. 未核实项（不要当结论用）

- **同层同名工具的实际胜负**：§6.1 的"后注册覆盖先注册"由 Map 组装顺序读出，**未实跑验证**（本次明确不做实测）。
- **profile 侧 patch 的合并细节**：`cordis.yml` 注释说明顺序是「各 bundle → `cordis.patch.yml` → `--patch`」，但 DSH 收集各 bundle 自带 patch 的具体实现**未逐行核对**。
- **版本漂移**：本机 profile 装的是 `0.1.0-preview.7`，而本文引用的源码仓库为 `0.1.0-preview.6`；行号可能在小版本间移动。
- **两个重名技能的真实归属**：只确认 `novel-drafting` / `novel-review` 在两侧都存在，**未构造同装环境验证**最终活下来的是哪一个。

---

## 9. 一句话备忘

> 插件不是被"调用"的对象，而是**开机时注册能力、运行时被按名派发**；
> 多插件共存的风险不在"选错插件"，而在**同名被静默吃掉**与**异名同义被逐轮漂移**。