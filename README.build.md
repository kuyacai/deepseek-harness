# DeepSeek Harness 编译与运行说明（Intel Mac / darwin-x64）

本文档记录在 **Intel Mac（darwin-x64）** 上从源码编译并运行 `deepseek-harness` 的完整流程，包含环境准备、编译、产物校验、启动与常见问题排查。
注意，这份文档仅仅记录在`renpingwudeMacBook-Pro`设备上的环境进行编译并运行的操作步骤。
---

## 1. 环境要求

| 项目 | 要求 | 说明 |
|------|------|------|
| 操作系统 | macOS（darwin） | Intel（x86_64）或 Apple Silicon 均可 |
| CPU 架构 | `x86_64`（Intel Mac） | 已验证 darwin-x64 可编译运行 |
| Node.js | `^22.19.0 \|\| >=24.0.0` | **不要用 Node 23**，奇数版本项目不支持 |
| pnpm | 项目锁定版本（如 11.7.0） | Intel Mac 上 pnpm 11.7.0 无原生二进制，需绕过或改用系统 pnpm |
| 工作目录 | `~/Documents/workspace/deepseek-harness` | 以实际路径为准 |

### 1.1 确认 Node 版本

```bash
node -v
```

应输出 `v22.x` 或 `v24.x`。

例如当前输出结果为：
```bash
(base)  deepseek-harness % node -v
v22.23.3
```

若为 `v23.x`，请切换：

```bash
nvm install 22
nvm use 22
node -v
```

### 1.2 确认 CPU 架构

```bash
uname -m
node -p "process.arch"
```

Intel Mac 应输出 `x86_64` / `x64`。

---

## 2. 安装依赖

```bash
cd ~/Documents/workspace/deepseek-harness
pnpm install
```

### 2.1 如果 pnpm 版本切换失败

Intel Mac 上可能出现：

```
ERR_PNPM_PNPM_ENGINE_NO_NATIVE_BINARY:
Cannot run @pnpm/exe@11.7.0 on this host:
it ships no native binary for darwin-x64.
```

绕过方法（在项目根目录 `.npmrc` 中加入）：

```ini
manage-package-manager-versions=false
package-manager-strict=false
```
并修改package.json文件中 packageManager 的值：

```json
"packageManager": "pnpm@12.6.0",
```
或临时使用环境变量：

```bash
export npm_config_manage_package_manager_versions=false
export npm_config_package_manager_strict=false
pnpm install
```

也可以直接把系统 pnpm 升到较新版本（如 12.x）后使用。

---

## 3. 编译

```bash
cd ~/Documents/workspace/deepseek-harness

# 可选：清理旧产物，避免脏状态
pnpm run clean

# 完整编译（宿主端 + 客户端 + Web 前端）
pnpm run build 2>&1 | tee build.log
```

### 3.1 判断编译是否成功

编译成功的**关键标志**是日志最后出现：

```
build: recorded 263 client artifact(s) with 3 public value(s)
```

其中数字可能因版本略有不同，但 `build: recorded ... client artifact(s)` 必须出现。

其他成功标志：

- `build: built darwin-x64/bin/system.node` —— 原生模块按 Intel Mac 正确编译
- `vite build` 输出 `dist/index.html`、`dist/assets/...`
- 没有以 `ERR_` 或 `Build failed` 结尾

### 3.2 可忽略的警告

- `[UNRESOLVED_IMPORT] Could not resolve '@deepseek-ai/dsh-app-boot'`（仅桌面端 external）
- `noExternal is deprecated`、`inlineDynamicImports option is deprecated`
- `PLUGIN_TIMINGS` 提示
- `Some chunks are larger than 500 kB`

---

## 4. 产物校验（重点）

编译后必须确认客户端产物存在，否则启动会报 `Cannot find module .../lib/client.js` 或 `.../lib/typert.host.js`。

### 4.1 核心产物检查

在项目根目录执行：

```bash
cd ~/Documents/workspace/deepseek-harness

test -s packages/client/ui-renderer/lib/client.js \
  && echo "ui-renderer OK" || echo "ui-renderer 缺失"

test -s packages/client/modules/lib/client.js \
  && echo "modules OK" || echo "modules 缺失"

test -s packages/client/hmr/lib/client.js \
  && echo "hmr OK" || echo "hmr 缺失"

test -s packages/client/ui-session/lib/client.js \
  && echo "ui-session OK" || echo "ui-session 缺失"

test -s packages/client/ui-layout/lib/client.js \
  && echo "ui-layout OK" || echo "ui-layout 缺失"
```

每个 `test -s` 的含义：

- `test -s <file>`：文件存在且**非空**则返回成功
- `&& echo "OK"`：成功时打印 OK
- `|| echo "缺失"`：失败时打印缺失

### 4.2 typert.host.js 检查

```bash
test -s packages/context/agent-instructions/node_modules/@deepseek-ai/dsh-llm/lib/typert.host.js \
  && echo "typert.host OK" || echo "typert.host 缺失"
```

### 4.3 批量检查所有客户端产物

```bash
find packages/client -name "client.js" -path "*/lib/*" | wc -l
```

正常应输出 40 左右（与构建日志中 `client artifact(s)` 数量对应）。

也可以用一行脚本逐个报告缺失：

```bash
find packages/client -type d -name lib | while read d; do
  f="$d/client.js"
  test -s "$f" && echo "OK   $f" || echo "MISS $f"
done
```

### 4.4 全部通过的样子

理想的校验输出类似：

```
ui-renderer OK
modules OK
hmr OK
ui-session OK
ui-layout OK
typert.host OK
```

如果出现 `缺失` 或 `MISS`，说明构建未完整，重新执行：

```bash
pnpm run clean
pnpm run build 2>&1 | tee build.log
```

---

## 5. 启动

```bash
cd ~/Documents/workspace/deepseek-harness
pnpm dsh web
```

启动成功后终端会输出本地服务地址（通常为 `http://localhost:xxxx`）。

### 5.1 如果启动仍报模块缺失

清理旧的 profile 缓存后重试：

```bash
rm -rf ~/.dsh/profiles/web
pnpm dsh web
```

`~/.dsh/profiles/web` 是启动失败时可能残留的 profile 状态，清理后重新生成。

---

## 6. 常见问题排查

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| `Cannot find module .../lib/client.js` | 未编译或编译未生成客户端产物 | 执行 `pnpm run build`，再做第 4 节校验 |
| `Cannot find module .../lib/typert.host.js` | 同上 | 同上 |
| `ERR_PNPM_PNPM_ENGINE_NO_NATIVE_BINARY` | pnpm 11.7.0 无 darwin-x64 二进制 | 见 2.1 |
| `Unsupported engine: wanted node ^22.19.0 \|\| >=24.0.0` | 使用了 Node 23 | 切到 Node 22 或 24 |
| `Unsupported platform: wanted cpu arm64` | Intel Mac 装到了 arm64 专用包 | 多为可选依赖，可忽略；若必需则项目不支持 Intel |
| 启动无输出、无报错 | 构建未完成或 profile 缓存损坏 | `rm -rf ~/.dsh/profiles/web` 后重试 |
| `build: recorded 0 client artifact(s)` | 构建脚本静默失败 | 检查 Node 版本，重跑 `pnpm run clean && pnpm run build` |

---

## 7. 一键编译 + 校验脚本

可保存为 `scripts/build-and-verify.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT"

echo "== Node 版本 =="
node -v

echo "== 清理 =="
pnpm run clean || true

echo "== 编译 =="
pnpm run build 2>&1 | tee build.log

echo "== 校验客户端产物 =="
for f in \
  packages/client/ui-renderer/lib/client.js \
  packages/client/modules/lib/client.js \
  packages/client/hmr/lib/client.js \
  packages/client/ui-session/lib/client.js \
  packages/client/ui-layout/lib/client.js
do
  test -s "$f" && echo "OK   $f" || { echo "MISS $f"; exit 1; }
done

test -s packages/context/agent-instructions/node_modules/@deepseek-ai/dsh-llm/lib/typert.host.js \
  && echo "OK   typert.host.js" || { echo "MISS typert.host.js"; exit 1; }

echo "== 校验通过，可执行：pnpm dsh web =="
```

赋予执行权限：

```bash
chmod +x scripts/build-and-verify.sh
./scripts/build-and-verify.sh
```

---

## 8. 交给 Agent 执行时的要点

如果需要把本编译任务交给某个 agent（如自动化流水线、CI、其他助手），请让它按顺序执行：

1. 检查 `node -v` 是否为 `v22.x` 或 `v24.x`，不是则 `nvm use 22`
2. 检查 `uname -m` 与 `node -p "process.arch"` 是否一致
3. 处理 pnpm 版本切换问题（见 2.1）
4. `pnpm install`
5. `pnpm run clean && pnpm run build 2>&1 | tee build.log`
6. 在 `build.log` 中确认存在 `build: recorded <N> client artifact(s)`
7. 执行第 4 节的所有 `test -s` 校验，任一失败则终止并报错
8. 全部通过后执行 `pnpm dsh web`

**判定成功的唯一标准**：`build.log` 出现 `build: recorded ... client artifact(s)`，且第 4 节所有校验输出 `OK`。

## 9.同步官方仓库及本地合并

明白了，你的习惯是：

1. 在 GitHub 网页上**手动**点 fork 仓库的 “Sync fork” 按钮，让 fork 的 master 和官方一致
2. 本地 `git fetch origin` + `git pull` 把 fork 的 master 拉到本地
3. 把 master 合并到 dev

这完全没问题，甚至比命令行 `upstream` 方案更简单，因为**不需要在本地配置 upstream remote**。下面按你的习惯重写流程。

---

## 9.1 配置检查

确认只剩 origin：

```bash
git remote -v
```

应只有 2 行，都是 `origin`。
例如：

```bash
origin  git@github.com:kuyacai/deepseek-harness.git (fetch)
origin  git@github.com:kuyacai/deepseek-harness.git (push)
```
---

## 9.2 日常同步流程

### 步骤 1：GitHub 网页上点 “Sync fork”

1. 打开 `https://github.com/<你的账号>/deepseek-harness`
2. 在 master 分支下，点 **Sync fork** → **Update branch**
3. 确认 fork 的 master 和官方一致

如果网页提示有冲突（少见，通常是因为你在 fork 的 master 上改过东西），按提示处理，或者先把 fork 的 master 恢复成官方版本。

### 步骤 2：本地切到 master

```bash
git checkout master
```

### 步骤 3：确认工作区干净

```bash
git status
```

必须是：

```
nothing to commit, working tree clean
```

不干净就先处理（`git restore`、`git stash` 等）。

### 步骤 4：把 fork 的 master 拉到本地

```bash
git fetch origin
git merge --ff-only origin/master
```

- 因为你从未在本地 master 上提交，通常直接快进
- 如果报 `Not possible to fast-forward`，说明本地 master 有本地提交，需要先处理

或者用一条命令：

```bash
git pull --ff-only origin master
```

`--ff-only` 保证只做快进合并，不会产生意外 merge commit。

### 步骤 5：切到 dev

```bash
git checkout dev
```

### 步骤 6：确认工作区干净

```bash
git status
```

不干净先 commit 或 stash。

### 步骤 7：合并 master 到 dev

```bash
git merge master
```

- 无冲突：自动完成
- 有冲突：见第四节

### 步骤 8：推送 dev 到 fork（可选）

```bash
git push origin dev
```

---

## 9.3 完整命令清单

```bash
cd ~/Documents/workspace/deepseek-harness

# === 前置：去 GitHub 网页点 "Sync fork" ===

# === 本地同步 master ===
git checkout master
git status                      # 必须 clean
git fetch origin
git merge --ff-only origin/master

# === 合并到 dev ===
git checkout dev
git status                      # 必须 clean
git merge master
git push origin dev             # 可选
```

---

## 9.4 如果有冲突

```bash
git status
```

列出冲突文件，逐个编辑解决，然后：

```bash
git add <冲突文件>
git commit
```

放弃合并：

```bash
git merge --abort
```

---

## 9.5 合并后编译

源码变了，必须重新编译，并且杀掉旧进程：

```bash
# 确认在 dev
git branch

# 清理产物
pnpm run clean || true
find packages -type d \( -name lib -o -name dist \) -prune -exec rm -rf {} +
rm -rf dist .dsh-build build.log

# 安装 + 编译
pnpm install
pnpm run build 2>&1 | tee build.log

# 校验
test -s packages/client/ui-renderer/lib/client.js \
  && echo "ui-renderer OK" || echo "ui-renderer 缺失"
test -s packages/client/modules/lib/client.js \
  && echo "modules OK" || echo "modules 缺失"
test -s packages/context/agent-instructions/node_modules/@deepseek-ai/dsh-llm/lib/typert.host.js \
  && echo "typert.host OK" || echo "typert.host 缺失"

# 杀掉旧进程
pkill -f "dsh web" || true
lsof -iTCP -sTCP:LISTEN -P | grep -i node || echo "无监听进程"

# 启动
pnpm dsh web
```

---

## 9.6 这套流程的关键点

| 事项 | 说明 |
|------|------|
| 不需要本地 `upstream` remote | 官方同步在网页上完成 |
| master 只用来接收同步 | 不在 master 上做任何本地改动 |
| 用 `--ff-only` | 避免误产生 merge commit |
| 合并前 `git status` 必须干净 | 否则 merge 可能带上未提交改动 |
| 合并后必须重新 build | 产物不属于 git，不会自动更新 |
| 启动前杀掉旧进程 | 否则浏览器访问的还是旧服务 |

---

## 9.7 给 agent 的版本

> 官方同步已在 GitHub 网页完成，fork 的 master 已与官方一致。本地流程：
>
> 1. `git checkout master && git status`（必须 clean）
> 2. `git fetch origin && git merge --ff-only origin/master`
> 3. `git checkout dev && git status`（必须 clean）
> 4. `git merge master`，冲突则解决后 `git add && git commit`
> 5. `pnpm run clean`，删掉 `packages/*/lib`、`dist`、`.dsh-build`
> 6. `pnpm install && pnpm run build 2>&1 | tee build.log`
> 7. 确认 `build.log` 出现 `build: recorded <N> client artifact(s)`
> 8. `test -s` 校验 `packages/client/*/lib/client.js` 和 `typert.host.js`
> 9. `pkill -f "dsh web"` 杀旧进程，再 `pnpm dsh web`

---