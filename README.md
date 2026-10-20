# DSH Telegram

[English](README.en.md)

DeepSeek Harness 的 Telegram 机器人插件，主要用于 Linux / WSL2，也支持桌面版。在 Telegram 私聊中与 Agent 对话、发送图片和文件，并与 Web UI 共享工作区和会话。

基于 **DeepSeek Harness v0.2.1-rc.1** 开发，通过 DSH 自带的 `dsh plugin` 安装。

> **本 fork 相对上游的差异**：新增「发文件回 Telegram」「定时提醒」「/usage /compact /search」（见下）。其余与上游 `IgnatAi401/dsh-telegram` 一致。

## 安装

命令行版（`npx @deepseek-ai/dsh` 启动）：

```bash
curl -fsSL https://raw.githubusercontent.com/LibbyXuFox/dsh-telegram/main/install.sh | bash -s -- install
```

桌面版：先完全退出 App，再运行：

```bash
curl -fsSL https://raw.githubusercontent.com/LibbyXuFox/dsh-telegram/main/install.sh | bash -s -- install desktop
```

如提示找不到 dsh 命令，先在 App 菜单的“管理 dsh 命令…”中安装。在本仓库目录中也可以运行 `bash install.sh` 或 `bash install.sh install desktop`。

Windows 桌面版：在 App 菜单的“管理 dsh 命令…”中安装命令，从托盘完全退出 App，再在 PowerShell 中运行：

```powershell
dsh plugin --profile desktop add github:LibbyXuFox/dsh-telegram
```

也可以在 App 的“插件”页添加 `github:LibbyXuFox/dsh-telegram`。Windows 上的命令行版请在 WSL2 中按上文安装。

## 配置与使用

通过 Telegram 的 @BotFather 创建机器人，然后在 `~/.dsh/.env`（Windows 为 `%USERPROFILE%\.dsh\.env`）中写入：

```
DSH_TELEGRAM_TOKEN=<BotFather 提供的 token>
DSH_TELEGRAM_ALLOWED_USER_IDS=<你的 Telegram 数字用户 ID>
```

多个用户 ID 用逗号分隔。仅支持私聊，只有允许的用户可以访问。

启动 DSH：命令行版运行 `npx @deepseek-ai/dsh web`，桌面版直接打开 App。在 Web UI 配置模型凭据并创建工作区后，私聊机器人：

- `/menu`：选择工作区，新建、打开或归档会话，切换模型和推理强度。
- `/help`：查看可用命令。
- `/usage`：查看当前会话的上下文用量与最近一次调用的 token。
- `/compact`：手动压缩当前会话的历史（任务空闲时可用）。
- `/search 关键词`：在所有历史会话里搜索。
- `/schedule`：管理绑定到当前会话的定时提醒（见下）。
- 直接发送文字、图片或文件开始对话。

同一个 token 不要同时运行多个机器人进程。使用其他运行配置时，在安装和卸载命令末尾写上配置名，并用 `npx @deepseek-ai/dsh --profile 配置名` 启动；自定义 `DSH_HOME` 时，`.env` 放在该目录下。

## 发文件回 Telegram（本 fork 新增）

上游只能回复文本，Agent 生成的图表、报告、PDF 留在机器上发不出去。本 fork 增加了文件投递。

**让 Agent 在最终回复里单独一行写：**

```
[[send: /绝对路径/报告.pdf]]
```

插件会：① 把这一行从正文中**移除**（不会显示给用户）② 把文件作为 **Telegram 文档**上传
③ 然后发送清理后的正文。一行一个文件，可以写多行。

**允许的目录：**

| 来源 | 说明 |
|---|---|
| **当前工作区** | 永远允许 |
| `sendFileRoots` | 额外允许的目录，默认空；支持 `~/Desktop` 这样的写法 |

```yaml
- id: telegram
  name: 'dsh-telegram'
  config:
    token: "..."
    allowedUserIds: [1009204960]
    sendFileRoots:
      - "~/Desktop"
      - "~/Downloads"
```

**边界（插件强制）：**

| 检查 | 行为 |
|---|---|
| 路径必须经 `realpath` 解析 | 软链接不能绕过 |
| 必须在允许的目录内（工作区 ∪ `sendFileRoots`） | 否则拒绝，并在回复里提示 |
| 必须是普通文件 | 目录 / 设备文件拒绝 |
| 单文件 ≤ **50 MiB** | 超限拒绝（公共 Bot API 上限） |
| 上传失败 | 正文照常发送，另附提示 |

**为什么不直接放开所有路径**：模型可以挑任意绝对路径，不限制就等于给它一条把
`~/.ssh/id_rsa` 发出去的路。显式声明 `sendFileRoots` = 你清楚知道哪些目录被共享了。

**实现位置**：

| 文件 | 内容 |
|---|---|
| `src/client.ts` | `sendDocument`（multipart 上传）+ 上传超时放宽到 ≥120s |
| `src/bridge.ts` | `extractFileDirectives()` 解析指令、`sendWorkspaceFiles()` 校验并上传、`deliverResult()` 先传文件再发正文 |
| `src/index.ts` | `sendFileRoots` 配置项 |

**注意**：指令行必须**独占一行**且以 `[[send:` 开头、`]]` 结尾；写在句子中间不会被识别。

## 定时提醒（本 fork 新增）

提醒**绑定到当前会话**，所以触发时结果会发回同一个 Telegram 对话——不需要额外路由。

| 用法 | 作用 |
| :--- | :--- |
| `/schedule` | 列出本会话的提醒 |
| `/schedule in 30m 内容` | 30 分钟后一次 |
| `/schedule in 2h 内容` | 2 小时后一次 |
| `/schedule every 2h 内容` | 每 2 小时 |
| `/schedule daily 08:00 内容` | 每天 08:00（宿主机的 IANA 时区） |
| `/schedule del <编号>` | 删除一条 |

单位：`m` 分钟、`h` 小时、`d` 天。无法解析的输入会显示语法说明，不会瞎猜。

**依赖**：`@deepseek-ai/dsh-schedule`（标准组合已挂载）。未挂载时命令会明确提示。

## 用量与历史（本 fork 新增）

| 命令 | 作用 | 依赖的服务 |
| :--- | :--- | :--- |
| `/usage` | 上下文用量 + 最近一次调用的 input/output/cache/reasoning tokens | `tokenMeter` |
| `/compact` | 手动压缩历史，报告折叠节点数与节省的 token；任务运行中会拒绝 | `compaction` |
| `/search 关键词` | 搜索全部历史会话。命中显示为「**工作区 — 会话名**」并带「打开」按钮，**点一下直接进入该会话** | `sessionQuery` |

**`/search` 的标签**：会话名从会话日志里折叠出来，所以**已归档的会话也保留原名**；不属于任何工作区的会话退化为显示 id，且不给按钮。

**`/compact` 的结果**：`ManualCompactionError` 有六种 code，只有两种是真问题：

| code | 含义 | 严重程度 |
| :--- | :--- | :--- |
| `busy` | 有轮次正占用会话 | ℹ️ 等任务结束 |
| `cancelled` | 请求被中止，历史未改动 | ℹ️ 无影响 |
| `changed` | 压缩期间历史被修改，已放弃 | ℹ️ 重试即可 |
| `summary` | **摘要没能比原文更短**，所以没提交 | ℹ️ **不是错误** |
| `commit` | 未干净提交，历史可能有变化 | ⚠️ 需确认状态 |
| `persistence` | 压缩完成但落盘失败 | ⚠️ 见 DSH 日志 |

> 看到「摘要没能更短」是正常的：服务在**拒绝一次没有收益的压缩**。常见于会话还不够长，或内容已经很紧凑。

**三个服务都是可选的**：部署里没有时，命令会说明"未启用"，而不是报错。

### ⚠️ `/search` 需要先在 DSH 里开启（默认是关的）

DSH 基础组合把全文会话搜索设成 **opt-in**：

```yaml
- id: session-query-sqlite
  name: '@deepseek-ai/dsh-session-query-sqlite'
  config:
    path: ':memory:'
    openAt: never          # ← 默认关闭，搜索返回 SESSION_QUERY_SEARCH_DISABLED
```

**症状**：`/search` 回「⚠️ 搜索失败 · session search is disabled: … openAt "never"」。

**开启方式**：在 `~/.dsh/profiles/<profile>/cordis.patch.yml` 里加一条覆盖：

```yaml
- id: session-query-sqlite
  name: '@deepseek-ai/dsh-session-query-sqlite'
  config:
    path: !!js dshHomePath('session-query.sqlite')   # durable，重启后不重建
    openAt: first-search                             # startup | first-search | never
```

改完**必须 ⌘Q 重启 DSH**。`/usage` 与 `/compact` 不需要这一步。

## 更新

停止 DSH（桌面版需完全退出），重新执行安装命令，再启动 DSH。使用本地仓库时，先 `git pull` 再运行 `bash install.sh`。

## 卸载

停止 DSH 后运行（桌面版在末尾加 `desktop`）：

```bash
curl -fsSL https://raw.githubusercontent.com/LibbyXuFox/dsh-telegram/main/install.sh | bash -s -- uninstall
```

在本仓库目录中也可以运行 `bash install.sh uninstall`；Windows 桌面版运行 `dsh plugin --profile desktop remove dsh-telegram`。卸载保留 DSH、模型配置、工作区和历史记录；`.env` 中的 Telegram 配置需自行删除。
