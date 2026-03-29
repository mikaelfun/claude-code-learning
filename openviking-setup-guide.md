# OpenViking + Claude Code 完整搭建指南

> **作者**: Kun Fang
> **日期**: 2026-03-29
> **环境**: Windows 11 + Git Bash (MSYS2) + Python 3.11 + Claude Code
> **OpenViking 版本**: 0.2.13

## 目录

- [概述](#概述)
- [架构](#架构)
- [安装步骤](#安装步骤)
  - [1. 安装 OpenViking](#1-安装-openviking)
  - [2. 配置 ov.conf](#2-配置-ovconf)
  - [3. 安装 Claude Code 插件](#3-安装-claude-code-插件)
  - [4. 自动启动脚本](#4-自动启动脚本)
  - [5. NewAPI 补丁](#5-newapi-补丁)
  - [6. Windows 便捷启动](#6-windows-便捷启动)
- [记忆导入](#记忆导入)
- [NewAPI 迁移](#newapi-迁移)
- [踩坑记录](#踩坑记录)
- [运维命令](#运维命令)
- [配置参考](#配置参考)

---

## 概述

[OpenViking](https://github.com/volcengine/OpenViking) 是一个 Agent-native 语义记忆数据库。配合 Claude Code 插件可以实现：

- **Auto-Recall (UserPromptSubmit hook)**: 每次用户输入时自动搜索相关记忆，注入 `<relevant-memories>` 到 context
- **Auto-Capture (Stop hook)**: 会话结束时自动提取对话中的记忆（偏好、事实、事件）存储到向量库
- **跨会话持久记忆**: 不依赖 Claude Code 的 session compact，信息不会丢失

## 架构

```
Claude Code Session
  │
  ├─ SessionStart hook ─→ ensure-server.sh ─→ 启动 openviking-server (port 1933)
  │
  ├─ UserPromptSubmit hook ─→ auto-recall.mjs
  │     └─ POST /api/v1/search/find ─→ 返回相关记忆 ─→ 注入 <relevant-memories>
  │
  └─ Stop hook ─→ auto-capture.mjs
        ├─ 读取会话 transcript
        ├─ 增量提取新 turns（跳过已 capture 的）
        ├─ 触发关键词/语义过滤
        └─ POST /api/v1/sessions → /messages → /extract ─→ VLM 提取记忆

openviking-server (127.0.0.1:1933)
  ├─ VectorDB: 本地 (hnswlib, 1536 维)
  ├─ Embedding: text-embedding-3-small via NewAPI
  ├─ VLM: gpt-4o via NewAPI (记忆提取)
  └─ Queue: SQLite (Embedding + Semantic + Semantic-Nodes)
```

---

## 安装步骤

### 1. 安装 OpenViking

```bash
# 使用 Python 3.11（不要用 3.12+，兼容性更好）
python -m pip install openviking --upgrade

# 验证
openviking --version
# openviking 0.2.13
```

> **注意**: Node.js 建议使用 v22 LTS。v24 曾导致 OpenViking 插件脚本报错。

### 2. 配置 ov.conf

创建 `~/.openviking/ov.conf`：

```json
{
  "server": {
    "host": "127.0.0.1",
    "port": 1933
  },
  "storage": {
    "workspace": "C:\\Users\\<username>\\.openviking\\data",
    "vectordb": { "backend": "local" },
    "agfs": { "backend": "local", "port": 1833 }
  },
  "embedding": {
    "dense": {
      "provider": "openai",
      "api_key": "<your-api-key>",
      "api_base": "https://kunnewapi.net/v1",
      "model": "text-embedding-3-small",
      "dimension": 1536
    }
  },
  "vlm": {
    "provider": "openai",
    "api_key": "<your-api-key>",
    "api_base": "https://kunnewapi.net/v1",
    "model": "gpt-4o"
  },
  "log": {
    "level": "WARNING",
    "output": "stdout"
  }
}
```

> **关键**: `provider` 必须是 `"openai"`（不是 `"azure"`），即使你用的是 Azure OpenAI 也走 OpenAI 兼容接口。见下方 [NewAPI 迁移](#newapi-迁移)。

### 3. 安装 Claude Code 插件

```bash
# 在 Claude Code CLI 中执行：
/plugin marketplace add Castor6/openviking-plugins
/plugin install claude-code-memory-plugin@openviking-plugin
```

安装后 `~/.claude/settings.json` 会自动添加：

```json
{
  "enabledPlugins": {
    "claude-code-memory-plugin@openviking-plugin": true
  },
  "extraKnownMarketplaces": {
    "openviking-plugin": {
      "source": { "source": "github", "repo": "Castor6/openviking-plugins" }
    }
  }
}
```

插件注册的 3 个 hooks：

| Hook | 脚本 | Timeout | 作用 |
|------|------|---------|------|
| `SessionStart` | `bootstrap-runtime.mjs` | 120s | 初始化运行时 |
| `UserPromptSubmit` | `auto-recall.mjs` | 8s | 每次输入搜索相关记忆 |
| `Stop` | `auto-capture.mjs` | 45s | 会话结束提取记忆 |

### 4. 自动启动脚本

创建 `~/.openviking/ensure-server.sh`：

```bash
#!/bin/bash
# Ensure OpenViking server is running before plugin hooks fire
# Called by Claude Code SessionStart hook (user-level)

OV_SERVER="/c/Users/<username>/AppData/Local/Programs/Python/Python311/Scripts/openviking-server.exe"
OV_CONF="$HOME/.openviking/ov.conf"
OV_PORT=1933

# Auto-patch embedder for NewAPI compatibility (idempotent)
bash "$HOME/.openviking/patch-embedder.sh" 2>/dev/null

# Quick check: is port already listening?
if netstat -an 2>/dev/null | grep -q ":${OV_PORT}.*LISTENING"; then
  exit 0
fi

# Start server in background (detached, no console window)
OPENVIKING_CONFIG_FILE="$OV_CONF" "$OV_SERVER" --config "$OV_CONF" >/dev/null 2>&1 &

# Wait up to 20s for server to be ready
for i in $(seq 1 40); do
  if curl -s --max-time 1 "http://127.0.0.1:${OV_PORT}/health" >/dev/null 2>&1; then
    exit 0
  fi
  sleep 0.5
done

echo "Warning: OpenViking server failed to start within 20s" >&2
exit 0  # Don't block Claude Code even if server fails
```

在 `~/.claude/settings.json` 添加 hook：

```json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "bash ~/.openviking/ensure-server.sh",
        "timeout": 25
      }]
    }]
  }
}
```

### 5. NewAPI 补丁

创建 `~/.openviking/patch-embedder.sh`：

```bash
#!/bin/bash
# Patch OpenViking openai_embedders.py to use array input format
# Required for NewAPI compatibility (only accepts ["text"] not "text")
# Re-run after each `pip install --upgrade openviking`

EMBEDDER="/c/Users/<username>/AppData/Local/Programs/Python/Python311/Lib/site-packages/openviking/models/embedder/openai_embedders.py"

if [ ! -f "$EMBEDDER" ]; then
  echo "[patch-embedder] File not found: $EMBEDDER" >&2
  exit 0
fi

# Check if already patched
if grep -q '"input": \[text\]' "$EMBEDDER" 2>/dev/null; then
  exit 0  # Already patched
fi

# Apply patch: "input": text -> "input": [text]
sed -i 's/"input": text, "model"/"input": [text], "model"/' "$EMBEDDER"

if grep -q '"input": \[text\]' "$EMBEDDER" 2>/dev/null; then
  echo "[patch-embedder] Patched successfully"
else
  echo "[patch-embedder] WARNING: patch failed" >&2
fi
```

> **为什么需要补丁？** OpenViking 发送 `"input": "text"` 给 embedding API，但 NewAPI (copilot-proxy) 只接受数组格式 `"input": ["text"]`。这是 NewAPI 的限制，不是 OpenViking 的 bug。

### 6. Windows 便捷启动

创建 `~/.openviking/start-server.bat`（双击即可手动启动）：

```bat
@echo off
set OPENVIKING_CONFIG_FILE=%USERPROFILE%\.openviking\ov.conf

netstat -an | findstr ":1933" >nul 2>&1
if %errorlevel%==0 (
    echo OpenViking server is already running on port 1933
    pause
    exit /b 0
)

echo Starting OpenViking server...
start /b "" "C:\Users\<username>\AppData\Local\Programs\Python\Python311\Scripts\openviking-server.exe" --config "%OPENVIKING_CONFIG_FILE%"

timeout /t 3 >nul
netstat -an | findstr ":1933" >nul 2>&1
if %errorlevel%==0 (
    echo OpenViking server started successfully on http://127.0.0.1:1933
) else (
    echo Failed to start server. Check logs.
)
pause
```

---

## 记忆导入

如果你有现成的知识库（比如 MEMORY.md），可以批量导入到 OpenViking：

```python
#!/usr/bin/env python3
"""Import memories into OpenViking - chunked by topic for better extraction."""
import json, urllib.request, time

BASE = "http://127.0.0.1:1933"

def api(method, path, body=None):
    data = json.dumps(body).encode() if body else None
    req = urllib.request.Request(f"{BASE}{path}", data=data, method=method,
                                 headers={"Content-Type": "application/json"} if data else {})
    with urllib.request.urlopen(req, timeout=180) as resp:
        return json.loads(resp.read())

def import_session(topic, chunks):
    """Create a session with focused chunks and commit."""
    print(f"\n--- Importing: {topic} ({len(chunks)} chunks) ---")
    session = api("POST", "/api/v1/sessions", {
        "agent_id": "claude-code",
        "metadata": {"source": "memory-import", "topic": topic}
    })
    sid = session["result"]["session_id"]

    for i, (label, content) in enumerate(chunks):
        api("POST", f"/api/v1/sessions/{sid}/messages", {"role": "user", "content": content})
        api("POST", f"/api/v1/sessions/{sid}/messages", {
            "role": "assistant",
            "content": f"Noted. I've recorded this information about {label}."
        })
        print(f"  [{i+1}/{len(chunks)}] {label}")

    print("  Committing (VLM extraction)...")
    result = api("POST", f"/api/v1/sessions/{sid}/commit", {"wait": True, "timeout": 180})
    print(f"  Status: {result.get('status', 'unknown')}")

# Example usage:
import_session("User Preferences", [
    ("Timezone", "Remember: user is in Asia/Singapore (GMT+8)"),
    ("Language", "User prefers Chinese for UI/docs"),
])
```

**最佳实践**：
- 按主题分组，每个 session 3-5 个相关 chunks
- 用 `commit(wait=True, timeout=180)` 等待 VLM 提取完成
- 导入间隔 `time.sleep(2)` 避免 rate limit

---

## NewAPI 迁移

### 背景

最初使用 Azure OpenAI 作为 embedding + VLM 后端。遇到严重问题：

1. **Azure OpenAI rate limit**: 420 条消息堆积在 Semantic queue，每条触发 7 次 VLM 调用 × 24 并发 → 速率限制级联超时
2. **Stop hook 挂死**: auto-capture.mjs 中的 extract API 45 秒超时
3. **Azure 收费**: 每个 token 都要钱

### 迁移步骤

将 `ov.conf` 中的 Azure 配置改为 NewAPI：

| 字段 | Azure (迁移前) | NewAPI (迁移后) |
|------|---------------|----------------|
| `embedding.provider` | `"azure"` | `"openai"` |
| `embedding.api_base` | `"https://xxx.openai.azure.com"` | `"https://kunnewapi.net/v1"` |
| `vlm.provider` | `"azure"` | `"openai"` |
| `vlm.api_base` | `"https://xxx.openai.azure.com"` | `"https://kunnewapi.net/v1"` |
| `vlm.model` | `"gpt-4o-mini"` | `"gpt-4o"` (免费，升级) |

> **重要**: 迁移后必须运行 `patch-embedder.sh`，因为 NewAPI 不接受裸字符串 embedding 输入。

### 效果

| 指标 | Azure | NewAPI |
|------|-------|--------|
| Extract 延迟 | 45s+ 超时 | ~19s |
| 费用 | 按 token 计费 | 免费 |
| Rate limit | 严格（TPM/RPM 限制） | 无限制 |
| VLM 模型 | gpt-4o-mini | gpt-4o |

---

## 踩坑记录

### 1. Azure OpenAI API base URL 写错

**错误**: `api_base: "https://xxx.cognitiveservices.azure.com"`
**正确**: `api_base: "https://xxx.openai.azure.com"`

Azure 认知服务有两个域名，OpenAI 端点必须用 `.openai.azure.com`。

### 2. SessionStart hook timeout 太短

**现象**: ensure-server.sh 首次启动 server 需要 5-8 秒，原始 timeout 15 秒不够
**修复**: timeout 改为 25 秒，内部等待循环改为 20 秒（40 次 × 0.5s）

### 3. 420 条消息堆积导致 Stop hook 挂死

**现象**: 每次退出 Claude Code 等待 40+ 秒
**根因**: Azure rate limit → queue.db 中 Semantic 队列积压 420 条 → VLM extract 超时
**修复**:

```bash
# 清空堆积的队列
rm ~/.openviking/data/_system/queue.db
# 重启 server
taskkill /F /IM openviking-server.exe
# 然后迁移到 NewAPI 避免再次堆积
```

### 4. NewAPI 不接受裸字符串 embedding 输入

**现象**: embedding API 返回 HTTP 400
**根因**: OpenViking 发送 `{"input": "text"}` 但 NewAPI 只接受 `{"input": ["text"]}`
**修复**: `patch-embedder.sh` 自动修补，集成到 `ensure-server.sh` 每次启动执行
**注意**: `pip install --upgrade openviking` 会覆盖补丁，但 ensure-server.sh 会自动重新打补丁

### 5. Windows PID lock 文件导致 server 无法启动 (process_lock.py)

**现象**: server 异常退出后无法重新启动，报错 `SystemError: <built-in function kill> returned a result with an exception set`
**根因**: CPython 在 Windows 上 `os.kill(pid, 0)` 对某些无效 PID 抛 `SystemError` 而不是 `OSError`。OpenViking 的 `process_lock.py` 只 catch 了 `OSError`，`SystemError` 未被捕获 → 启动崩溃

**修复**: 编辑 `site-packages/openviking/utils/process_lock.py`，第 46 行：

```python
# Before:
except OSError:
# After:
except (OSError, SystemError):
```

**临时 workaround**:

```bash
rm ~/.openviking/data/.openviking.pid
```

**注意**: `pip install --upgrade openviking` 会覆盖此修复。建议给 OpenViking 项目提 PR。

### 6. Node.js v24 兼容性问题

**现象**: 插件脚本执行报错
**修复**: 回退到 Node.js v22 LTS

### 7. settings.json 中 model 名称被 ANSI 转义码污染

**现象**: `"model": "opus[1m]"` — `[1m]` 是 ANSI bold 转义码
**修复**: 手动改回 `"model": "opus"`

### 8. `openviking add-memory` 命令提取 0 条

**现象**: CLI `add-memory` 不提取任何记忆
**修复**: 必须走完整流程 `session new → add-message → commit` 才能触发 VLM 提取

---

## 运维命令

```bash
# === Server 管理 ===
# 启动
bash ~/.openviking/ensure-server.sh

# 健康检查
curl -s http://127.0.0.1:1933/health | python -m json.tool

# 完整状态（队列、向量库、锁）
curl -s http://127.0.0.1:1933/api/v1/observer/system | python -m json.tool

# 停止
taskkill /F /IM openviking-server.exe

# === 故障排查 ===
# 清理 stale PID lock（server 无法启动时）
rm ~/.openviking/data/.openviking.pid

# 清理堆积队列（Stop hook 挂死时）
rm ~/.openviking/data/_system/queue.db

# 查看 debug 日志
tail -100 ~/.openviking/server-debug.log

# 查看 plugin hook 日志
cat ~/.openviking/logs/cc-hooks.log

# === 记忆管理 ===
# 搜索记忆
curl -s -X POST http://127.0.0.1:1933/api/v1/search/find \
  -H "Content-Type: application/json" \
  -d '{"query": "your search query", "target_uri": "viking://user/memories", "limit": 5}'

# 使用 CLI
openviking search "your query"
openviking status
openviking health

# === 升级后修复 ===
# pip upgrade 后重新打 embedder 补丁
bash ~/.openviking/patch-embedder.sh

# pip upgrade 后重新修复 process_lock.py
# 编辑 site-packages/openviking/utils/process_lock.py
# 将 "except OSError:" 改为 "except (OSError, SystemError):"
```

---

## 配置参考

### 最终文件布局

```
~/.openviking/
├── ov.conf                    # 主配置（server + embedding + VLM）
├── ensure-server.sh           # SessionStart hook — 自动启动
├── patch-embedder.sh          # NewAPI embedding 补丁
├── start-server.bat           # Windows 双击启动
├── import-memories.py         # 简单记忆导入
├── import-memories-v2.py      # 按主题分块导入
├── server-debug.log           # Server debug 日志
├── logs/
│   └── cc-hooks.log           # Plugin hook 日志
└── data/
    ├── .openviking.pid        # PID lock（自动管理）
    ├── _system/
    │   └── queue.db           # 任务队列（SQLite）
    ├── vectordb/              # 向量索引
    └── viking/                # 记忆文件（markdown）

~/.claude/settings.json        # SessionStart hook 注册
~/.claude/plugins/cache/openviking-plugin/  # 插件代码
```

### Plugin Hook 超时配置

`ov.conf` 中可选添加 `claude_code` 节来调整插件行为：

```json
{
  "claude_code": {
    "autoRecall": true,
    "autoCapture": true,
    "captureMode": "semantic",
    "recallLimit": 6,
    "scoreThreshold": 0.01,
    "timeoutMs": 15000,
    "captureTimeoutMs": 30000,
    "captureAssistantTurns": false,
    "debug": false
  }
}
```

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `autoRecall` | `true` | 是否自动搜索记忆 |
| `autoCapture` | `true` | 是否自动提取记忆 |
| `captureMode` | `"semantic"` | `"semantic"` 全量捕获 / `"keyword"` 仅关键词触发 |
| `recallLimit` | `6` | 每次召回最多几条记忆 |
| `timeoutMs` | `15000` | Recall 超时 (ms) |
| `captureTimeoutMs` | `30000` | Capture 超时 (ms) |
| `captureAssistantTurns` | `false` | 是否捕获 assistant turns（通常只需 user turns） |
