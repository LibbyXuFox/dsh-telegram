# DSH Telegram

[简体中文](README.md)

A Telegram bot plugin for DeepSeek Harness, primarily for Linux / WSL2, with Desktop support. Chat with agents and send images and files in private Telegram chats, sharing workspaces and sessions with the Web UI.

Developed against **DeepSeek Harness v0.2.1-rc.1** and installed through DSH's own `dsh plugin` command.

> **Difference from upstream**: this fork adds file delivery, scheduled reminders, and `/usage` `/compact` `/search` (see below). Everything else matches `IgnatAi401/dsh-telegram`.

## Install

CLI (DSH started with `npx @deepseek-ai/dsh`):

```bash
curl -fsSL https://raw.githubusercontent.com/IgnatAi401/dsh-telegram/main/install.sh | bash -s -- install
```

Desktop: quit the app completely, then run:

```bash
curl -fsSL https://raw.githubusercontent.com/IgnatAi401/dsh-telegram/main/install.sh | bash -s -- install desktop
```

If the dsh command is not found, install it first from the app menu (Manage dsh command…). From this repository you can also run `bash install.sh` or `bash install.sh install desktop`.

Windows Desktop: install the command from the app menu (Manage dsh command…), quit the app completely from the tray, then run in PowerShell:

```powershell
dsh plugin --profile desktop add github:IgnatAi401/dsh-telegram
```

You can also add `github:IgnatAi401/dsh-telegram` on the app's Plugins page. For the CLI on Windows, install inside WSL2 as above.

## Configure and use

Create a bot through @BotFather on Telegram, then add to `~/.dsh/.env` (`%USERPROFILE%\.dsh\.env` on Windows):

```
DSH_TELEGRAM_TOKEN=<your BotFather token>
DSH_TELEGRAM_ALLOWED_USER_IDS=<your numeric Telegram user ID>
```

Separate multiple user IDs with commas. Only private chats and allowed users are supported.

Start DSH: run `npx @deepseek-ai/dsh web` for the CLI, or open the Desktop app. Configure model credentials and create a workspace in the Web UI, then open a private chat with the bot:

- `/menu`: select a workspace, create, open or archive sessions, and switch models and reasoning effort.
- `/help`: list available commands.
- `/usage`: show the session's context pressure and the last call's tokens.
- `/compact`: compact the session's history on demand (while idle).
- `/search <keyword>`: search every past session.
- `/schedule`: manage reminders bound to the current session (see below).
- Send text, images, or files to start chatting.

Do not run multiple bot processes with the same token. For another profile, append its name to the install and uninstall commands and start it with `npx @deepseek-ai/dsh --profile NAME`; with a custom `DSH_HOME`, put `.env` in that directory.

## Sending files to Telegram (added by this fork)

Upstream replies in text only, so any chart, report, or PDF the agent produces stays on the machine. This fork adds file delivery.

**Ask the agent to write a standalone line in its final reply:**

```
[[send: /absolute/path/report.pdf]]
```

The plugin removes that line from the body (the user never sees it), uploads the file as a **Telegram document**, and then sends the cleaned text. One file per line; multiple lines are allowed.

**Allowed locations:**

| Source | Meaning |
|---|---|
| **The current workspace** | Always allowed |
| `sendFileRoots` | Extra allowed roots, empty by default; `~/Desktop` style paths are supported |

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

**Enforced limits:**

| Check | Behaviour |
|---|---|
| The path is resolved through `realpath` | A symlink cannot escape |
| Must sit inside an allowed root (workspace ∪ `sendFileRoots`) | Otherwise refused, with a note in the reply |
| Must be a regular file | Directories and device files are refused |
| At most **50 MiB** per file | Larger files are refused (public Bot API ceiling) |
| Upload failure | The text is still delivered, followed by a notice |

**Why not simply allow every path**: the model picks the absolute path, so an unrestricted upload is a way to send `~/.ssh/id_rsa` out. Declaring `sendFileRoots` makes the shared directories an explicit choice.

**Where it lives:**

| File | Content |
|---|---|
| `src/client.ts` | `sendDocument` (multipart upload) and a raised upload timeout (≥120 s) |
| `src/bridge.ts` | `extractFileDirectives()` parses the directive, `sendWorkspaceFiles()` validates and uploads, `deliverResult()` uploads before sending the text |
| `src/index.ts` | The `sendFileRoots` config field |

**Note**: the directive must be its own line, starting with `[[send:` and ending with `]]`; inside a sentence it is not recognised.

## Scheduled reminders (added by this fork)

A reminder binds to the **current session**, so its result is delivered back into the same Telegram chat — no extra routing.

| Usage | Effect |
| :--- | :--- |
| `/schedule` | List this session's reminders |
| `/schedule in 30m <prompt>` | Once, after 30 minutes |
| `/schedule in 2h <prompt>` | Once, after 2 hours |
| `/schedule every 2h <prompt>` | Recurring every 2 hours |
| `/schedule daily 08:00 <prompt>` | Daily at 08:00 (the host's IANA zone) |
| `/schedule del <id>` | Delete one |

Units: `m` minutes, `h` hours, `d` days. Input that cannot be parsed shows the syntax instead of guessing.

**Requires** `@deepseek-ai/dsh-schedule` (mounted by the standard composition). Without it the command says so.

## Usage and history (added by this fork)

| Command | Effect | Service used |
| :--- | :--- | :--- |
| `/usage` | Context pressure plus the last call's input/output/cache/reasoning tokens | `tokenMeter` |
| `/compact` | Compact history on demand, reporting folded nodes and saved tokens; refuses while a task runs | `compaction` |
| `/search <keyword>` | Search every past session, one best snippet per hit | `sessionQuery` |

**All three services are optional**: when a deployment lacks one, the command explains that it is not enabled instead of failing.

### ⚠️ `/search` must be enabled in DSH first (it is off by default)

The base composition ships full-text session search as **opt-in**:

```yaml
- id: session-query-sqlite
  name: '@deepseek-ai/dsh-session-query-sqlite'
  config:
    path: ':memory:'
    openAt: never          # ← off by default; search returns SESSION_QUERY_SEARCH_DISABLED
```

**Symptom**: `/search` answers "⚠️ 搜索失败 · session search is disabled: … openAt \"never\"".

**To enable it**, add an override in `~/.dsh/profiles/<profile>/cordis.patch.yml`:

```yaml
- id: session-query-sqlite
  name: '@deepseek-ai/dsh-session-query-sqlite'
  config:
    path: !!js dshHomePath('session-query.sqlite')   # durable; not rebuilt on restart
    openAt: first-search                             # startup | first-search | never
```

Then **quit DSH completely (⌘Q) and restart**. `/usage` and `/compact` do not need this.

## Update

Stop DSH (quit Desktop completely), run the install command again, then start DSH. For a local checkout, `git pull` first and then run `bash install.sh`.

## Uninstall

Stop DSH, then run (append `desktop` for Desktop):

```bash
curl -fsSL https://raw.githubusercontent.com/IgnatAi401/dsh-telegram/main/install.sh | bash -s -- uninstall
```

From this repository you can also run `bash install.sh uninstall`; on Windows Desktop run `dsh plugin --profile desktop remove dsh-telegram`. DSH, model settings, workspaces, and history are preserved; remove the Telegram settings from `.env` yourself.
