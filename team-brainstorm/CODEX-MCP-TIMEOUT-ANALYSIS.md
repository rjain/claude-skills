# Codex MCP Server Timeout Analysis

**Date:** 2026-03-27
**Codex version:** 0.117.0 (codex_cli_rs)
**Claude Code version:** 2.1.78 (embedded in Claude Desktop)
**Claude Desktop version:** Claude.app (Electron)
**Platform:** macOS 15.7.3, arm64

## Summary

Codex MCP tool calls time out after 60 seconds when invoked from Claude Desktop, but `codex exec` CLI calls complete successfully even when taking 2-3 minutes. The root cause is a **hardcoded 60-second MCP tool timeout in Claude Desktop's Electron app** that cannot be overridden by user configuration.

## Symptom

```
Error: MCP error -32001: Request timed out
```

Simple queries (e.g., "What is the capital of Sri Lanka?") succeed via MCP because they complete in <60s. Research-heavy queries that trigger web searches and tool use (taking 2-3 minutes) always fail.

## Root Cause

Claude Desktop (`Claude.app`) has a hardcoded default MCP tool timeout of **60 seconds** (`6e4` ms), defined as `DEFAULT_MCP_TOOL_TIMEOUT_MS` in the bundled `app.asar`:

```javascript
// In app.asar (minified)
const lIt = 6e4;  // 60,000 ms = 60 seconds
// ...
yke = lIt;  // yke = DEFAULT_MCP_TOOL_TIMEOUT_MS

// Per-server timeout lookup:
function lTt(serverName) {
    const config = getServerConfig();
    return config?.mcpToolTimeoutOverridesMs?.[serverName]  // per-server override
        ?? config?.mcpToolTimeoutMs                          // global override
        ?? yke;                                              // 60s default
}
```

The override values (`mcpToolTimeoutMs`, `mcpToolTimeoutOverridesMs`) are populated from a **server-side GrowthBook feature flag system**, not from any local user config file. There is no user-facing way to change this value.

## Architecture: Two Different MCP Clients

When running Claude Code inside Claude Desktop, there are **two separate MCP client implementations** with different timeout behaviors:

### Claude Desktop (Electron App)
- **Config file:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **MCP tool timeout:** 60 seconds (hardcoded, overridable only via server-side feature flags)
- **Spawns MCP servers** as child processes of `Claude.app` (via `disclaimer` wrapper)
- **Does NOT read** `~/.claude/settings.json` for MCP configuration
- **Does NOT support** `env` field in `claude_desktop_config.json` MCP server entries

### Claude Code (CLI)
- **Config file:** `~/.claude/settings.json`
- **MCP tool timeout:** `parseInt(process.env.MCP_TOOL_TIMEOUT) || 1e8` (~27 hours default)
- **Spawns its own MCP servers** from `settings.json` `mcpServers` entries
- **Supports** `env` field per MCP server for passing environment variables

### The Proxy Problem

When Claude Code runs embedded inside Claude Desktop:

```
Claude Desktop (Electron)
  └─ spawns codex mcp-server (PID 7935) ← Desktop manages this
  └─ spawns claude code CLI (PID 7954)
       └─ gets codex MCP tools via SDK bridge ("type":"sdk")
       └─ does NOT spawn its own codex mcp-server
```

Claude Code receives Codex MCP tools as `"type":"sdk"` servers proxied from Claude Desktop. When Claude Code calls `mcp__codex__codex`, the request flows through Claude Desktop's MCP client, which enforces the 60-second timeout. The `MCP_TOOL_TIMEOUT` env var in `settings.json` is never consulted.

Confirmed by process tree:
```
PID 7129 /Applications/Claude.app/Contents/MacOS/Claude
  ├─ PID 7701 disclaimer → node codex mcp-server → codex binary  (Desktop's instance)
  ├─ PID 7935 disclaimer → node codex mcp-server → codex binary  (Code's instance)
  └─ PID 7954 claude code CLI (--mcp-config with "codex":{"type":"sdk"})
```

## Why `codex exec` Works

`codex exec` is invoked via the Bash tool as a direct subprocess:

```bash
codex exec --config 'approval_policy="never"' "prompt here"
```

This bypasses the MCP transport entirely:
- No JSON-RPC protocol layer
- No MCP client timeout
- Bash tool timeout is configurable (up to 600,000ms / 10 minutes)
- The subprocess runs until completion or Bash timeout

## Configuration Audit

### Settings that DO NOT fix the timeout

| Setting | Location | Why it doesn't help |
|---------|----------|-------------------|
| `MCP_TOOL_TIMEOUT=1800000` | `~/.claude/settings.json` → `env` | Only read by Claude Code CLI, not Desktop |
| `MCP_TIMEOUT=30000` | `~/.claude/settings.json` → `env` | Controls MCP server **startup** timeout, not tool calls |
| `tool_timeout_sec` | `~/.codex/config.toml` | Controls how long Codex waits for **its own** MCP tool calls to other servers |
| `env.RUST_LOG=debug` | `~/.claude/settings.json` → `mcpServers.codex.env` | Only applied when Claude Code spawns the server; Desktop ignores it |

### Settings that COULD fix it (but aren't user-accessible)

| Setting | Location | Notes |
|---------|----------|-------|
| `mcpToolTimeoutMs` | Server-side (GrowthBook feature flags) | Global MCP timeout override for Desktop |
| `mcpToolTimeoutOverridesMs` | Server-side (GrowthBook feature flags) | Per-server timeout override (keyed by server name) |

## Workarounds

### 1. Use `codex exec` instead of MCP (recommended)

In the team-brainstorm skill, fall back to `codex exec` for Codex calls:

```bash
codex exec \
  --config 'approval_policy="never"' \
  --config 'sandbox_permissions=["disk-full-read-access"]' \
  "Your prompt here" 2>&1
```

**Trade-off:** Loses cross-round statefulness (no `threadId`). Each round must inline prior context in the prompt.

### 2. Run Claude Code from the terminal (not Desktop)

When running `claude` directly from the terminal (not embedded in Desktop), Claude Code manages its own MCP servers and uses `MCP_TOOL_TIMEOUT` from `settings.json`:

```bash
# This will use the 30-minute timeout from settings.json
claude
```

### 3. Keep prompts short to stay under 60s

For MCP usage, keep Codex prompts simple enough to complete within 60 seconds. The "capital of Sri Lanka" test succeeded because it didn't trigger web searches.

## Log Locations

| Log | Path | Contents |
|-----|------|----------|
| MCP server log (per-server) | `~/Library/Logs/Claude/mcp-server-codex.log` | JSON-RPC messages, server init/shutdown |
| General MCP log | `~/Library/Logs/Claude/mcp.log` | All MCP servers, errors, stderr capture |
| Claude Desktop main log | `~/Library/Logs/Claude/main.log` | App lifecycle, MCP server connect/disconnect |
| MCP info | `~/Library/Logs/Claude/mcp-info.json` | Server status snapshot |

To enable Codex Rust debug logging (when running from CLI, not Desktop):
```toml
# ~/.claude/settings.json mcpServers.codex.env
{ "RUST_LOG": "debug" }
```

Stderr from the Codex binary appears in `mcp-server-codex.log` and `mcp.log`.

## Codex MCP vs CLI Comparison

| Aspect | `codex mcp-server` (MCP) | `codex exec` (CLI) |
|--------|--------------------------|---------------------|
| Protocol | JSON-RPC over stdio | Direct subprocess |
| Timeout | 60s (Desktop) / ~27h (Code CLI) | Bash tool timeout (up to 10min) |
| Statefulness | `threadId` across rounds | Stateless (must inline context) |
| Web search | Works (when `web_search = "live"`) | Works (same config) |
| Tokens (typical) | Same | Same |
| Config errors | `web_search = "live"` causes `codex exec` to fail with config parse error | MCP server handles it fine |

### Config incompatibility note

The `~/.codex/config.toml` setting `web_search = "live"` (a string) is valid for the MCP server and interactive mode, but causes `codex exec` to fail:

```
Error loading config.toml: invalid type: string "live", expected a boolean in `features`
```

Workaround for `codex exec`: use a temp config or override:
```bash
CODEX_HOME=$(mktemp -d)
cat > "$CODEX_HOME/config.toml" << 'EOF'
model = "gpt-5.4"
[features]
web_search = true
EOF
CODEX_HOME="$CODEX_HOME" codex exec "prompt" 2>&1
```

## Recommendations

1. **File a feature request** with Anthropic to expose `mcpToolTimeoutMs` in `claude_desktop_config.json`
2. **File an issue** with OpenAI Codex for the `web_search = "live"` config incompatibility between MCP and exec modes
3. **Default to `codex exec`** in automation skills that may trigger long-running Codex research
4. **Use MCP only for quick queries** (<60s) where `threadId` statefulness is valuable
