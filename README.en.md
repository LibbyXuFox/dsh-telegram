# DSH Telegram

[简体中文](README.md)

A Telegram bot plugin for DeepSeek Harness, primarily for Linux / WSL2, with Desktop support. Chat with agents and send images and files in private Telegram chats, sharing workspaces and sessions with the Web UI.

Developed against **DeepSeek Harness v0.2.1-rc.1** and installed through DSH's own `dsh plugin` command.

> **Difference from upstream**: this fork adds file delivery to Telegram (see below). Everything else matches `IgnatAi401/dsh-telegram`.

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

## Update

Stop DSH (quit Desktop completely), run the install command again, then start DSH. For a local checkout, `git pull` first and then run `bash install.sh`.

## Uninstall

Stop DSH, then run (append `desktop` for Desktop):

```bash
curl -fsSL https://raw.githubusercontent.com/IgnatAi401/dsh-telegram/main/install.sh | bash -s -- uninstall
```

From this repository you can also run `bash install.sh uninstall`; on Windows Desktop run `dsh plugin --profile desktop remove dsh-telegram`. DSH, model settings, workspaces, and history are preserved; remove the Telegram settings from `.env` yourself.
