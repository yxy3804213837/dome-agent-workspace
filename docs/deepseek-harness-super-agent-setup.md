# DeepSeek Harness Super Agent 安装与配置指南

> 本文说明如何在一台机器上安装、启动并配置 **dsh-superagent**（DeepSeek Harness 的 Super Agent 扩展）。dsh-superagent 是研究原型，不是 DeepSeek 官方产品；代码和文档都可能继续变化，正式使用前请结合自己的环境评估。
>
> 本文侧重“装起来、配好、验证通过”，不展开授权模型和内部架构。概念和设计见 dsh-superagent 源码仓库的 `docs/000`–`docs/008`。
>
> 命令默认在 dsh-superagent 源码仓库根目录执行；`<...>` 表示需要替换的占位符。除特别说明外，示例使用 POSIX shell（Linux/macOS）；Windows 的差异在相应小节单独说明。

## 0. 最短路径

如果你只想先跑起来，按下面的顺序做，每一步的细节在后面对应章节：

1. 安装 DeepSeek Harness 并配置好至少一个模型提供方（[§2](#2-前置条件)、[§4.1](#41-安装-deepseek-harness)）。
2. 准备一个**专用**的智能体 GitHub 账号和只登录该账号的 `gh` 包装器（[§3.1](#31-github-智能体账号)）。
3. 克隆并构建 dsh-superagent 源码（[§4.2](#42-获取并构建-dsh-superagent)）。
4. 从 Web 模板创建独立的 `superagent` profile，并安装 bundle（[§4.3](#43-创建专用-profile)、[§4.4](#44-安装-bundle)）。
5. 写入 `cordis.patch.yml`，至少设置 `stateDir`（[§5](#5-机器级配置)）。
6. 启动服务；首次 `status` 可能是 `3`（等待补全配置），打开日志里的面板地址（[§6](#6-启动停止与状态检查)）。
7. 在面板的“设置 → 来源/评估/执行/提示词”中完成仓库、名单、准备命令等配置（[§7](#7-在面板中完成业务配置)）。
8. 再确认 `status` 为 `0`；用名单内的账号在 Issue 里 `@<智能体账号> <需求>`，确认一分钟内出现工单（[§9](#9-验证安装)）。

## 1. 部署后有哪些东西

dsh-superagent 不是一个独立应用，而是加载进 DeepSeek Harness 的一个 bundle。一次完整部署包含：

| 组件 | 作用 | 是否必需 |
| --- | --- | --- |
| `superagent` DSH profile | 独立进程，加载 dsh-superagent bundle，在 `127.0.0.1` 上提供面板 | 必需 |
| 智能体 GitHub 账号 | 以这个账号评论、推送分支、开 PR；通过专用 `gh` 包装器使用 | 必需 |
| 目标仓库的本地克隆 | 每个工单用 `git worktree` 从这里检出独立目录 | 建议，取决于准备命令 |
| 飞书机器人 | 从飞书群或私聊接收请求；通过专用 `lark-cli` 包装器使用 | 可选 |
| 状态目录 `stateDir` | 数据库、锁、worker 命令 shim、智能体的 git 配置；默认也放工单目录 | 必需 |
| 模型提供方 | DeepSeek Harness 已配置的模型；评估和执行可分别指定模型 | 必需 |

一次请求的路径大致是：

```text
GitHub Issue/PR 评论或飞书消息
        │
        ▼
Source 轮询 ──► Trigger ──► Eval（谁、是否接、范围）
                                  │ accept
                                  ▼
                               Ticket ──► Executor Session + 独立目录
                                              │
                                              ▼
                                   PR / 评论 / 飞书回复，回到原线程
```

这套系统刻意把“主 Session”和“执行器”分开：主 Session 只做评估和路由，执行器读原始上下文、写代码并验证。安装和配置时，主要工作就是给这两类 Session 准备好账号、目录和规则。

## 2. 前置条件

| 依赖 | 要求 | 检查命令 |
| --- | --- | --- |
| Node.js | 22.19+ 或 24+ | `node -v` |
| pnpm | 能安装并构建本仓库；仓库声明了 `packageManager` | `pnpm -v` |
| git | 2.32+；准备命令用 `git worktree` | `git --version` |
| Python | 3.8+ | `python3 --version`（Windows 可能是 `python --version`） |
| tmux | 仅 POSIX 启停脚本需要 | `tmux -V` |
| GitHub CLI | 2.48+；必须是 `gh` 或包装器 | `gh --version` |
| DeepSeek Harness | 版本满足 `package.json` 的 `engines.dsh`（`^0.2.1-alpha.2`） | `dsh --version` |
| 目标仓库 | github.com 上的仓库；暂不支持 GitHub Enterprise Server；智能体账号有写权限 | 见 [§3.1](#31-github-智能体账号) |
| 模型提供方 | 在 DeepSeek Harness 中已配置并能正常使用 | 在 DSH 里发一条测试消息 |
| 飞书 CLI（可选） | `lark-cli`（测试过 1.0.90）和带机器人能力的飞书/Lark 自建应用 | `lark-cli --version` |

补充说明：

- 使用 DeepSeek Harness 桌面版时没有独立的 `dsh` 命令；用桌面版内置 CLI，其 DSH 版本随桌面版，见 [§4.5](#45-windows-桌面版没有独立-dsh-命令时)。
- **Windows 不需要 tmux**。Windows 使用 `scripts/dsh-superagent.ps1`，见 [§6.1](#61-启动与停止)。
- **准备命令的 shell 因平台而异**：POSIX 用 `sh -c`，Windows 用 PowerShell。写准备命令时按目标平台选择语法。
- **隔离不是安全沙箱**。执行器以运行 dsh-superagent 的 OS 用户身份运行，拥有完整文件访问权限；包装器只负责选择凭据。需要真正隔离时，请使用只含智能体凭据的专用 OS 账号（见 [§11](#11-安全边界)）。
- 如果目标仓库使用 Git LFS，记得安装 `git-lfs`。

## 3. 准备账号与身份

### 3.1 GitHub 智能体账号

智能体的所有评论、分支和 PR 都以这个账号出现，所以**不要使用运营者自己的账号**。建议为它单独注册一个 GitHub 用户账号，并授予目标仓库写权限。

1. 创建专用 GitHub 用户账号，邀请它加入目标仓库（或组织），并接受邀请；确认它对仓库有写权限。
2. 给这个账号一个只使用它自己凭据的 `gh` 包装器。

POSIX：

```sh
mkdir -p ~/.local/bin
cat > ~/.local/bin/gh-agent <<'EOF'
#!/bin/sh
GH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"
EOF
chmod +x ~/.local/bin/gh-agent
gh-agent auth login
gh-agent api user --jq .login
```

`gh-agent auth login` 会要求你在终端里完成登录；最后一条命令应打印智能体账号的登录名。把 `~/.local/bin` 放进 `PATH`，或者稍后在来源设置里填包装器的绝对路径。

Windows（CMD）：

```bat
mkdir "%USERPROFILE%\.local\bin" 2>nul
> "%USERPROFILE%\.local\bin\gh-agent.cmd" (
  echo @echo off
  echo set "GH_CONFIG_DIR=%%USERPROFILE%%\.config\gh-agent"
  echo gh %%*
)
```

PowerShell 下可以用下面这段创建同一个 `.cmd` 文件：

```powershell
$dir = "$env:USERPROFILE\.local\bin"
New-Item -ItemType Directory -Force -Path $dir | Out-Null
@'
@echo off
set "GH_CONFIG_DIR=%USERPROFILE%\.config\gh-agent"
gh %*
'@ | Set-Content -Encoding ASCII "$dir\gh-agent.cmd"
```

把 `%USERPROFILE%\.local\bin` 加入 `PATH`，然后运行：

```bat
gh-agent auth login
gh-agent api user --jq .login
```

注意事项：

- **不能用 GitHub App 代替**。dsh-superagent 需要读取该账号的通知（notifications），GitHub App 不能满足这个前提。
- 包装器只固定 `GH_CONFIG_DIR`，不隔离进程；不要把它当作安全边界。
- 登录凭据只在本机终端输入，不要写进文档、聊天或仓库。

### 3.2 飞书机器人（可选）

只有需要从飞书接收请求时才配置。

1. 在飞书开放平台创建**自建应用**，启用机器人能力。
2. 授予接收消息事件、读取和发送群消息、读取群成员等权限，然后发布应用版本。
3. 给机器人一份独立的 `lark-cli` 配置，并限制为 bot 身份。

下面的 `cli_xxx` 换成飞书应用的 app id。

POSIX：

```sh
LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" \
  lark-cli config init --app-id cli_xxx --app-secret-stdin

LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" \
  lark-cli config strict-mode bot --global

cat > ~/.local/bin/lark-bot <<'EOF'
#!/bin/sh
export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
exec lark-cli "$@"
EOF
chmod +x ~/.local/bin/lark-bot
```

Windows（CMD）：

```bat
set "LARKSUITE_CLI_CONFIG_DIR=%USERPROFILE%\.lark-cli-dsh-superagent"
lark-cli config init --app-id cli_xxx --app-secret-stdin
lark-cli config strict-mode bot --global

> "%USERPROFILE%\.local\bin\lark-bot.cmd" (
  echo @echo off
  echo set "LARKSUITE_CLI_CONFIG_DIR=%%USERPROFILE%%\.lark-cli-dsh-superagent"
  echo lark-cli %%*
)
```

第一条 `config init` 会从标准输入读取应用密钥，按提示粘贴即可。PowerShell 下把 `set "NAME=value"` 换成 `$env:NAME = 'value'`，并用 `Set-Content` 创建同样的 `lark-bot.cmd`。

注意事项：

- 使用**单独的配置目录**。`lark-cli` 对一个 app id 只允许一份 profile，你的个人配置里可能已有该应用的用户登录。
- 宿主会拒绝能作为用户身份行动的 `lark-cli`；`strict-mode bot` 是必需的。
- 两种配置共享 keychain 里的应用密钥。只想移除机器人配置时，删除它的配置目录即可；**不要**运行 `lark-cli config remove`，那会影响共享密钥。
- 机器人必须被拉进要监听的群，否则来源会报“机器人不是群成员”。

### 3.3 准备目标仓库的本地克隆

每个工单默认会在自己的目录里工作。最常见的做法是保留一份目标仓库的克隆，用 `git worktree` 为每个工单创建独立工作树。

```sh
git clone https://github.com/OWNER/REPO.git ~/src/REPO
```

把 `OWNER` 换成仓库所属账号，`REPO` 换成仓库名；本地目录名也用 `REPO`，方便和下面的准备命令对应。

这份克隆和 dsh-superagent 源码是两回事：

- dsh-superagent 源码：安装 bundle 用。
- 目标仓库克隆：准备命令用它创建工单工作树。

准备命令会以智能体账号的凭据访问 github.com；个人 git 配置（例如 HTTP 代理）不会自动生效。如果目标仓库使用 LFS，先在这份克隆里确认 LFS 可用。

## 4. 安装

### 4.1 安装 DeepSeek Harness

有两种方式，任选其一：

**方式 A：安装独立 CLI（Linux/macOS）。**

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
dsh --version
```

上面是官方部署文档给出的固定版本；也可以安装任何满足 `package.json` 中 `engines.dsh`（`^0.2.1-alpha.2`）的版本。首次安装后，先在 DeepSeek Harness 中配置好模型提供方并确认能正常对话。

**方式 B：使用 DeepSeek Harness 桌面版。**

桌面版本自带 DSH，不需要单独执行 `npm install -g`。它的 CLI 位于应用安装目录内，Windows 上的用法见 [§4.5](#45-windows-桌面版没有独立-dsh-命令时)。macOS 若使用桌面版，也使用其内置 CLI，具体路径以实际安装为准。

### 4.2 获取并构建 dsh-superagent

```sh
git clone REPO_URL dsh-superagent
cd dsh-superagent
pnpm install
pnpm build
```

把 `REPO_URL` 换成 dsh-superagent 源码仓库的地址。构建产物在 `lib/`。之后 profile 会以 `link:` 方式引用这个目录，因此：

- 源码目录不要随意移动或删除，否则 profile 会找不到 bundle。
- 修改源码后要重新 `pnpm build`，再重启服务。

### 4.3 创建专用 profile

为 Super Agent 单独创建一个 profile，不要和日常工作的 profile 混用。这样插件对 `PATH`、git 配置的修改，以及独立的重启和升级，都不会影响你正在用的 Session。

```sh
dsh superagent --from-default-profile web --dump-config > /dev/null
```

这条命令用 DeepSeek Harness 的 Web 模板创建 `~/.dsh/profiles/superagent`。`--dump-config` 只打印配置后退出，不启动服务。设置 `DSH_HOME` 可以把默认的 `~/.dsh` 换到别处。

> **警告：这条命令只用于全新安装。** 如果 `superagent` profile 已经存在，它会用 Web 模板替换整个 profile，并删除其中的 `cordis.patch.yml`（包括 `stateDir` 等机器配置）。升级时不要运行它，直接改现有 profile 的 `cordis.patch.yml` 即可。

### 4.4 安装 bundle

在 dsh-superagent 源码根目录执行：

```sh
dsh plugin --profile superagent add "link:$PWD"
```

这会把当前目录作为 `dsh-superagent` 依赖链接进 `superagent` profile。可以用下面的方式确认：

```sh
cat "${DSH_HOME:-$HOME/.dsh}/profiles/superagent/package.json"
```

依赖里应出现 `"dsh-superagent": "link:<源码目录>"`。

### 4.5 Windows 桌面版：没有独立 `dsh` 命令时

如果使用桌面版且没有安装独立 CLI，可以用桌面版自带的 CLI。下面假设 DeepSeek Harness 安装在 `C:\Program Files\DeepSeek Harness`；请换成你的实际路径。

```powershell
$exe = 'C:\Program Files\DeepSeek Harness\DeepSeek Harness.exe'
$cli = Join-Path (Split-Path $exe) 'resources\app.asar\dsh\node_modules\@deepseek-ai\dsh-desktop-host\lib\cli.js'
$env:ELECTRON_RUN_AS_NODE = '1'

# 只在新安装时运行一次
& $exe --expose-internals $cli superagent --from-default-profile web --dump-config | Out-Null

# 在源码目录执行；link: 里用正斜杠
Set-Location 'D:\src\dsh-superagent'
$src = (Get-Location).Path -replace '\\', '/'
& $exe --expose-internals $cli plugin --profile superagent add "link:$src"
```

之后用 Windows 服务脚本启停：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File scripts\dsh-superagent.ps1 start
powershell -NoProfile -ExecutionPolicy Bypass -File scripts\dsh-superagent.ps1 status
```

脚本默认会从正在运行的 DeepSeek Harness 进程推断可执行文件；推断不到时，设置 `DSH_SUPERAGENT_EXE` 环境变量，或给脚本传 `-DshExe`。如果服务端口不是 3082，还要设置 `DSH_SUPERAGENT_URL`，或传 `-Url`。

## 5. 机器级配置

### 5.1 `cordis.patch.yml`

首次启动前，把源码仓库的 `examples/profile.cordis.patch.yml` 复制到 `~/.dsh/profiles/superagent/cordis.patch.yml`，替换其中的占位符。如果该文件已经存在（例如升级时），保留已有条目，只补上或修改 `stateDir` 和端口，不要整文件覆盖：

```yaml
# ~/.dsh/profiles/superagent/cordis.patch.yml
- id: webserver
  config:
    host: 127.0.0.1
    port: 3082

- id: dsh-superagent
  config:
    stateDir: /home/me/.dsh/state/superagent

# 需要飞书时取消注释；机器人 CLI 在面板的飞书来源卡片里设置
# - id: dsh-superagent-feishu
#   disabled: false
```

Windows 的 `stateDir` 用单引号或正斜杠，避免 YAML 把反斜杠当转义：

```yaml
- id: dsh-superagent
  config:
    stateDir: 'C:\Users\me\.dsh\state\superagent'
```

端口说明：

- 面板和管理 API 默认监听 `127.0.0.1:3082`。
- 如果 3082 已被其他 DSH profile 占用，改 `port`，并让启停脚本使用新地址。例如改成 3083 后：
  ```sh
  export DSH_SUPERAGENT_URL=http://127.0.0.1:3083
  ```
  Windows 上设置 `$env:DSH_SUPERAGENT_URL = 'http://127.0.0.1:3083'`。
- 只监听 `127.0.0.1`。不要把面板端口转发到公网，管理 API 具有面板级权限。

### 5.2 可选机器上限

同一文件里还可以设置几个不常改的上限；不写就使用默认值。首次启动后，也可以在面板的“设置 → 部署”里修改（`stateDir` 除外），修改立即生效，不需要重启。

| 键 | 默认值 | 含义 |
| --- | --- | --- |
| `stateDir` | 无默认，必填 | 绝对路径；数据库、锁、worker 命令 shim、git 配置和默认工单目录都放在这里 |
| `promptMaxChars` | `100000` | 保存的提示词配置最大字符数（1000–1000000） |
| `admissionContextMaxPages` | `10` | 一次评估最多读取多少页线程上下文（1–100） |
| `admissionContextTimeoutMs` | `15000` | 一次评估读取上下文的超时，毫秒（1–90000） |

无效值或未知键会导致启动失败，日志会指出是哪一个键。

### 5.3 `stateDir` 里有什么

| 路径 | 内容 |
| --- | --- |
| `<stateDir>/orchestrator.sqlite` | 工单、触发、决定、授权等状态；**升级前必须备份** |
| `<stateDir>/worker-bin` | 执行器看到的命令 shim：`gh` 指向智能体 CLI，`lark-cli` 指向机器人 CLI，被禁止的命令直接失败 |
| `<stateDir>/worker-gitconfig` | 执行器使用的 git 配置；只使用智能体凭据，不读运营者的全局/系统 git 配置 |
| `<stateDir>/work` | 默认的工单目录根；每个工单一个子目录 |
| `<stateDir>/...` | 锁、临时文件等 |

两个容易踩的坑：

- `worker-bin` 和 `worker-gitconfig` 由 dsh-superagent 生成。如果里面有不是它写的文件，部署检查会失败。不要把个人脚本塞进去。
- 改 `stateDir` 等于换一套状态，不会自动迁移旧数据；确定路径后再启动。

## 6. 启动、停止与状态检查

### 6.1 启动与停止

POSIX：

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
scripts/dsh-superagent stop
scripts/dsh-superagent restart
```

Windows：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File scripts\dsh-superagent.ps1 start
powershell -NoProfile -ExecutionPolicy Bypass -File scripts\dsh-superagent.ps1 status
powershell -NoProfile -ExecutionPolicy Bypass -File scripts\dsh-superagent.ps1 stop
powershell -NoProfile -ExecutionPolicy Bypass -File scripts\dsh-superagent.ps1 restart
```

行为说明：

- `start` 在 tmux 会话 `dsh-superagent` 中运行 `dsh --profile superagent`（Windows 上是独立的隐藏进程），最多等待一分钟，直到运行时就绪或报告缺少的部署项。首次启动会打印需要补的配置。
- `start` 会拒绝重复实例，包括在脚本之外启动的实例。
- POSIX 下，`stop` 和 `restart` 会请求优雅关闭，并最多等待五分钟让执行器收尾；不会强制杀掉其他 tmux 会话。Windows 脚本用 `taskkill` 结束进程树，SQLite 会在下次启动时恢复。
- 只有显式 `stop` 会写入“手动停止”标记；`start` 会清除它。自动恢复应使用 `--respect-stop`（PowerShell 为 `-RespectStop`），以免把有意停止的服务拉起来。
- 换 profile 或端口时，使用 `--profile`/`--url`，或设置 `DSH_SUPERAGENT_PROFILE`/`DSH_SUPERAGENT_URL`。

### 6.2 退出码

`status`、`start`、`restart` 的退出码含义相同：

| 退出码 | 含义 | 处理 |
| --- | --- | --- |
| `0` | 服务正常运行，部署检查通过 | 无需处理 |
| `1` | 没运行、跑了两份、不响应，或运行时报告错误 | 先看输出和最新日志；通常重启可恢复 |
| `2` | 服务在运行，但运行时之外的部分（如某个来源、`sync:<source id>`）报错 | 重启不解决；看 `errors` 里的具体部分 |
| `3` | 服务在运行，但等待补全部署配置 | 重启不解决；看 `setup` 里的缺失项 |

`status` 是 JSON 输出，失败部分在 `errors`，部署缺失在 `setup`。`start`/`restart` 会把问题按行打印；如果服务始终没起来，它们退出 1。

### 6.3 日志与面板

每次启动都会在仓库的 `.artifacts/` 下写日志：

- POSIX：`dsh-superagent-<profile>-<日期>-<时间>-<pid>.log`，保留最新 20 份。
- Windows：同名 `.log`，另有 `.log.err`。
- 控制文件：`.artifacts/control/profiles/<profile>/`，包含停止标记、锁和 pid。

日志里会打印带访问 token 的面板地址（形如 `dsh web: http://127.0.0.1:3082/...`）。**token 每次启动都会变**，以最新日志为准。打开地址后，在 DeepSeek Harness 侧栏选择 **Superagent**。

如果面板提示 dsh-superagent 没有响应，通常是正在重启，或配置有误；后者应查看 DSH 自己的日志。

### 6.4 每小时健康检查（可选）

建议在**日常使用的 DSH profile**（不是 `superagent`）里创建一个每小时任务，检查服务是否还活着。它的逻辑应是：

- `status` 退出 0：什么都不做，不通知。
- 退出 2 或 3：不要重启；分别报告 `errors` 或 `setup` 的内容，除非同样的内容已经报过。
- 退出 1：运行 `scripts/dsh-superagent --respect-stop restart`（PowerShell 用 `scripts\dsh-superagent.ps1 -RespectStop restart`）。恢复成功就保持安静；仍失败则查看最新日志并报告。
- 如果 `status` 显示 `manually_stopped: true`，不要恢复。

源码仓库的 `docs/006-deployment.md` 里有可直接使用的完整提示词。

## 7. 在面板中完成业务配置

首次启动后，如果部署检查未通过，**概览（Overview）** 会列出缺失项并链接到对应设置。设置保存后会重新运行检查；如果你在应用之外修好了问题（例如重新登录了 CLI），在概览上点“重新检查”。

设置分为五组：**来源（Sources）**、**评估（Eval）**、**执行（Execution）**、**提示词（Prompts）**、**部署（Deployment）**。下面按配置顺序说明。

### 7.1 来源（Sources）

#### GitHub 卡片

| 字段 | 说明 |
| --- | --- |
| 监听的仓库 | `owner/name`，只支持 github.com；保存时会检查智能体账号能否推送 |
| 智能体的 GitHub CLI | `gh-agent` 的命令名或绝对路径；保存时会检查它登录的账号 |
| 轮询间隔 / 秒 | 默认 60，范围 60–3600；GitHub 要求更长时以 GitHub 为准 |
| 连续失败几次后报告异常 | 默认 3，范围 1–20；单次网络中断会在下一次轮询重试 |
| 接收仓库里的新 Issue | 默认关闭；打开后没有 @ 智能体的新 Issue 也进入准入表 |
| 接收仓库里的新 PR | 默认关闭 |
| 接收仓库里的评论 | 默认关闭；包括 Issue/PR 评论和代码评论 |
| 接收默认分支的 CI | 默认关闭；接收默认分支 CI 的失败和恢复 |

请求通常通过智能体账号的 GitHub 通知到达。**新 Issue 或新评论**中的 @ 会在下一次轮询被取到；**编辑已有 Issue 正文**加入的 @ 不会被当作新通知，请单独发一条评论。

#### 飞书卡片（可选）

| 字段 | 说明 |
| --- | --- |
| 机器人的 CLI | `lark-bot` 的命令名或绝对路径 |
| 群列表 | 每项包含 `chatId`、名称和接收方式（“仅 @”或“全部消息”） |
| 其他群 | “不接收”（默认）或“仅 @” |
| 接收私聊 | 默认开启 |

飞书来源的部署检查失败只会暂停飞书来源，不影响 GitHub。机器人必须已在目标群里；如果设置时机器人还没入群，入群后重试即可。

### 7.2 策略（Policy）

策略决定“谁能发起工作”和“主动工作接不接”。它由评估页的规则、各来源的名单和接单模式共同组成，每次修改都会生成新的策略版本，并通知进行中的工单。

#### 名单

- 每个来源各有一份名单：GitHub 填登录名，飞书填用户 `open_id`。
- **名单为空时，没有人能发起工作**，任何请求都不会成为工单。这是新手最常见的“没反应”原因。
- 飞书可以从某个群同步成员名单：启动时同步一次，也可以点“立即同步”。
- GitHub 上还可以配置“机器人名单”（例如 `dependabot[bot]`），名单内机器人的消息最多成为“推断候选”，永远不能是明确请求。

#### 接单模式

| 模式 | 行为 |
| --- | --- |
| `off`（关闭） | 只处理明确的请求；不评估其他触发 |
| `observe`（仅观察，默认） | 评估其他触发并记录“本来会接什么”，但不接单 |
| `auto`（自动接单） | 其他触发符合已启用规则时接单 |

**首次部署保持 `observe`**。先看它记录的决定是否符合预期，再考虑切到 `auto`。明确请求在任何模式下都会处理。

#### 准入表

准入表按“接收范围”分行（GitHub 的 Issue/PR/评论/CI，飞书的每个群/其他群/私聊），每行最多三格：

| 格 | 可选值 | 默认 |
| --- | --- | --- |
| 名单内、未 @ 智能体 | 推断候选 / 忽略 | 推断候选 |
| 名单外、@ 了智能体 | 明确请求 / 推断候选 / 忽略 | 忽略 |
| 名单外、未 @ 智能体 | 推断候选 / 忽略 | 忽略 |

只有“接收全部消息”的范围才有“未 @”两格。把“名单外、@ 了智能体”设为“明确请求”，可以让某个支持群里的任何人直接发起工作；这会放宽权限，请谨慎使用。

即使准入表允许，**推断候选仍受接单模式约束**：`observe` 只记录，`auto` 才执行。

#### 规则

评估页有两条示例规则（文案修正、局部视觉修正），可以编辑或删除，也可以新增。每条规则由名称和“接单条件”组成，写明可以接什么、应忽略什么；如果某种工作有指定的 PR 评审人，也可以写在规则里。

主动接单需要同时满足：接单模式为 `auto`、至少一条启用的规则命中、评估给出明确方向。规则命中的证据和范围会写进工单的授权记录。

### 7.3 执行（Execution）

| 设置 | 默认值 | 说明 |
| --- | --- | --- |
| 执行器模型 | 空（使用 profile 默认模型） | 每个工单的执行器 Session 使用的模型 |
| 并行工单数 | `3` | 超过上限的工单排队 |
| 执行目录的根目录 | 空（`<stateDir>/work`） | 每个工单在这里有一个以短名命名的目录；只按目录分开，不是安全边界 |
| 准备命令 | 空（空目录） | 新工单目录建好前，在根目录下运行一次；可用执行环境变量 |
| 禁止调用的命令 | 空 | 执行器按命令名调用这些命令会直接失败；`gh`、`git`、`lark-cli` 不能禁止 |

#### 准备命令

准备命令的作用是“把一个空目录变成这个工单的工作区”。最常用的是从目标仓库克隆创建 `git worktree`。

POSIX（`sh`）：

```sh
git -C /home/me/src/widgets fetch --no-tags \
  "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" \
  "${DSH_SUPERAGENT_GITHUB_PR_HEAD:-HEAD}:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET" &&
git -C /home/me/src/widgets worktree add -b "agent/$DSH_SUPERAGENT_NAME" \
  "$DSH_SUPERAGENT_DIR" "refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
```

Windows（PowerShell）：

```powershell
$prHead = if ($env:DSH_SUPERAGENT_GITHUB_PR_HEAD) { $env:DSH_SUPERAGENT_GITHUB_PR_HEAD } else { 'HEAD' }
git -C 'C:\Users\me\src\widgets' fetch --no-tags "https://github.com/$($env:DSH_SUPERAGENT_GITHUB_REPOSITORY).git" "${prHead}:refs/dsh-superagent/$($env:DSH_SUPERAGENT_TICKET)"
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
git -C 'C:\Users\me\src\widgets' worktree add -b "agent/$($env:DSH_SUPERAGENT_NAME)" "$env:DSH_SUPERAGENT_DIR" "refs/dsh-superagent/$($env:DSH_SUPERAGENT_TICKET)"
```

说明：

- 准备命令在根目录下运行，环境变量由执行环境提供（见 [§13.3](#133-执行环境变量)）。
- 失败时工单会标记为失败，并附上命令输出；修好命令后重试工单即可。
- 已有目录会被视为“已准备”，所以自动重试不会重跑准备命令；失败后要先处理目录。
- `blockedCommands` 只拦“按名字调用”，用绝对路径仍能运行，因此它**不是**安全边界。

### 7.4 评估（Eval）

| 设置 | 默认值 | 说明 |
| --- | --- | --- |
| 接单模式 | `observe` | 见 [§7.2](#72-策略policy) |
| 单次评估时限 / 秒 | `120` | 主 Session 读取上下文并研判；超时记为未完成，后续触发可重新评估 |
| 研判模型 | 空（使用 profile 默认模型） | 负责评估和路由的主 Session 模型；保存后用于新建主 Session，已有 Session 保持原选择 |

### 7.5 提示词（Prompts）

提示词决定智能体“怎么说话、怎么交付”。至少确认下面三项团队约定：

1. **GitHub 首行声明**：默认每条 GitHub 消息以 `This is by an automated agent, not a human.` 开头。团队可以改文案，但不要删除“自动化”这一事实。
2. **自动评审机器人**：有些自动评审工具使用普通 GitHub 账号，宿主会把它当人。把这些账号写进“Communication”部分，智能体才会按机器人规则处理它们（例如每个 PR 最多三轮评审）。
3. **提交作者与自动合并条件**：默认以智能体账号的 noreply 邮箱提交。若要用其他身份，写在“Delivery requirements”；默认不自动合并，由人合并。若开启自动合并，需要另一个可合并账号的 CLI，并在提示词里写明条件。

“提示词”页还支持覆盖内置提示词片段、添加团队技能，以及查看完整提示词和历史版本。覆盖过的片段在默认内容升级后会标记 **“默认内容已更新”**，可以选择保留团队文本、采用新默认或继续编辑。

### 7.6 保存、生效与权限

- 每组设置各有修订号；保存时带上当前修订号，冲突会返回 409，不会覆盖别人的修改。
- 修改仓库或 CLI 会先跑部署检查；检查失败返回 400，并给出 `setup` 详情。
- 账号类修改成功，或部署尚未完成时的保存，会返回 `restarting: true` 并重启 dsh-superagent。等 `status` 不再退出 1，再从 `GET settings` 读新的修订号。
- 策略版本变化会通知进行中的工单；削弱名单或准入表可能在工单恢复时使其授权失效。
- 暂停处理（执行页）会停止新的评估、派工和回复，但来源仍继续记录触发。

## 8. 管理 API（可选）

面板本身调用本地 HTTP API，也可以用脚本或代理调用。地址是 `http://127.0.0.1:<port>/orchestrator/api/`，请求必须带 `x-orchestrator-request: 1`，Host 必须是 `127.0.0.1:<port>` 或 `localhost:<port>`；缺少请求头或来自其他站点的 `Origin` 会收到 403。

常用路由：

| 请求 | 作用 |
| --- | --- |
| `GET state` | 部署检查、健康状态、执行器负载、暂停状态 |
| `GET settings` | 策略、评估、执行、来源和修订号 |
| `PUT settings/<block>` | 保存一个块（`source:github`、`source:feishu`、`policy`、`eval`、`executor`） |
| `POST executor/pause` | `{"paused": true}` 暂停或恢复处理 |
| `POST setup/check` | 重新运行部署检查（会重启服务） |
| `GET/PUT prompts` | 读取或保存提示词配置 |
| `POST policy-sync/<source id>` | 立即同步该来源的名单 |
| `POST tickets/<id>/retry`、`POST tickets/<id>/stop` | 重试或停止工单 |

示例：把 GitHub 来源指向仓库和智能体 CLI。

```sh
api=http://127.0.0.1:3082/orchestrator/api
rev=$(curl -s -H 'x-orchestrator-request: 1' "$api/settings" \
  | jq '.sources[] | select(.id == "github") | .revision')

curl -s -X PUT \
  -H 'x-orchestrator-request: 1' \
  -H 'content-type: application/json' \
  "$api/settings/source:github" \
  -d "{\"value\":{\"enabled\":true,\"config\":{\"repository\":\"acme/widgets\",\"cli\":\"$HOME/.local/bin/gh-agent\"}},\"revision\":$rev}"
```

错误格式：

- 400：值无效（`issues` 指出字段）、账号检查失败（`setup` 详情）、固定拒绝（`code` 和 `values`），或请求体不是 JSON/过大。
- 404：对象不存在。
- 409：修订号过期。
- 500：其他错误。

API 具有面板级权限。不要把端口暴露到公网；也不要把管理 API 当作跨机器接口。

## 9. 验证安装

按顺序检查：

1. **进程与部署检查**：运行 `scripts/dsh-superagent status`，退出码应为 `0`。首次启动若为 `3`，按 `setup` 的提示在面板修正。
2. **面板**：打开最新日志里的 URL，侧栏选择 **Superagent**。概览不应再显示部署问题；来源页应显示 GitHub（和飞书，如果启用）为运行中。
3. **来源**：GitHub 卡片应显示仓库和智能体账号；名单里至少有一个人。
4. **测试请求**：用名单内的 GitHub 账号，在目标仓库新建一个 Issue 或发一条评论：
   ```text
   @<智能体账号> 请把 README 里这个拼写错误改掉
   ```
   默认轮询间隔约一分钟；概览应出现一张工单。智能体应在原 Issue/评论下确认，然后创建自己的 Session 和目录，完成后把 PR 链接回到原线程。
5. **飞书（如启用）**：在配置的群里 @ 机器人，发一个请求，确认原话题里收到回复。
6. **没有反应时**，依次检查：来源是否打开、请求者是否在名单内、处理是否被暂停、@ 是否出现在新评论而不是编辑后的正文、GitHub CLI 是否仍登录智能体账号。

## 10. 升级、备份与回滚

### 10.1 升级 dsh-superagent

1. 等执行器都不在处理中的回合（避免中途打断）。
2. 备份 `<stateDir>/orchestrator.sqlite`。
3. 在源码目录更新并重新构建：
   ```sh
   git pull
   pnpm install
   pnpm build
   ```
4. 重启并检查：
   ```sh
   scripts/dsh-superagent restart
   scripts/dsh-superagent status
   ```

升级会结束正在进行的派工，并用恢复说明启动替代派工。团队提示词和已覆盖的片段会保留；没有覆盖的片段跟随新默认值。升级后检查提示词页里标记“默认内容已更新”的片段。

### 10.2 数据库与回滚

- 新版迁移过的数据库**不能被旧版打开**，所以回滚必须依赖升级前的备份。
- DeepSeek Harness 自身升级后，也要重启正在运行的服务，让面板加载匹配的产物。
- 如果升级的是桌面版 DSH，重启 `superagent` 进程即可；不需要重新创建 profile。

### 10.3 不要重复创建 profile

再次运行 `dsh superagent --from-default-profile web ...` 会替换 profile 并删除 `cordis.patch.yml`。升级时只更新源码、构建、重启，或者编辑现有 profile 的配置。

## 11. 安全边界

请先接受这一点：**dsh-superagent 的隔离不是安全沙箱。**

- 执行器和主 Session 都以运行 DSH 的 OS 用户身份运行，拥有 `danger-full-access`。执行器可以按绝对路径执行程序、读取主目录。
- `gh-agent` / `lark-bot` 只是选择默认凭据，不阻止程序使用其他凭据。
- `blockedCommands` 只拦按名字的调用，用绝对路径仍能运行；它不是安全边界。
- 需要真正隔离时，使用一个只包含智能体凭据的专用 OS 账号来运行 DSH，并限制该账号能访问的目录和网络。
- 一个部署只服务**一个 GitHub 仓库**，且只支持 github.com，不支持 GitHub Enterprise Server。
- 智能体的 GitHub 账号应是专用的；不要让运营者的个人账号出现在评论、提交和 PR 里。
- 智能体不会用自己的账号批准或合并 PR。没有配置额外的合并 CLI 时，由人合并；开启自动合并时，也必须使用另一个有合并权限的账号的 CLI。
- 私有飞书内容不应被复制到 GitHub。提示词和运营流程都应明确这一点。
- 不要把面板或管理 API 暴露到公网；默认只监听 `127.0.0.1`。
- `dsh-superagent-worker` 和 `dsh-superagent-router` 预设只供该应用使用。不要把它们选给个人 Session，也不要禁用它们在 `cordis.patch.yml` 中的配置行，否则恢复 Session 会失败。
- 来源轮询通常有一分钟左右的延迟，不适合需要实时响应的场景。

## 12. 故障排查

部署问题会显示在概览上，并链接到修正位置；日志和管理 API 里是英文错误。下表覆盖最常见的情况。

| 症状 | 原因与处理 |
| --- | --- |
| `The agent GitHub CLI is not set.` | 在 GitHub 卡片上设置智能体 CLI |
| `The agent GitHub CLI was not found.` | 填包装器的绝对路径，或确保命令名在 `PATH` 上 |
| `The agent GitHub CLI cannot read its account` | 用该包装器重新执行 `auth login` |
| `@… cannot read …` / `@… cannot push to …` | 给智能体账号目标仓库的写权限 |
| `The GitHub Source names no repository.` | 在 GitHub 卡片上填 `owner/name` |
| 工单失败，提示 `The prepare command failed: …` | 输出里会写原因（例如克隆路径错）；修好命令后重试工单 |
| `git does not read the github.com credential configuration dsh-superagent writes` | 安装 git 2.32 或更高版本 |
| 工单失败，提示 `The git configuration of the repository in … sets …` | 从该克隆的 `.git/config` 里删除指名的凭据或 `http.*.extraheader` |
| `… was not written by dsh-superagent` | 把指名的文件移出 `<stateDir>/worker-bin` 或 `<stateDir>/worker-gitconfig`，或换一个 `stateDir` |
| `The bot lark-cli allows user identities` | 把飞书来源指向 bot-only 配置（见 [§3.2](#32-飞书机器人可选)） |
| `The bot lark-cli does not work.` / `The bot identity of app … is …` | 检查配置里的 app id 和密钥 |
| `The Feishu bot is not a member of the chat.` | 在群设置里把机器人拉进群，下次尝试会成功 |
| `status` 退出 2 | 某个部分报错，重启不解决；看 `errors` 里的部分和详情，恢复后会自动清除 |
| `status` 退出 3 | 等待补全部署配置；看 `setup` 的缺失项，概览上有链接 |
| `status` 退出 1 | 服务没运行、跑了两份、不响应，或运行时错误；看输出和最新日志 |
| `… timed out after 90 s`、`… was stopped by …`、`… exited with code …`、`… could not run: …` | CLI 没打印原因就失败了；用运营者用户手动运行指名的程序看它自己的报错 |
| 面板提示 dsh-superagent 没有响应 | 正在重启，或配置有误；持续如此时看 DSH 日志 |
| 日志出现 `dsh-superagent configuration has keys it does not use` | 删除指名的旧键；日志会说明每个值现在放在哪里 |
| 日志说插件与当前 dsh 不兼容 | 安装满足 `engines.dsh` 范围的 DeepSeek Harness 版本 |
| 升级 DSH 后面板显示 “Failed to load plugins” | 重启正在运行的实例 |
| 提到智能体后没有任何反应 | 检查名单、来源开关、处理是否暂停；GitHub 上编辑 Issue 正文加入的 @ 不会收到，请用新评论 |

## 13. 速查

### 13.1 路径与命令

| 项目 | 位置/命令 |
| --- | --- |
| profile 目录 | `$DSH_HOME/profiles/superagent`（默认 `~/.dsh/profiles/superagent`） |
| 机器配置 | `$DSH_HOME/profiles/superagent/cordis.patch.yml` |
| 状态目录 | `stateDir`，推荐 `$DSH_HOME/state/superagent` |
| 数据库 | `<stateDir>/orchestrator.sqlite` |
| 工单目录 | `<stateDir>/work`，或 `executor.directory` |
| 命令 shim | `<stateDir>/worker-bin` |
| worker git 配置 | `<stateDir>/worker-gitconfig` |
| 日志 | `<源码目录>/.artifacts/` |
| 面板 | `http://127.0.0.1:3082`，带 token 的完整地址在最新日志里 |
| 管理 API | `http://127.0.0.1:3082/orchestrator/api/` |
| 启动（POSIX） | `scripts/dsh-superagent start` |
| 启动（Windows） | `powershell -NoProfile -ExecutionPolicy Bypass -File scripts\dsh-superagent.ps1 start` |

### 13.2 主要默认值

| 设置 | 默认值 |
| --- | --- |
| 接单模式 `eval.mode` | `observe` |
| 评估时限 `eval.timeoutSeconds` | `120` 秒 |
| 并行工单数 `executor.maxWorkers` | `3` |
| 工单目录根 `executor.directory` | 空，即 `<stateDir>/work` |
| 禁止调用的命令 `executor.blockedCommands` | 空 |
| GitHub 轮询间隔 | `60` 秒 |
| GitHub 连续失败阈值 | `3` 次 |
| GitHub 接收全部消息的四个开关 | 全部关闭 |
| 飞书其他群 | 不接收 |
| 飞书私聊 | 接收 |
| `promptMaxChars` | `100000` |
| `admissionContextMaxPages` | `10` |
| `admissionContextTimeoutMs` | `15000` 毫秒 |

### 13.3 执行环境变量

准备命令以环境变量读取，执行器 Session 在指令中读取同样的值；这些变量**不含任何凭据**。

| 变量 | 含义 |
| --- | --- |
| `DSH_SUPERAGENT_DIR` | 这个工单的执行目录 |
| `DSH_SUPERAGENT_NAME` | 工单短名，也是目录名 |
| `DSH_SUPERAGENT_TICKET` | 工单 id |
| `DSH_SUPERAGENT_BLOCKED_COMMANDS` | 被禁止的命令，空格分隔 |
| `DSH_SUPERAGENT_GITHUB_REPOSITORY` | 来源监听的仓库，`owner/name` |
| `DSH_SUPERAGENT_GITHUB_ACCOUNT` | 智能体 GitHub 账号 |
| `DSH_SUPERAGENT_GITHUB_COMMIT_EMAIL` | 智能体账号的提交邮箱 |
| `DSH_SUPERAGENT_GITHUB_NUMBER` | 发起工单的 Issue/PR 编号 |
| `DSH_SUPERAGENT_GITHUB_PR_HEAD` | 发起工单的 PR 的 head ref（仅 PR 来源） |
| `DSH_SUPERAGENT_FEISHU_BOT` | 飞书机器人标识 |
| `DSH_SUPERAGENT_FEISHU_CHAT` | 发起工单的飞书会话 |

服务脚本另外读取 `DSH_SUPERAGENT_PROFILE` 和 `DSH_SUPERAGENT_URL`。

## 14. 延伸阅读

- dsh-superagent 源码仓库的 `README.md` 和 `SETUP.md`
- `docs/000-philosophy.md` 到 `docs/008-cookbook.md`（建议按顺序阅读）
- `docs/006-deployment.md`：安装、配置、升级和故障排查的完整原文
- [DeepSeek Harness](https://www.deepseek.com/en/harness/)

> 本文依据 dsh-superagent 0.1.0 的源码和部署文档编写，并对照 DeepSeek Harness 0.2.1 的 CLI 与配置做了核对。实现仍在演进，具体行为和默认值以源码和实际面板为准。
