# NewAPI + Claude Code: Enable 1M Context Window

When using Claude Code with a third-party API proxy (NewAPI/OneAPI), the effective context window may show 200k instead of 1M even when using a 1M-capable model like `claude-opus-4-6`. This guide explains why and how to fix it.

## The Problem

```
❯ /context
  30.5k/200k tokens (15%)    ← should be 30.5k/1M
```

You're using `claude-opus-4-6` (which supports 1M context), but Claude Code only gives you 200k.

## Root Cause

Claude Code determines the effective context window size through two detection paths in its compiled source (`cli.js`):

### Path 1: Model name contains `[1m]` tag

```javascript
// Regex check on model name
function cE(modelName) {
  return /\[1m\]/i.test(modelName);  // looks for literal "[1m]"
}

// If matched → 1,000,000 tokens
if (cE(modelName)) return 1_000_000;
```

### Path 2: Model name matches known patterns + API returns beta feature flag

```javascript
// Check if model name contains "opus-4-6" or "claude-sonnet-4"
function FX1(modelName) {
  return modelName.includes("opus-4-6") || modelName.includes("claude-sonnet-4");
}

// Also requires the API response to include this beta header:
// "context-1m-2025-08-07"
```

### Why it fails with NewAPI

- **Path 1 fails**: Your model name is `claude-opus-4-6` (no `[1m]` tag)
- **Path 2 fails**: NewAPI proxy doesn't pass through the `context-1m-2025-08-07` beta feature flag from Anthropic's API response

Result: Claude Code falls back to the default `200000` token limit.

### The 200k default

```javascript
var sT1 = 200000;  // default context window cap

function lf(modelName, features) {
  if (cE(modelName)) return 1_000_000;        // Path 1
  let info = getModelInfo(modelName);
  if (info?.max_input_tokens > sT1 && isDisabled1M())
    return sT1;                                 // capped at 200k
  if (features?.includes("context-1m-...") && FX1(modelName))
    return 1_000_000;                           // Path 2
  return sT1;                                   // default: 200k
}
```

## The Fix

Add `[1m]` to the model name on the Claude Code side, and add a model mapping in NewAPI to strip it.

### Step 1: Claude Code — Add `[1m]` to model name

Edit `~/.claude/settings.json`:

```json
{
  "env": {
    "ANTHROPIC_MODEL": "claude-opus-4-6[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6[1m]"
  }
}
```

Or via CLI:

```bash
claude config set --global model 'claude-opus-4-6[1m]'
```

### Step 2: NewAPI — Add model mapping

In NewAPI admin panel → Channels → Edit your Anthropic channel → **Model Redirect** section:

| Key (incoming model name) | Value (backend model name) |
|---------------------------|---------------------------|
| `claude-opus-4-6[1m]` | `claude-opus-4.6-1m` |

This maps the `[1m]`-tagged name back to the actual backend model name.

> **Note**: The exact backend model name depends on your upstream provider. Common formats:
> - `claude-opus-4-6-1m` (hyphen-separated)
> - `claude-opus-4.6-1m` (dot-separated, GitHub Copilot format)
> - `claude-opus-4-6-20250827` (date-stamped, official Anthropic format)

### Step 3: Verify

Restart Claude Code and check:

```
❯ /context
  30.7k/1m tokens (3%)    ← 1M context enabled!
```

## How It Works

```
Claude Code (local)                    NewAPI (proxy)                  Backend
─────────────────                      ──────────────                  ───────
model = "claude-opus-4-6[1m]"
        │
        ├─ cE() regex → /\[1m\]/ ✓
        │  → effective window = 1M
        │
        └─ API request ──────────────→ model redirect:
           model: "claude-opus-4-6[1m]"  "claude-opus-4-6[1m]"
                                          → "claude-opus-4.6-1m"
                                                    │
                                                    └──────→ normal API call
                                                              model: "claude-opus-4.6-1m"
```

## Also Useful: Disable Blocking Compact Prompt

With 1M context, Claude Code's blocking compact prompt (`MANUAL_COMPACT_BUFFER_TOKENS = 3000`) fires at ~997k tokens. Auto-compact triggers much earlier (~13k buffer), so the blocking prompt is rarely needed. To disable it:

```bash
# patch-autocompact.sh — patches cli.js to set MANUAL_COMPACT_BUFFER_TOKENS = 0
# Re-run after every `npm update -g @anthropic-ai/claude-code`

NPM_ROOT="$(npm root -g | tr '\\' '/')"
CLI_JS="${NPM_ROOT}/@anthropic-ai/claude-code/cli.js"

# Pattern: VARNAME=3000,kDK=3 (stable across minified versions)
PATTERN=$(grep -oP '\w+=3000,kDK=3' "$CLI_JS" | head -1)
VAR_NAME="${PATTERN%%=*}"
sed -i "s/${VAR_NAME}=3000/${VAR_NAME}=0/g" "$CLI_JS"
```

This makes the blocking limit equal to the full context window, so only auto-compact handles compaction silently.

## Environment

- Claude Code 1.0.x+
- NewAPI (calciumion/new-api) v0.11.x
- Windows 11 + Git Bash
- Tested with `claude-opus-4-6` and `claude-sonnet-4-6` models

## Key Takeaway

The `[1m]` tag is a **local-only signal** — Claude Code checks it via regex before sending any API request. The API proxy never sees `[1m]` as a real model identifier; it gets mapped away by NewAPI's model redirect. This is a clean, non-destructive approach that survives Claude Code updates.
