# claude-to-im Setup Guide — Bridge Claude Code to Feishu (飞书)

claude-to-im is a Node.js-based Claude Code skill that bridges Claude Code / Codex to IM platforms (Telegram, Discord, Feishu/Lark, QQ, WeChat). It uses the `@anthropic-ai/claude-agent-sdk` to spawn Claude Code CLI as a subprocess, supporting streaming responses, tool-call progress display, and interactive permission approval buttons in chat.

- **GitHub**: https://github.com/op7418/Claude-to-IM (core lib) + https://github.com/op7418/Claude-to-IM-skill (skill)
- **Version tested**: 0.1.0 (skill) + claude-agent-sdk 0.2.88
- **Compared to cc-connect**: claude-to-im is a Claude Code native skill (runs via `/claude-to-im start`), supports streaming cards, tool progress display, and permission approval buttons in Feishu. cc-connect is a standalone Go binary with simpler setup but fewer features.

## Table of Contents

- [1. Installation](#1-installation)
- [2. Feishu Bot Setup](#2-feishu-bot-setup)
- [3. Configuration](#3-configuration)
- [4. Starting and Stopping](#4-starting-and-stopping)
- [5. Useful Commands](#5-useful-commands)
- [6. Windows Gotchas (8 Bugs Fixed)](#6-windows-gotchas-8-bugs-fixed)
- [7. Troubleshooting](#7-troubleshooting)
- [8. claude-to-im vs cc-connect](#8-claude-to-im-vs-cc-connect)

---

## 1. Installation

claude-to-im is installed as a Claude Code skill. Clone the repo into your skills directory:

```bash
# Two repos needed: core library + skill wrapper
# On Windows (NTFS case-insensitive), cannot have Claude-to-IM/ and claude-to-im/ in same dir
# So rename skill dir to claude-to-im-skill/
#
# Clone to Claude Code skills directory
cd ~/.claude/skills
git clone https://github.com/anthropics/claude-to-im.git

# Install dependencies and build
cd claude-to-im
npm install
npm run build
```

Verify the build output:

```bash
ls dist/daemon.mjs
# dist/daemon.mjs should exist
```

### Prerequisites

| Requirement | Version | Check |
|-------------|---------|-------|
| Node.js | >= 20 | `node --version` |
| Claude Code CLI | >= 2.x | `claude --version` |
| npm | any | `npm --version` |

> **Important**: On Windows, if Claude Code was installed via npm (not the native installer), the CLI entry point is a `.js` file (`cli.js`), not a native binary. This matters for configuration — see [Section 3](#3-configuration).

---

## 2. Feishu Bot Setup

Feishu setup requires **two publish cycles** because event subscription validation needs an active WebSocket connection from the bridge.

### Phase 1: Create app + permissions + bot (before starting bridge)

**Step 1 — Create app**

1. Go to [Feishu Open Platform](https://open.feishu.cn/app)
2. Click **"Create Custom App"** (创建自定义应用)
3. Fill in app name and description, click **"Create"**
4. Note the **App ID** (`cli_xxxxxxxxxx`) and **App Secret** from "Credentials & Basic Info"

**Step 2 — Batch-add permissions**

1. Go to **"Permissions & Scopes"** (权限管理)
2. Click **"Batch switch to configure by dependency"** (or find the JSON editor)
3. Paste this JSON:

```json
{
  "scopes": {
    "tenant": [
      "im:message:send_as_bot",
      "im:message:readonly",
      "im:message.p2p_msg:readonly",
      "im:message.group_at_msg:readonly",
      "im:message:update",
      "im:message.reactions:read",
      "im:message.reactions:write_only",
      "im:chat:read",
      "im:resource",
      "cardkit:card:write",
      "cardkit:card:read"
    ],
    "user": []
  }
}
```

4. Click **"Save"**

> If batch import is unavailable, add each scope manually via the search box.

**Step 3 — Enable bot**

1. Go to **"Add Features"** (添加应用能力) → enable **"Bot"** (机器人)
2. Set bot name and description

**Step 4 — First publish**

1. Go to **"Version Management & Release"** (版本管理与发布)
2. Create version `1.0.0`, fill description, **"Save"** → **"Submit for Review"**
3. Admin approves in **Feishu Admin Console** → **App Review**

> The bot will NOT work until this version is approved.

### Phase 2: Event subscription (requires running bridge)

**Step 5 — Start the bridge**

```bash
# In Claude Code
/claude-to-im start
```

This establishes the WebSocket long connection that Feishu needs.

**Step 6 — Configure events & callbacks**

1. Go to **"Events & Callbacks"** (事件与回调)
2. Set **"Event Dispatch Method"** to **"Long Connection"** (长连接)
3. Add event: `im.message.receive_v1` (接收消息)
4. Add callback: `card.action.trigger` (卡片交互回调，用于权限审批按钮)
5. Click **"Save"**

> If you get "未检测到应用连接信息" (connection not detected), make sure the bridge is running.

**Step 7 — Second publish**

1. Create version `1.1.0`, submit for review, admin approves
2. After approval, the bot can receive and respond to messages

> **Rule**: Any change to permissions, events, or capabilities requires a new version publish + admin approval.

---

## 3. Configuration

Config file location: `~/.claude-to-im/config.env`

### Setup wizard (recommended)

In Claude Code, run:

```
/claude-to-im setup
```

The wizard collects credentials one at a time and writes `config.env` for you.

### Manual configuration

Create `~/.claude-to-im/config.env`:

```bash
# Runtime backend: claude (default), codex, auto
CTI_RUNTIME=claude

# Enabled channels (comma-separated): telegram, discord, feishu, qq, weixin
CTI_ENABLED_CHANNELS=feishu

# Default working directory for Claude Code
CTI_DEFAULT_WORKDIR=C:\path\to\your\project

# Default mode: code (default), plan, ask
CTI_DEFAULT_MODE=code

# Claude CLI path — CRITICAL on Windows (see note below)
CTI_CLAUDE_CODE_EXECUTABLE=C:\Users\YourName\AppData\Roaming\npm\node_modules\@anthropic-ai\claude-code\cli.js

# ── Feishu / Lark ──
CTI_FEISHU_APP_ID=cli_xxxxxxxxxx
CTI_FEISHU_APP_SECRET=your_app_secret_here
CTI_FEISHU_DOMAIN=https://open.feishu.cn

# Auto-approve all tool permissions (no approval buttons needed)
# Recommended for personal use — skips permission popups in Feishu
CTI_AUTO_APPROVE=true
```

### Windows: Finding the correct CLI path

On Windows, if Claude Code was installed via npm, you **must** point to the actual `cli.js` file, not the npm shim:

```bash
# WRONG — this is the npm shim (no .js extension), SDK treats it as native binary → ENOENT
CTI_CLAUDE_CODE_EXECUTABLE=C:\Users\YourName\AppData\Roaming\npm\claude

# CORRECT — this is the actual entry point, SDK detects .js and uses node
CTI_CLAUDE_CODE_EXECUTABLE=C:\Users\YourName\AppData\Roaming\npm\node_modules\@anthropic-ai\claude-code\cli.js
```

To find the correct path:

```bash
# Method 1: Check where npm installed it
npm list -g @anthropic-ai/claude-code --parseable
# Append /cli.js to the output

# Method 2: Find the .js file directly
ls "$(npm root -g)/@anthropic-ai/claude-code/cli.js"
```

> **Why this matters**: The Claude Agent SDK uses a function called `KI()` to decide if a path is a native binary or a Node.js script. It checks if the path ends in `.js`/`.mjs`. If the path does NOT end in `.js` (like the npm shim `npm\claude`), the SDK tries to execute it as a native binary and fails with "Claude Code native binary not found". Pointing to `cli.js` makes the SDK spawn `node cli.js` correctly.

### Directory structure

After setup, your `~/.claude-to-im/` directory looks like:

```
~/.claude-to-im/
├── config.env          # Main configuration (chmod 600)
├── data/
│   └── messages/       # Message history (per-channel JSON files)
├── logs/
│   ├── bridge.log      # Main daemon log (buffered writes)
│   └── bridge-stderr.log  # Stderr capture
└── runtime/
    ├── bridge.pid      # PID of running daemon
    └── status.json     # Runtime status (channels, runId, etc.)
```

### Set file permissions (Linux/macOS)

```bash
chmod 600 ~/.claude-to-im/config.env
```

---

## 4. Starting and Stopping

### Via Claude Code skill (recommended)

```
/claude-to-im start     # Start the bridge daemon
/claude-to-im stop      # Stop the bridge daemon
/claude-to-im status    # Check if bridge is running
/claude-to-im logs      # View last 50 log lines
/claude-to-im logs 200  # View last 200 log lines
/claude-to-im doctor    # Run diagnostics
```

### Via shell directly

```bash
# Start
bash ~/.claude/skills/claude-to-im/scripts/daemon.sh start

# Stop
bash ~/.claude/skills/claude-to-im/scripts/daemon.sh stop

# Status
bash ~/.claude/skills/claude-to-im/scripts/daemon.sh status

# Logs
bash ~/.claude/skills/claude-to-im/scripts/daemon.sh logs 100
```

### Windows: PowerShell directly

```powershell
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\.claude\skills\claude-to-im\scripts\supervisor-windows.ps1" -Command start
```

### How it runs

| Platform | Mechanism | Details |
|----------|-----------|---------|
| macOS | launchd | `~/Library/LaunchAgents/com.claude-to-im.bridge.plist` |
| Linux | setsid + nohup | Detached background process |
| Windows | `Start-Process -WindowStyle Hidden` | Detached hidden process, survives terminal close |
| Windows (service) | WinSW or NSSM | Optional, via `install-service` command |

The daemon process is fully detached — closing your terminal or Claude Code session does NOT kill the bridge.

---

## 5. Useful Commands

### In-chat commands (send from Feishu to the bot)

| Command | Description |
|---------|-------------|
| `/new` | Start a new conversation session |
| `/mode code\|plan\|ask` | Switch Claude Code mode |
| `/model <name>` | Switch model |
| `/sessions` | List recent sessions |
| `/resume <id>` | Resume a previous session |
| `/perm allow <id>` | Approve a pending permission |
| `/perm deny <id>` | Deny a pending permission |
| `1` / `2` / `3` | Quick permission reply (allow / allow+remember / deny) |

### Feishu streaming cards

When you send a message, the bot creates a streaming card:

1. Shows "Thinking..." while Claude processes
2. Streams response text in real-time (updated every ~1.5 seconds)
3. Shows tool-call progress (e.g., "Reading file...", "Running bash...")
4. Permission requests show inline approval/deny buttons

---

## 6. Windows Gotchas (8 Bugs Fixed)

Getting claude-to-im running on Windows required fixing 8 bugs in the skill code. These are all Windows-specific issues that don't affect macOS/Linux.

### Bug 1: CLI preflight silent crash

**Symptom**: Bridge starts then immediately exits with code 1. bridge.log is empty.

**Root cause**: `getCliVersion()` runs `"path/to/cli.js" --version` which doesn't work on Windows (`.js` file association doesn't capture stdout). Returns empty string → preflight fails → `process.exit(1)` runs before log buffer flushes.

**Fix**: When `cliPath` ends in `.js`/`.mjs`, prepend `node`:

```typescript
const cmd = cliPath.match(/\.(m?js)$/i)
  ? `node "${cliPath}" --version`
  : `"${cliPath}" --version`;
```

### Bug 2: PowerShell Join-Path 3 arguments

**Symptom**: `Join-Path : A positional parameter cannot be found that accepts argument 'bridge.log'`

**Root cause**: Windows PowerShell 5.1's `Join-Path` only accepts 2 arguments (PowerShell 7+ accepts more).

**Fix**: Nest calls: `Join-Path (Join-Path $CtiHome 'logs') 'bridge.log'`

### Bug 3: PowerShell $PID read-only

**Symptom**: `Cannot overwrite variable PID because it is read-only`

**Root cause**: `$PID` is a PowerShell automatic variable (current process ID).

**Fix**: Rename all `$pid` to `$bridgePid`.

### Bug 4: config.env not loaded by PowerShell

**Symptom**: Bridge starts but all config values are empty, CLI path not found.

**Root cause**: `daemon.sh` delegates to PowerShell on Windows BEFORE sourcing `config.env`. The `set -a && source` never runs.

**Fix**: Added `Load-ConfigEnv` function in `supervisor-windows.ps1` to parse config.env before spawning the daemon.

### Bug 5: Startup timeout too short

**Symptom**: `Failed to start bridge` even though the process is alive.

**Root cause**: 3-second timeout, but Feishu WebSocket handshake takes ~3.5 seconds.

**Fix**: Increased to 8 seconds.

### Bug 6: loadConfig() doesn't set process.env

**Symptom**: `resolveClaudeCliPath()` can't find the configured CLI path.

**Root cause**: Known bug (#87) — `loadConfig()` parses `config.env` into an internal Map but never sets `process.env`. The SDK and `resolveClaudeCliPath()` read from `process.env`.

**Fix**: Added env injection in `main.ts` to manually read config.env and set `process.env`:

```typescript
try {
  const envFile = fs.readFileSync(path.join(CTI_HOME, 'config.env'), 'utf-8');
  for (const line of envFile.split('\n')) {
    // ... parse KEY=VALUE and set process.env[KEY]
  }
} catch { /* config.env may not exist */ }
```

### Bug 7: checkRequiredFlags() same issue as Bug 1

**Symptom**: `--help` command also fails silently for `.js` paths.

**Fix**: Same `node` prefix for `.js`/`.mjs` paths.

### Bug 8: isExecutable() fails for .js files on Windows (ROOT CAUSE)

**Symptom**: `Error: Claude Code native binary not found at C:\...\npm\claude`

**Root cause chain**:

```
config.env sets CTI_CLAUDE_CODE_EXECUTABLE=...cli.js
  → loadConfig() doesn't set process.env (Bug 6, fixed)
  → main.ts env injection sets process.env (Fix 6)
  → resolveClaudeCliPath() reads env var ✓
  → isExecutable(cli.js) uses fs.accessSync(path, X_OK) → false on Windows!
  → Falls back to findAllInPath() → returns npm shim "npm\claude" (no .js)
  → SDK's KI() function: path doesn't end in .js → treats as native binary
  → Spawns "npm\claude" as native binary → ENOENT
  → "Claude Code native binary not found"
```

**Fix**: Use `R_OK` instead of `X_OK` on Windows:

```typescript
function isExecutable(p: string): boolean {
  try {
    fs.accessSync(p, process.platform === 'win32' ? fs.constants.R_OK : fs.constants.X_OK);
    return true;
  } catch { return false; }
}
```

**After fix**: `isExecutable()` returns true → returns cli.js path → SDK's `KI()` sees `.js` → spawns `node cli.js` → success.

### Files modified

| File | Changes |
|------|---------|
| `src/llm-provider.ts` | Bug 1, 7, 8: `node` prefix for `.js` paths + `R_OK` on Windows |
| `src/main.ts` | Bug 6: env injection workaround |
| `scripts/supervisor-windows.ps1` | Bug 2, 3, 4, 5: PowerShell 5.1 compatibility |
| `scripts/daemon.sh` | Bug 4: bash→PowerShell argument passing |

---

## 7. Troubleshooting

### Bridge starts but exits immediately (empty log)

This is Bug 1. The `setupLogger()` replaces `console.log/error` with a buffered `fs.WriteStream`. When `process.exit()` runs before the buffer flushes, both bridge.log and stderr are empty.

**Diagnosis**:

```bash
# Run directly (not as daemon) to see output
cd ~/.claude/skills/claude-to-im
node dist/daemon.mjs
```

### "Claude Code native binary not found"

This is Bug 8 (the root cause). Make sure:

1. `CTI_CLAUDE_CODE_EXECUTABLE` in config.env points to `cli.js`, not the npm shim
2. The skill code has the `R_OK` fix applied

### "Prompt is too long"

The conversation history is too large for the model's context window. Send `/new` in Feishu to start a fresh session.

### Feishu: "未检测到应用连接信息" when saving events

The bridge is not running. Start it first with `/claude-to-im start`, then configure events.

### No response from bot (message not received)

1. Check event subscription: must have `im.message.receive_v1` + long connection mode
2. Check callback: must have `card.action.trigger`
3. Check version published: all permission/event changes require a new version + admin approval
4. Check bridge logs: `bash ~/.claude/skills/claude-to-im/scripts/daemon.sh logs 50`

### Permission buttons not working in Feishu

Missing permissions or callback. Add:
- Permissions: `cardkit:card:write`, `cardkit:card:read`
- Callback: `card.action.trigger`
- Then publish a new version and restart bridge

### Bridge running but log file never updates

The `setupLogger()` uses buffered `fs.WriteStream`. Logs may be delayed. Check the file size and modification time to see if writes are happening. For real-time debugging, run the daemon directly in foreground:

```bash
cd ~/.claude/skills/claude-to-im
node dist/daemon.mjs
```

### Doctor script

Run diagnostics:

```bash
bash ~/.claude/skills/claude-to-im/scripts/doctor.sh
```

Or in Claude Code:

```
/claude-to-im doctor
```

---

## 8. claude-to-im vs cc-connect

| Feature | claude-to-im | cc-connect |
|---------|-------------|------------|
| Type | Claude Code skill | Standalone Go binary |
| Install | `git clone` + `npm install` | `npm install -g cc-connect` |
| Start | `/claude-to-im start` (in Claude Code) | `cc-connect` (or scheduled task) |
| Config format | `~/.claude-to-im/config.env` (KEY=VALUE) | `~/.cc-connect/config.toml` (TOML) |
| SDK | `@anthropic-ai/claude-agent-sdk` (official) | Custom Claude Code CLI wrapper |
| Streaming | Feishu streaming cards with real-time updates | Optional stream preview |
| Tool progress | Shows tool name + status in card | Configurable, can hide |
| Permission UI | Inline buttons in Feishu card | N/A (auto-approve only) |
| Platforms | Telegram, Discord, Feishu, QQ, WeChat | Feishu, Telegram, Slack, DingTalk |
| Windows support | Works but requires 8 bug fixes (see above) | Works out of the box |
| Daemon management | Platform-specific (launchd/setsid/PowerShell) | Manual (scheduled task + VBS) |
| Session management | Built-in per-channel sessions | Built-in with resume |
| Third-party API | Via env vars (ANTHROPIC_BASE_URL etc.) | Via config.toml providers |
| Quiet mode | N/A (streams everything) | `quiet = true` hides tool messages |
| Maturity | Early (0.1.0), rough on Windows | Stable (1.2.x), works on all platforms |

### When to use which

- **claude-to-im**: If you want streaming cards, tool progress display, and permission buttons in Feishu. Willing to fix Windows bugs.
- **cc-connect**: If you want a simple, stable setup that works immediately on Windows. Don't need streaming/permission features.

---

## Summary

| What | How |
|------|-----|
| Install | `cd ~/.claude/skills && git clone ... && cd claude-to-im && npm install && npm run build` |
| Config | `~/.claude-to-im/config.env` |
| Setup wizard | `/claude-to-im setup` (in Claude Code) |
| Start | `/claude-to-im start` |
| Stop | `/claude-to-im stop` |
| Status | `/claude-to-im status` |
| Logs | `/claude-to-im logs` |
| Diagnose | `/claude-to-im doctor` |
| Key Windows fix | Set `CTI_CLAUDE_CODE_EXECUTABLE` to `cli.js` path (not npm shim) |

---

## 9. Additional Findings (2026-04 Redeployment)

### Bug 3 — Scope was too narrow

The original Bug 3 fix only renamed `$pid` in the `start` block. But `$pid` (PowerShell's read-only automatic variable) is also used in:
- `Test-PidAlive` function **parameter**: `param([string]$Pid)` → must rename to `$ProcessId`
- `stop` block: `$pid = Read-Pid` → `$bridgePid = Read-Pid`
- `status` block: `$pid = Read-Pid` → `$bridgePid = Read-Pid`

**All** occurrences of `$pid` must be globally replaced, not just the `start` block.

### Bug 9: Start-Process stdout/stderr same file

**Symptom**: `Start-Process : This command cannot be run because "RedirectStandardOutput" and "RedirectStandardError" are same.`

**Root cause**: PowerShell's `Start-Process` does not allow `-RedirectStandardOutput` and `-RedirectStandardError` to point to the same file.

**Fix**: Use separate files:

```powershell
$StderrLog = Join-Path (Join-Path $CtiHome 'logs') 'bridge-stderr.log'

$proc = Start-Process -FilePath $nodePath `
    -ArgumentList $DaemonMjs `
    -WorkingDirectory $SkillDir `
    -WindowStyle Hidden `
    -RedirectStandardOutput $LogFile `
    -RedirectStandardError $StderrLog `
    -PassThru
```

### Two-repo architecture

The skill's `package.json` declares `"claude-to-im": "file:../Claude-to-IM"` — a local file dependency pointing to the core library one level up. Both repos must be cloned side by side:

```
parent-dir/
├── Claude-to-IM/           ← core library (npm install + build first)
└── claude-to-im-skill/     ← skill wrapper (npm install + build second)
```

**Windows gotcha**: NTFS is case-insensitive. `Claude-to-IM/` and `claude-to-im/` resolve to the same directory. Rename the skill to `claude-to-im-skill/` to avoid the collision.

### Symlink unreliable on Windows

Git symlinks (even with `core.symlinks=true`) don't reliably work for skill directories on Windows. After renaming the target directory, the symlink may break silently — showing stale content or an empty directory.

**Solution**: Use `cp -r` instead of `ln -s` to populate `.claude/skills/claude-to-im/`. After patching source files, always re-copy:

```bash
cp -r .agents/skills/claude-to-im-skill/* .claude/skills/claude-to-im/
```

### Updated files modified table

| File | Changes |
|------|---------|
| `src/llm-provider.ts` | Bug 1, 7, 8: `node` prefix for `.js` paths + `R_OK` on Windows |
| `src/main.ts` | Bug 6: env injection workaround |
| `scripts/supervisor-windows.ps1` | Bug 2, 3 (global), 4, 5, 9: PS 5.1 compat + separate stderr log |
| `scripts/daemon.sh` | Bug 4: bash→PowerShell argument passing |

### Bug 10: cardkit.v2 not available — fallback to v1 static cards

**Symptom**: `Failed to create streaming card: Cannot read properties of undefined (reading 'card')` — repeated hundreds of times, messages stuck.

**Root cause**: The feishu adapter calls `this.restClient.cardkit.v2.card.create()`, but `@larksuiteoapi/node-sdk` v1.59–1.60 only ships `cardkit.v1`. `cardkit.v2` is `undefined`. This is **upstream Issue #15**.

**Fix**: Modify `feishu-adapter.ts` to auto-detect SDK version and fallback:

```typescript
// In _doCreateStreamingCard():
const hasV2 = !!this.restClient.cardkit?.v2?.card;
const hasV1 = !!this.restClient.cardkit?.v1?.card;
if (!hasV2 && !hasV1) return false;

// Use whichever is available:
const cardApi = hasV2 ? this.restClient.cardkit.v2.card : this.restClient.cardkit.v1.card;
```

v1 cards support `create` and `update` but NOT `streamContent` or `streamingMode`. Result: cards work but without real-time streaming — content updates once at completion.

**Also patched**: `flushCardUpdate()` (skip if no `streamContent`) and `finalizeCard()` (use v1 `card.update`, skip `streamingMode.set`).

### CTI_FEISHU_CARD_MODE — card/text mode switch

Added `CTI_FEISHU_CARD_MODE` env var to `config.env` for switching between card and plain text output:

| Value | Behavior |
|-------|----------|
| `text` | Plain text messages only (most reliable) |
| `v1` | Static cards via cardkit v1 (Thinking → final update) |
| _(unset)_ | Auto-detect (v2 > v1 > text) |

```bash
# In ~/.claude-to-im/config.env:
CTI_FEISHU_CARD_MODE=text    # plain text
# CTI_FEISHU_CARD_MODE=v1   # static cards
```

Requires bridge restart after change.

### SDK v2 status (as of 2026-04)

`@larksuiteoapi/node-sdk` npm latest is **1.60.0**. `cardkit.v2` (streaming cards) is NOT included. No timeline from Feishu SDK team. The core library was likely developed against an internal/beta SDK version.

### Note: Feishu rich text rendering vs cardkit cards

After setting `CTI_FEISHU_CARD_MODE=text`, you may still see card-like formatting in Feishu. This is **not** our cardkit card — it's Feishu's native rendering of `msg_type: 'post'` (rich text) messages. Long messages or messages with markdown formatting are automatically displayed by the Feishu client in a card-like container. This is client-side behavior and cannot be changed from the bot side.

| What you see | Source | Controllable? |
|---|---|---|
| Card with "Thinking..." → streaming updates → final content | Our cardkit card (v1/v2) | ✅ Yes — `CTI_FEISHU_CARD_MODE` |
| Card-like container with formatted text (long message) | Feishu client rich text rendering | ❌ No — Feishu native behavior |
