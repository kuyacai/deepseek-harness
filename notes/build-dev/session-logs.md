# DeepSeek Harness 会话轨迹与事件日志说明

本文档记录 DSH 里「**对话**」「**轨迹**」两个 tab 背后的数据究竟存在哪里、以什么格式存放，以及如何直接查阅。
注意：本文所有路径与统计均基于**本机（`renpingwudeMacBook-Pro`）**的实际观测；工作区转义名与会话 id 是环境相关的，换机器/换工作区会对不上。

---

## 1. 一句话结论

「轨迹」**不是**独立的日志文件，而是**会话事件日志的一个视图（projection）**。

- 唯一真源：`~/.dsh/sessions/` 下每个会话一份的 **`session.v4.jsonl.zstd`**。
- 「对话」和「轨迹」都从这份事件流组装，区别只在呈现方式（聊天气泡 vs 按轮次的事件记录表 + 时间线）。
- 按 `ui-trajectory` 包的自述，轨迹视图是**纯投影**：它既不读取也不改变对话快照。

---

## 2. 存放位置

### 2.1 真源：会话事件日志

```
~/.dsh/sessions/<工作区转义路径>/<会话id>/
├── session.v4.jsonl.zstd     # 事件日志（zstd 压缩的 JSONL）
└── session.lock              # 会话锁
```

**一个工作区一个目录**，目录名由工作区的绝对路径转义而来。实际观测到的两个例子：

| 工作区绝对路径 | 转义后的目录名 |
|---|---|
| `/Users/renpingwu/Documents/workspace/webnovel-feisheng` | `--Users-renpingwu-Documents-workspace-webnovel-feisheng--` |
| `/Users/renpingwu/Documents/deepseek-harness/默认工作区` | `--Users-renpingwu-Documents-deepseek-harness-~9ED8~8BA4~5DE5~4F5C~533A--` |

从这两个样本可推得转义规则（**由观测归纳，未逐字核对源码**）：

- 路径分隔符 `/` → `-`；
- 非 ASCII 字符 → `~XXXX~`，其中 `XXXX` 是该字符 UTF-16 码元的十六进制（如「默」U+9ED8 → `~9ED8~`）。

> 实践中不必自己算转义名——直接 `ls ~/.dsh/sessions/` 按修改时间挑目录即可（见 §9）。

### 2.2 投影缓存（不是真源）

```
~/.dsh/storages/session_projcache/sessions/<会话id>.json
```

这是给客户端读取用的会话投影缓存（索引/摘要性质）。**不要把它当真源**：要核对事实一律回到 `session.v4.jsonl.zstd`。

### 2.3 顺带记录：服务端其他位置

| 位置 | 内容 |
|---|---|
| `~/.dsh/storages/workspace.json` | 工作区列表（全局，跨 profile 共享） |
| `~/.dsh/.credentials.yaml` | 凭据（全局共享，如 `DEEPSEEK_API_KEY` 的引用） |
| `~/.dsh/profiles/<profile>/cordis.yml` | profile 入口表（出厂为空列表，树由 bundle patch 组合） |
| `~/.dsh/logs/` | 本次观测中**为空**——传输层错误没有在这里留档 |

---

## 3. 文件格式

**zstd 压缩的 JSONL**：解压后一行一个 JSON 事件。首个记录是会话元数据：

```json
{"type":"session","version":4,"id":"session-e25482d0-e5ca-4a04-8c50-d6738e5762e8","createdAt":1790776108157,"cwd":"/Users/renpingwu/Documents/workspace/webnovel-feisheng","isSeeded":false,"delegationDepth":0,"agentPreset":"standard"}
```

普通事件的统一外壳：

| 字段 | 含义 |
|---|---|
| `type` | 事件类型（见 §4） |
| `seq` | 全局递增序号 |
| `time` | 毫秒级 Unix 时间戳 |
| `data` | 事件负载，随类型而异；常见含 `turn` / `step` |

---

## 4. 事件类型清单

以本机 `webnovel-feisheng` 工作区的一次实际会话为样本（压缩 1.7 MB，解压 **1186 行**），出现最多的 20 类如下：

| 事件类型 | 条数 | 含义 |
|---|---:|---|
| `tool/call` | 209 | 工具调用（轨迹里的工具行） |
| `tool/result` | 209 | 工具结果 |
| `step/start` | 174 | 步骤开始 |
| `session-log-deepseek/delivery-accepted` | 174 | 传输层回执（请求被接受） |
| `step/end` | 174 | 步骤结束 |
| `assistant/message` | 173 | 模型输出 |
| `agent/inbox/spliced` | 12 | 中途引导（steering）拼接 |
| `user/message` | 10 | 用户输入 |
| `subagent/catalog` | 6 | 子代理目录 |
| `assistant/attempt` | 6 | 单次请求尝试 |
| `llm/retry` | 5 | 重试结果 |
| `llm/retry-started` | 5 | 重试开始 |
| `turn/start` | 4 | 轮次开始 |
| `turn/end` | 4 | 轮次结束 |
| `request/header` | 3 | 该次请求的历史头（生效的系统提示词、工具目录） |
| `session/title` | 2 | 会话标题变更 |
| `goal/change` | 2 | 目标变更 |
| `approval/asked` | 2 | 发起审批 |
| `approval/decided` | 2 | 审批结论 |
| `workspace/changes` | 2 | 工作区文件变更 |

> 上表合计 1178 行，与总行数 1186 相差 8 行——这 8 行属其余低频类型（未逐条展开）。

---

## 5. 样例事件（原样摘录）

```json
{"type":"tool/call","seq":20,"time":1790776147124,"data":{"turn":1,"step":1,"callId":"call_00_ET_muyKPmM3PqePKWznhaNN9611","name":"skill","arguments":"{\"name\": \"novel-director\"}"}}
```

```json
{"type":"step/end","seq":24,"time":1790776147291,"data":{"turn":1,"step":1}}
```

```json
{"type":"llm/retry","seq":929,"time":1790779694346,"data":{"retryId":"03442cb7-e22b-4496-bd9f-10f0e7af702e","turn":3,"step":13,"provider":"deepseek-official","mode":"normal","policyKey":"[\"normal\",5,[\"EMPTY_RESPONSE\",\"RATE_LIMIT\",\"SERVER\",\"TIMEOUT\",\"TRANSPORT\"],500,10000,0.1]","retry":1}}
```

---

## 6. 用它定位一次失败

### 6.1 复现背景

在 DSH 里执行写作任务时曾出现：

```
本轮运行失败：DeepSeek Messages transport failed
重试延迟：7400毫秒
失败原因：DeepSeek Messages transport failed
```

### 6.2 从日志里读到什么

对应的 `llm/retry` 事件给出了完整上下文：

| 字段 | 值 | 含义 |
|---|---|---|
| `provider` | `deepseek-official` | 失败发生在与 DeepSeek 官方接口的传输层 |
| `turn` / `step` | `3` / `13` | 精确定位到第 3 轮第 13 步 |
| `retry` | `1` | 第 1 次重试 |
| `policyKey` | `["normal",5,["EMPTY_RESPONSE","RATE_LIMIT","SERVER","TIMEOUT","TRANSPORT"],500,10000,0.1]` | 重试策略 |

`policyKey` 逐项解读：

1. `"normal"` —— 策略模式；
2. `5` —— 最多重试 5 次；
3. `["EMPTY_RESPONSE","RATE_LIMIT","SERVER","TIMEOUT","TRANSPORT"]` —— 触发重试的错误类别（**`TRANSPORT` 正在其中**）；
4. `500` —— 基础退避延迟 500 ms；
5. `10000` —— 退避上限 10000 ms；
6. `0.1` —— 抖动系数。

界面上的「重试延迟：7400 毫秒」即该退避算法算出的等待时间。

### 6.3 结论性判断（当时实测所得）

该次失败是**到 `api.deepseek.com` 的网络间歇性抖动**，不是配置或插件问题。证据：同一条探测请求首字节时延在 **11.26 s** 与 **0.53 / 0.63 / 0.81 s** 之间跳变（对照 `api.github.com` 为 2.33 s）；`web` 与 `scriptor` 两个 profile 的 `cordis.yml` 完全一致，全局 `DEEPSEEK_API_KEY` 引用存在。

**处理办法**：网络恢复后直接在原会话重发消息即可，无需重新建书——会话状态与书仓都在磁盘上。

---

## 7. 三个容易混淆的存储

不要把下面三者混为一谈：

| 名称 | 位置 | 是什么 |
|---|---|---|
| **DSH 轨迹 / 事件日志** | `~/.dsh/sessions/**/session.v4.jsonl.zstd` | 宿主层会话事件流；「对话」「轨迹」都读它，**所有会话都有** |
| **V8 插件事务记录** | `<书仓>/.webnovel/operations/tool-call_*.json`、`<书仓>/.webnovel/transactions/` | 网文插件自写的操作与事务记录（供回滚用），**不是宿主轨迹** |
| **写作材料溯源** | `<书仓>/草稿区/材料包/<章>/10-来源与版本清单.md` | 「本章用了哪份材料的哪个版本」，属业务产物 |

---

## 8. 「轨迹」tab 的机制（要点）

引自 `packages/client/ui-trajectory/README.zh.md`：

- 视图是**纯投影**：从共享会话窗口组装业务记录，**既不读取也不改变 Chat 会话快照**；因此删改视图不影响会话历史，反之日志才是唯一真源。
- 它把**用户 / 助手 / 工具 / 嵌套子工具 / 压缩（compaction）**记录分组，标示轮次与步骤边界。
- 选中记录可看 token 用量、耗时、输入、输出、计时、图片与附件摘要；助手记录还能显示 TTFT（首 token 时间）与吞吐量。
- 进行中的记录**只显示开始标记，不虚构耗时**。
- 工具记录从持久的 `tool/call` 事件起算（带完整参数）；Chat 的瞬态 preparing 阶段不产生轨迹工具行。

进一步阅读（DSH 源码）：

- `packages/client/ui-trajectory/` —— 本视图
- `packages/session/session-projection/` —— 面向客户端的会话状态投影注册表
- `packages/core/session/` —— 持有原始事件的会话 seam
- `packages/compaction/compaction/` —— 压缩 seam（其请求也出现在轨迹里）

---

## 9. 命令行查阅

### 9.1 定位最近的会话文件

```bash
ls -lt ~/.dsh/sessions/ | head        # 找最近活跃的工作区目录
find ~/.dsh/sessions --name 'session.v4.jsonl.zstd' -print0 \
  | xargs -0 ls -lt | head            # 直接按时间列全部会话日志
```

### 9.2 本机可用工具（已实测）

| 工具 | 本机状态 |
|---|---|
| `zstd`（命令行） | **有** |
| Python `zstandard` 模块 | **有** |
| `jq` | **无**（若要用 jq 写法需先 `brew install jq`） |

因此下文以 **zstd + Python** 为主；`jq` 写法附在 §9.4 供已安装者参考。

### 9.3 解压并筛选（zstd + Python，已实测可跑）

先解压到临时文件，再用 Python 查询：

```bash
F=~/.dsh/sessions/--Users-renpingwu-Documents-workspace-webnovel-feisheng--/session-e25482d0-e5ca-4a04-8c50-d6738e5762e8/session.v4.jsonl.zstd
zstd -dc "$F" > /tmp/session.jsonl
```

**A. 事件类型分布**

```bash
python3 - /tmp/session.jsonl <<'PY'
import json, sys, collections
events = [json.loads(l) for l in open(sys.argv[1]) if l.strip()]
for k, v in collections.Counter(e.get('type') for e in events).most_common():
    print(f'{v:6d}  {k}')
PY
```

**B. 只看重试 / 传输失败**

```bash
python3 - /tmp/session.jsonl <<'PY'
import json, sys
for l in open(sys.argv[1]):
    if not l.strip(): continue
    o = json.loads(l)
    if o.get('type') == 'llm/retry':
        print(json.dumps(o, ensure_ascii=False))
PY
```

**C. 工具调用次数排行**

```bash
python3 - /tmp/session.jsonl <<'PY'
import json, sys, collections
events = [json.loads(l) for l in open(sys.argv[1]) if l.strip()]
c = collections.Counter(e['data']['name'] for e in events if e.get('type') == 'tool/call')
for k, v in c.most_common():
    print(f'{v:6d}  {k}')
PY
```

**D. 把时间戳还原成可读时间**

```bash
python3 - /tmp/session.jsonl <<'PY'
import json, sys, datetime
for l in open(sys.argv[1]):
    if not l.strip(): continue
    o = json.loads(l)
    if o.get('type') == 'tool/call':
        t = datetime.datetime.fromtimestamp(o['time']/1000).strftime('%Y-%m-%d %H:%M:%S')
        print(t, o['data']['name'])
PY
```

> 只解压一次、复用 `/tmp/session.jsonl`，避免反复解压大文件。

### 9.4 jq 写法（本机需先安装 jq）

```bash
zstd -dc "$F" | jq -r '.type' | sort | uniq -c | sort -rn
zstd -dc "$F" | jq 'select(.type=="llm/retry")'
zstd -dc "$F" | jq -r 'select(.type=="tool/call") | .data.name' | sort | uniq -c | sort -rn
zstd -dc "$F" | jq -r 'select(.type=="tool/call") | "\(.time/1000|todate)  \(.data.name)"'
```

---

## 10. 未核实项（不要当结论用）

- **保留 / 轮转策略**：本次未在配置中找到清理或上限设置，会话日志看起来**只增不减**。是否有上限、能否导出或清理，**尚未核实**。
- **转义规则的精确实现**：§2.1 的规则由两个样本归纳，未逐行核对源码；含空格或其他特殊字符的路径可能另有处理。
- **`session_projcache` 的内部字段**：只确认它是投影缓存、不是真源，字段结构未展开。
- **`~/.dsh/logs/` 为空**是本次观测结果，不代表其他环境或版本下始终为空。

---

## 11. 一句话备忘

> 要查「Agent 到底干了什么」→ 去 `~/.dsh/sessions/**/session.v4.jsonl.zstd`（zstd 解压后是 JSONL 事件流）；
> 「轨迹」tab 只是它的一个视图，不是另一份日志。