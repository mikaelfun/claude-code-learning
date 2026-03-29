# cc-connect Setup Guide — Bridge Claude Code to Feishu (飞书)

cc-connect is a Go-based tool that bridges local AI coding agents (like Claude Code) with messaging platforms (Feishu, Telegram, Slack, etc.). This guide covers the full setup process on Windows, including third-party API provider configuration, auto-start, silent background running, and troubleshooting.

- **GitHub**: https://github.com/chenhg5/cc-connect
- **Version tested**: v1.2.1

## Table of Contents

- [1. Installation](#1-installation)
- [2. Feishu Bot Setup](#2-feishu-bot-setup)
- [3. Configuration](#3-configuration)
- [4. Auto-Start on Login (Windows)](#4-auto-start-on-login-windows)
- [5. Silent Background Running (No CMD Window)](#5-silent-background-running-no-cmd-window)
- [6. Useful Commands](#6-useful-commands)
- [7. Troubleshooting](#7-troubleshooting)
- [8. Configuration Reference](#8-configuration-reference)

---

## 1. Installation

```bash
npm install -g cc-connect
```

Verify:

```bash
cc-connect --version
# cc-connect v1.2.1
```

## 2. Feishu Bot Setup

1. Go to [Feishu Open Platform](https://open.feishu.cn/) and create a new app
2. Enable **Bot** capability
3. Add permissions: `im:message`, `im:message.group_at_msg`, `im:message.p2p_msg`
4. Get your `app_id` and `app_secret` from the app credentials page
5. Subscribe to events: use **WebSocket** mode (no public URL needed)

## 3. Configuration

Config file location: `~/.cc-connect/config.toml`

### Minimal config (direct Anthropic API)

```toml
language = "zh"

[log]
level = "info"

[[projects]]
name = "my-project"
quiet = true  # hide tool call messages in chat

[projects.agent]
type = "claudecode"

[projects.agent.options]
work_dir = "C:\\path\\to\\your\\project"
mode = "auto"  # auto = bypassPermissions

[[projects.platforms]]
type = "feishu"

[projects.platforms.options]
app_id = "cli_xxxxxxxxxxxx"
app_secret = "your_app_secret_here"
```

### With third-party API provider (e.g. NewAPI, OneAPI)

If you use a third-party API proxy instead of the official Anthropic API:

```toml
language = "zh"

[log]
level = "info"

[[projects]]
name = "my-project"
quiet = true

[projects.agent]
type = "claudecode"

[projects.agent.options]
work_dir = "C:\\path\\to\\your\\project"
mode = "auto"

provider = "my-provider"  # must match the provider name below

[[projects.agent.providers]]
name = "my-provider"
api_key = "sk-your-api-key"
base_url = "https://your-api-proxy.com"
model = "claude-sonnet-4-6"

[[projects.platforms]]
type = "feishu"

[projects.platforms.options]
app_id = "cli_xxxxxxxxxxxx"
app_secret = "your_app_secret_here"
```

### Key configuration options

| Option | Description |
|--------|-------------|
| `quiet = true` | Hide thinking/tool-call intermediate messages, only show final response |
| `mode = "auto"` | Auto-approve all tool calls (equivalent to `--permission-mode bypassPermissions`) |
| `provider = "name"` | Select which provider to use for API calls |

## 4. Auto-Start on Login (Windows)

### Step 1: Create startup script

Save as `~/.cc-connect/start-cc-connect.bat`:

```bat
@echo off
REM cc-connect auto-start script (Windows Task Scheduler)

if not exist "%USERPROFILE%\.cc-connect\logs" mkdir "%USERPROFILE%\.cc-connect\logs"

REM Delete stale session files to prevent API 400 errors on restart
if exist "%USERPROFILE%\.cc-connect\sessions" (
    del /Q "%USERPROFILE%\.cc-connect\sessions\*.json" 2>nul
    echo [%date% %time%] Deleted session files >> "%USERPROFILE%\.cc-connect\logs\startup.log"
)

REM Set API env vars for Claude Code SDK (needed for third-party providers)
set "ANTHROPIC_BASE_URL=https://your-api-proxy.com"
set "ANTHROPIC_AUTH_TOKEN=sk-your-api-key"
set "ANTHROPIC_MODEL=claude-sonnet-4-6"

REM Start cc-connect
echo [%date% %time%] Starting cc-connect >> "%USERPROFILE%\.cc-connect\logs\startup.log"
cc-connect >> "%USERPROFILE%\.cc-connect\logs\cc-connect.log" 2>&1
```

> **Important**: The session cleanup step (`del /Q sessions\*.json`) prevents stale sessions from causing API 400 errors after a restart. Always include this.

### Step 2: Create scheduled task

```cmd
schtasks /Create /TN "cc-connect" /TR "wscript.exe \"%USERPROFILE%\.cc-connect\start-cc-connect.vbs\"" /SC ONLOGON /RL HIGHEST
```

### Manual start/stop

```bash
# Start via scheduled task
schtasks /Run /TN "cc-connect"

# Stop
taskkill /IM cc-connect.exe /F

# Check if running
tasklist | findstr cc-connect
```

## 5. Silent Background Running (No CMD Window)

Running the bat file directly opens a visible CMD window. To run silently, use a VBS wrapper.

Save as `~/.cc-connect/start-cc-connect.vbs`:

```vbs
' cc-connect silent launcher — runs bat without visible window
Set WshShell = CreateObject("WScript.Shell")
WshShell.Run """" & WshShell.ExpandEnvironmentStrings("%USERPROFILE%") & "\.cc-connect\start-cc-connect.bat""", 0, False
```

Then update the scheduled task to use the VBS file:

```cmd
schtasks /Change /TN "cc-connect" /TR "wscript.exe \"%USERPROFILE%\.cc-connect\start-cc-connect.vbs\""
```

## 6. Useful Commands

### In-chat commands (send from Feishu)

| Command | Description |
|---------|-------------|
| `/quiet` | Toggle quiet mode (hide/show tool messages) |
| `/new` | Start a new session (clear history) |
| `/sessions` | List all sessions |
| `/help` | Show available commands |

### Cron jobs (scheduled tasks via chat)

```bash
# Add a recurring task
cc-connect cron add --cron "0 6 * * *" --prompt "Summarize GitHub trending" --desc "Daily Trending"

# List cron jobs
cc-connect cron list

# Delete a cron job
cc-connect cron del <job-id>
```

### Provider management

```bash
cc-connect provider list --project my-project
cc-connect provider add --project my-project --name new-provider --api-key sk-xxx --base-url https://api.example.com
```

## 7. Troubleshooting

### Problem: API 400 errors after restart

**Symptom**: Bot replies with `API Error: 400 {"error":{"type":"<nil>","message":"Bad Request..."}}`.
The cc-connect log shows normal `turn complete` with a fixed `response_len=128`.

**Root cause**: Stale session files cause Claude Code to resume an invalid session.

**Fix**: Delete session files before starting:

```bat
del /Q "%USERPROFILE%\.cc-connect\sessions\*.json"
```

This is already included in the startup script above.

### Problem: 400 errors persist even after session cleanup

**Symptom**: Fresh start still returns 400 with certain models.

**Root cause**: Some third-party API providers don't support all Claude Code parameters for certain models (e.g. `claude-opus-4-6` may trigger 400 while `claude-sonnet-4-6` works fine).

**Diagnosis**:
1. Check the session file for the actual error:
   ```bash
   cat ~/.cc-connect/sessions/*.json
   # Look for "API Error: 400" in the history field
   ```
2. Test the API directly with curl:
   ```bash
   curl -X POST "https://your-api.com/v1/messages" \
     -H "Content-Type: application/json" \
     -H "x-api-key: sk-xxx" \
     -H "anthropic-version: 2023-06-01" \
     -d '{"model":"claude-sonnet-4-6","max_tokens":100,"messages":[{"role":"user","content":"hi"}]}'
   ```
3. If curl works but cc-connect doesn't, the issue is with Claude Code CLI's request format (large context, beta headers, etc.)

**Fix**: Switch to a different model in `config.toml`:

```toml
model = "claude-sonnet-4-6"  # instead of claude-opus-4-6
```

### Problem: Multiple cc-connect processes running

**Fix**: Kill all and restart via scheduled task:

```bash
taskkill /IM cc-connect.exe /F
schtasks /Run /TN "cc-connect"
```

### Problem: Feishu WebSocket disconnects

This is normal — cc-connect auto-reconnects. Check the log:

```
[Error] receive message failed, err: read tcp ... wsarecv: An existing connection was forcibly closed
[Info] trying to reconnect: 1
[Info] connected to wss://msg-frontier.feishu.cn/ws/v2?...
```

### Problem: Cron jobs fail with "platform not found"

**Symptom**: `cron: job failed - platform "" not found for session ""`

**Root cause**: Cron job was created in a different session context. The session reference is stale.

**Fix**: Delete and recreate the cron job from an active Feishu chat session.

### Debugging tips

1. **Enable debug logging**: Set `level = "debug"` in config.toml, restart cc-connect
2. **Check session files**: `~/.cc-connect/sessions/*.json` contains chat history and error messages — this is the best place to find the actual error
3. **Log files**:
   - `~/.cc-connect/logs/cc-connect.log` — main log
   - `~/.cc-connect/logs/startup.log` — startup/restart history

## 8. Configuration Reference

Run `cc-connect config-example` to see the full annotated config. Key sections:

```toml
# Global quiet mode
# quiet = true

[log]
level = "info"  # debug, info, warn, error

# Display settings — control intermediate message visibility
# [display]
# thinking_max_len = 300
# tool_max_len = 500

# Streaming preview — real-time typing effect in chat
# [stream_preview]
# enabled = true
# interval_ms = 1500
# min_delta_chars = 30
# max_chars = 2000

# Cron settings
# [cron]
# silent = false  # suppress "task started" notification
```

---

## Summary

| What | How |
|------|-----|
| Install | `npm install -g cc-connect` |
| Config | `~/.cc-connect/config.toml` |
| Start | `schtasks /Run /TN "cc-connect"` |
| Stop | `taskkill /IM cc-connect.exe /F` |
| Logs | `~/.cc-connect/logs/cc-connect.log` |
| Sessions | `~/.cc-connect/sessions/*.json` |
| Debug | Set `level = "debug"` in config, check session JSON for errors |
