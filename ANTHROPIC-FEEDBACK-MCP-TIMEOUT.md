# Feedback: Claude Desktop MCP Tool Timeout Needs to be User-Configurable

## Summary

Claude Desktop hardcodes a 60-second timeout for MCP tool calls via `@modelcontextprotocol/sdk`'s `DEFAULT_REQUEST_TIMEOUT_MSEC = 60000`. This makes it impossible to use any MCP server that takes longer than 60 seconds to respond — including OpenAI's official Codex MCP server, which routinely takes 2-5 minutes for research tasks involving web searches.

## Current State

| Component | Version | Timeout | Configurable? |
|-----------|---------|---------|---------------|
| Claude Desktop (macOS) | Latest (Mar 2026) | **60s** | **No** |
| `@modelcontextprotocol/sdk` | **1.26.0** (bundled) | 60s default | Yes — callers can pass `timeout` option |
| `@modelcontextprotocol/sdk` | **1.28.0** (latest) | 60s default | Yes — same, plus `resetTimeoutOnProgress` |
| Claude Code CLI | 2.1.85 | **~27 hours** (1e8 ms) | Yes — `MCP_TOOL_TIMEOUT` env var |

## The Problem in Detail

1. **Claude Desktop bundles SDK v1.26.0**, which has `DEFAULT_REQUEST_TIMEOUT_MSEC = 60000`
2. Claude Desktop does NOT pass a custom `timeout` option when calling MCP tools — it uses the SDK default
3. The `env` field in `~/.claude/settings.json` sets `MCP_TOOL_TIMEOUT` but this only affects **Claude Code CLI**, not Claude Desktop
4. There is **no `claude_desktop_config.json` field** to override this timeout
5. The SDK (even latest 1.28.0) still defaults to 60s — this was never changed upstream

## Evidence

### Error observed:
```
Error: MCP error -32001: Request timed out
```

### Claude Desktop's bundled SDK (extracted from app.asar):
```
@modelcontextprotocol/sdk: 1.26.0
```

### SDK source (identical in v1.26.0 and v1.28.0):
```javascript
// @modelcontextprotocol/sdk/dist/esm/shared/protocol.js
export const DEFAULT_REQUEST_TIMEOUT_MSEC = 60000;
const timeout = options?.timeout ?? DEFAULT_REQUEST_TIMEOUT_MSEC;
```

### Claude Code CLI source (works fine — uses 1e8 = ~27 hour timeout):
```javascript
// cli.js (minified)
parseInt(process.env.MCP_TOOL_TIMEOUT||"",10)||1e8  // 100,000,000ms
```

### Why `codex exec` works but Codex MCP doesn't:
- `codex exec` runs as a direct subprocess with no MCP timeout
- `codex mcp-server` responds via JSON-RPC, but Claude Desktop kills the request at 60s
- The Codex MCP server itself has no internal timeout — it's the **client** (Claude Desktop) that times out

### Log evidence (`~/Library/Logs/Claude/mcp-server-codex.log`):
The log shows the tool call request going out but no response logged — Claude Desktop cancels before the response arrives.

## Proposed Solutions (in order of preference)

### 1. Add per-server timeout config to `claude_desktop_config.json`
```json
{
  "mcpServers": {
    "codex": {
      "command": "codex",
      "args": ["mcp-server"],
      "timeout": 300000
    }
  }
}
```

### 2. Respect `MCP_TOOL_TIMEOUT` env var (parity with Claude Code CLI)
The env var already exists and works in Claude Code CLI. Claude Desktop should read it too.

### 3. Pass `resetTimeoutOnProgress: true` when calling tools
The SDK (v1.28.0) already supports this. If the MCP server sends progress notifications, the timeout resets — allowing long-running tools to complete as long as they're making progress. This is already available in the SDK but Claude Desktop doesn't use it.

### 4. Bump SDK to 1.28.0 and use `maxTotalTimeout`
SDK v1.28.0 added `maxTotalTimeout` support alongside `resetTimeoutOnProgress`. This allows setting a generous per-progress timeout (e.g., 120s) with a hard ceiling (e.g., 30 minutes).

## Impact

This affects any MCP server that performs non-trivial work:
- **OpenAI Codex MCP** — research tasks with web searches take 2-5 minutes
- **Database query servers** — complex queries can exceed 60s
- **Code analysis servers** — large codebase analysis takes minutes
- **Any AI-backed MCP server** — LLM inference chains routinely exceed 60s

## Related Issues

- MCP TypeScript SDK #245: Tool call timeout hardcoded at 60s (17+ reactions)
- MCP TypeScript SDK PR #849: Proposed `resetTimeoutOnProgress=true` default — rejected, but feature exists as opt-in
- MCP Spec SEP-1539: Timeout Coordination proposal
- Claude Code #22542: MCP tool timeout issue (labeled `external`)

## Workaround (Current)

The only workaround is to use `codex exec` via Bash in Claude Code CLI instead of the MCP server. This loses statefulness (no `threadId` for multi-turn conversations) and requires Claude Code CLI rather than Desktop.
