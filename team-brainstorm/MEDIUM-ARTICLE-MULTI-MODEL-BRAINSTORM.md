# Building a Multi-Model Debate System: Claude, Codex, and Gemini Brainstorming Together

*How I built a skill that orchestrates structured debates across three AI models — and the hard timeout bug that almost killed it.*

---

When you ask one AI model a question, you get one perspective. When you ask three models the same question and have them debate each other across multiple rounds, you get something closer to consensus — or at least a clearer map of where they disagree.

I built a "team brainstorm" skill for Claude Code that runs a structured multi-round debate between Claude (Anthropic), Codex (OpenAI), and Gemini (Google). Each model researches independently, critiques the others' findings, and defends or concedes positions. Claude orchestrates the loop and synthesizes a final consensus table.

The goal was simple: ask all three models the same question, let them argue, and surface what they agree on. The implementation turned into a deep dive into CLI tools, MCP servers, and a 60-second timeout buried inside an Electron app.

---

## The Architecture

The skill runs in three phases across up to three rounds:

**Round 1 — Independent Research.** All three models investigate the topic in parallel. No model sees the others' work. This produces three independent perspectives.

**Round 2 — Cross-Critique.** Each model receives the other two models' findings. They agree, challenge with evidence, or revise their positions. This is where the real value emerges — models catch each other's errors and strengthen weak arguments.

**Round 3 — Final Rebuttal (if needed).** Only runs if disputes remain. Models make their final case. No hedging allowed.

After the rounds complete, Claude builds a consensus table showing each finding, each model's position, and how it was resolved (unanimous, majority, or flagged for user review).

---

## Integrating Gemini: The Straightforward Path

Gemini was the easiest model to integrate. Google ships a CLI tool (`gemini`) that supports a non-interactive headless mode, making it ideal for automation.

**Round 1** uses `-o json` to capture the `session_id` from the response:

```bash
gemini --approval-mode yolo -p "Your research prompt here" -o json 2>/dev/null
```

The JSON output includes a `session_id` field. Subsequent rounds use `--resume` with that ID, giving Gemini full conversation memory across the debate:

```bash
gemini --resume "29bb0338-cb2a-4194-a350-94218b010cfe" \
  --approval-mode yolo \
  -p "Round 2: Here are the other agents' positions..." \
  -o json 2>/dev/null
```

**Key pitfall:** The `--approval-mode yolo` flag is NOT inherited on `--resume`. You must pass it explicitly on every call, or Gemini hangs waiting for interactive approval — silently, with no error message. This cost me a debugging session before I figured out why Round 2 calls were stalling.

Gemini's model (`gemini-3-flash-preview`) performed well in our tests. It used Google web search effectively (2 successful searches in Round 1 of our restaurant debate), produced opinionated recommendations with specific addresses and dishes, and responded to critiques thoughtfully in Round 2 — successfully challenging another model's factual error about a restaurant's menu.

---

## Integrating Codex: Two Paths, Neither Perfect

Codex presented a harder integration challenge. OpenAI offers two ways to use it programmatically: the MCP server (`codex mcp-server`) and the CLI (`codex exec`). Each has trade-offs.

### Option 1: Codex MCP Server

The MCP (Model Context Protocol) server is the preferred integration. Claude Code can call Codex tools directly through the MCP protocol, and Codex returns a `threadId` that enables full statefulness across rounds — just like Gemini's `--resume`.

```json
{
  "prompt": "Research the best vegetarian restaurant near Santa Clara",
  "approval-policy": "never",
  "sandbox": "read-only"
}
```

When it works, it's elegant. Simple queries complete in seconds and return structured JSON with a `threadId` for follow-up calls via `codex-reply`.

**The problem:** Any query that triggers serious research — web searches, shell commands, menu scraping — takes longer than 60 seconds. And that's where the timeout kills it.

### Option 2: Codex CLI (`codex exec`)

The fallback is running Codex as a direct subprocess:

```bash
codex exec \
  --config 'approval_policy="never"' \
  --config 'sandbox_permissions=["disk-full-read-access"]' \
  "Your research prompt here" 2>&1
```

This bypasses the MCP protocol entirely. There's no 60-second limit — the Bash tool timeout goes up to 10 minutes, and background tasks can run even longer. In our tests, Codex research queries used 13,000-114,000 tokens, ran 0-22 web searches, executed 0-13 shell commands (including Python scripts, Ruby parsers, and curl pipelines), and completed in 30 seconds to 3 minutes.

**The trade-off:** No `threadId`. Each round is stateless. You have to summarize the prior round's findings and inject them into the next prompt. It works, but the model loses the nuance of its own prior reasoning.

**Additional pitfall:** The `~/.codex/config.toml` setting `web_search = "live"` (a string value) works fine for the MCP server and interactive mode, but causes `codex exec` to crash:

```
Error loading config.toml: invalid type: string "live", expected a boolean in `features`
```

The workaround is creating a temporary config directory with `web_search = true` (boolean) and pointing `CODEX_HOME` at it. This config format inconsistency between modes is a bug that should be filed with OpenAI.

---

## The 60-Second Wall: A Deep Dive

When testing the MCP integration, simple queries succeeded but research queries consistently failed with:

```
Error: MCP error -32001: Request timed out
```

The investigation took several hours and produced a clear root cause.

### Where the timeout lives

I was running Claude Code embedded inside Claude Desktop (the Electron app on macOS). This creates a specific process architecture:

```
Claude Desktop (Electron, PID 7129)
  ├── codex mcp-server (Desktop-managed, PID 7935)
  ├── codex mcp-server (Code-managed, PID 7712)
  └── claude code CLI (PID 7954)
        └── receives codex tools via SDK bridge ("type":"sdk")
```

Claude Desktop spawns the Codex MCP server and proxies the tools to the embedded Claude Code process. When Claude Code calls `mcp__codex__codex`, the request flows through Desktop's MCP client.

### The configuration maze

I had set `MCP_TOOL_TIMEOUT=1800000` (30 minutes) in `~/.claude/settings.json`:

```json
{
  "env": {
    "MCP_TOOL_TIMEOUT": "1800000"
  }
}
```

This setting is read by Claude Code's CLI — confirmed by searching the minified source:

```javascript
// In cli.js
function getTimeout() {
  return parseInt(process.env.MCP_TOOL_TIMEOUT || "", 10) || 1e8; // default ~27 hours
}
```

But Claude Desktop doesn't read `settings.json` for MCP configuration. It reads `~/Library/Application Support/Claude/claude_desktop_config.json`, which has no timeout field.

### Finding the hardcoded value

By extracting strings from `app.asar` (Claude Desktop's bundled JavaScript), I found:

```javascript
const DEFAULT_MCP_TOOL_TIMEOUT_MS = 6e4; // 60,000 ms = 60 seconds
```

The timeout can be overridden by `mcpToolTimeoutMs` or per-server via `mcpToolTimeoutOverridesMs` — but these values come from a server-side feature flag system (GrowthBook), not from any local configuration file.

There is currently no user-facing way to change this timeout in Claude Desktop.

### The two-client problem

This is the core architectural issue. Two different MCP clients exist with completely different timeout behaviors:

| Client | Default Timeout | User Configurable? |
|--------|----------------|-------------------|
| Claude Code CLI | ~27 hours (`1e8` ms) | Yes, via `MCP_TOOL_TIMEOUT` env var |
| Claude Desktop | 60 seconds (`6e4` ms) | No (server-side feature flags only) |

When Claude Code runs inside Desktop, it inherits Desktop's 60-second limit because the MCP server is managed by Desktop's process, not Code's.

---

## The Final Design

Given these constraints, the team-brainstorm skill uses a hybrid approach:

**For Gemini:** Always use the CLI with `--resume` for statefulness. It's reliable, fast, and the `--approval-mode yolo` flag (passed on every call) prevents interactive hangs.

**For Codex:** Prefer the MCP server (`codex` and `codex-reply` tools) when available, but expect timeouts on research-heavy queries. Fall back to `codex exec` via Bash when MCP fails or when running inside Claude Desktop. The skill detects which environment it's in and routes accordingly.

**For Claude:** Direct tool access (WebSearch, WebFetch, codebase tools). No protocol overhead. Claude also serves as the orchestrator — launching the other two models in parallel, collecting results, running the cross-critique rounds, and building the final consensus.

### Parallel execution

All three models research independently in Round 1. Codex and Gemini run as background tasks while Claude does its own research:

```
T=0s   Launch Codex (background) + Gemini (background)
T=0s   Claude starts own research (WebSearch, etc.)
T=30s  Claude research complete
T=45s  Gemini Round 1 complete (session_id saved)
T=90s  Codex Round 1 complete (via exec fallback)
T=90s  Round 2 begins — cross-critique prompts sent
```

This parallelism means a 3-round debate completes in roughly 3-5 minutes total, not 9-15 minutes if run sequentially.

### Consensus synthesis

After all rounds complete, Claude builds a table:

```
| # | Finding              | Codex | Gemini | Claude | Resolution              |
|---|----------------------|-------|--------|--------|-------------------------|
| 1 | LUNA Mexican Kitchen | ✓     | ✓      | ✓      | Unanimous (round 1)     |
| 2 | Puesto as runner-up  | ✓     | ✗→✓    | ✓      | Unanimous (round 2)     |
| 3 | Adelita's vegan menu | ✗     | ✓      | ✗      | Majority reject (2/3)   |
```

The `✗→✓` notation shows when a model changed its position after seeing the others' evidence — which is the whole point of the debate format.

---

## What Codex Did That Surprised Me

In our Mexican restaurant debate (Round 2), Codex went far beyond simple web searches. When its search tools failed, it:

1. Wrote a **Python haversine distance calculator** to compute exact distances from the target zip code to each restaurant candidate
2. Used `curl` to fetch restaurant websites and piped through `rg` (ripgrep) for menu analysis
3. Wrote a **Ruby script** to parse JSON-LD structured data from a restaurant's website, extracting all vegetarian and vegan dishes programmatically
4. Cross-referenced Yelp and Michelin guide pages to verify ratings claims
5. Discovered that a competitor model's recommended restaurant returned a 404 on its "vegan menu" page — effectively debunking the recommendation with evidence

This level of autonomous research — 13 shell commands, 64,980 tokens — is exactly what makes the multi-model debate format valuable. Each model brings different research strategies, and they catch each other's unverified claims.

---

## Open Issues and Next Steps

**Claude Desktop timeout:** The 60-second hardcoded MCP tool timeout needs a user-facing override. A `mcpToolTimeoutMs` field in `claude_desktop_config.json` would solve this. Until then, the `codex exec` fallback is the only reliable path for research-heavy queries.

**Codex config inconsistency:** The `web_search = "live"` (string) vs `web_search = true` (boolean) discrepancy between MCP and exec modes needs to be fixed upstream.

**Statefulness gap:** When falling back to `codex exec`, cross-round memory is lost. The skill compensates by injecting summaries of prior positions, but this is a lossy compression of the model's actual reasoning state.

**Token costs:** A full 3-round debate uses roughly 100,000-200,000 tokens across all three models. For casual questions, the `--consensus any` flag skips the debate and just presents three independent perspectives — much cheaper and usually sufficient.

---

## Conclusion

Multi-model debate isn't just a novelty. In our restaurant tests, the debate format caught factual errors (a non-vegetarian restaurant recommended as vegetarian-only), surfaced hidden gems (Madurai Iyer Mess, which only Gemini knew about), and forced each model to distinguish between "I'm guessing" and "I verified this." The consensus table gives you a clear view of confidence levels.

The engineering challenge is less about the AI and more about the plumbing. CLI tools, MCP servers, Electron app timeouts, config file format mismatches — these are the mundane obstacles that determine whether a multi-model system works reliably or fails mysteriously after 60 seconds.

The code is open source and available as a Claude Code skill. If you build on it, remember: always pass `--approval-mode yolo` on every Gemini `--resume` call, and never trust that a 60-second timeout is long enough for real research.

---

*The full technical analysis, including process traces, binary string searches, and timeout source code references, is available in [CODEX-MCP-TIMEOUT-ANALYSIS.md](./CODEX-MCP-TIMEOUT-ANALYSIS.md).*
