# Team Brainstorm Skill: Stateful Multi-Model Debate

**Status:** Proposal
**Date:** 2026-03-26
**Builds on:** `/dual-brainstorm`, `/codex-brainstorm`, `/gemini-brainstorm`

## Problem

The current `/dual-brainstorm` skill runs Codex and Gemini in parallel but they never see each other's output. Claude synthesizes alone. This is a 1-round, hub-and-spoke pattern — useful, but leaves value on the table.

In contrast, Claude Code's native `TeamCreate` + `SendMessage` enables true peer-to-peer debate between Claude sub-agents (as demonstrated in the Burlingame Chinese food search, where 3 agents independently searched, shared findings via DM, challenged each other's rankings, and converged on a unanimous top 3 over multiple rounds).

**Can we get that same multi-round debate dynamic across different AI models (Claude + Codex + Gemini)?**

## Constraint (Revisited)

The original assumption was that Codex and Gemini are one-shot CLI tools. **This is wrong.** Both support stateful multi-turn conversations:

### Codex: `codex mcp-server`

Codex ships a persistent MCP server on stdio exposing two tools:
- **`codex`** — start a new session, returns `{threadId, content}`
- **`codex-reply`** — continue an existing session by `threadId`, returns `{threadId, content}`

The MCP server maintains the conversation in memory. Full statefulness across turns.

### Gemini: `--resume SID -p "prompt"` (Verified Working)

Gemini supports stateful headless sessions via session resume:

```bash
# Round 1: Start session, capture session_id from JSON output
R1=$(gemini -p "Your task: research X" -o json --approval-mode yolo)
SID=$(echo "$R1" | python3 -c "import sys,json; print(json.load(sys.stdin)['session_id'])")

# Round 2: Resume same session with new prompt — full memory of Round 1
R2=$(gemini --resume "$SID" -p "Critique from other agent: ..." -o json --approval-mode yolo)

# Round 3: Resume again — remembers both Round 1 and Round 2
R3=$(gemini --resume "$SID" -p "Final position?" -o json --approval-mode yolo)
```

**Tested and verified (2026-03-26):** Gemini correctly:
- Returns `session_id` in `-o json` output
- Resumes sessions by ID with `--resume SID -p "..."`
- Maintains full conversation memory across 3+ turns
- Remembers its prior positions and can reference them
- Responds to challenges with genuine continuity (not "I was told I argued X" but actually remembering)

### Gemini ACP Mode (Broken — Multiple Issues)

Gemini implements ACP (Agent Client Protocol) via `gemini --acp`. Retested on **v0.35.1** (latest stable as of 2026-03-26):

| ACP Method | Status | Notes |
|---|---|---|
| `initialize` | ✅ Works | Returns agent capabilities, auth methods |
| `session/new` | ✅ Works | Creates session, returns `sessionId` |
| `session/set_mode` | ✅ Works | Sets yolo/plan/etc mode |
| `session/load` | ✅ Works | Loads saved sessions by UUID, index, or "latest"; streams history |
| `session/prompt` | ❌ **Broken** | Silently accepts, zero bytes on stdout/stderr. Confirmed processing happens server-side (cancellation returns `"stopReason":"cancelled"`), but turn events never reach stdout |
| `session/list` | ❌ Not implemented | `"Method not found"` |
| `session/cancel` | ❌ Not implemented | `"Method not found"` |
| `run_shell_command` | ❌ **Broken** (Issue #23507) | Fails silently — can't render confirmation prompts in non-interactive ACP |

**Conclusion:** ACP is fundamentally broken for headless/automated use. Multiple independent failure modes — not just `session/prompt`. Even if prompt streaming gets fixed, tool execution (`run_shell_command`) would still be unreliable. The `--resume -p` headless approach is the only reliable path.

### Gemini `-o stream-json` (Performance Optimization)

The `-o stream-json` flag outputs **newline-delimited JSONL events** in real-time:

| Event Type | Purpose |
|---|---|
| `init` | Session metadata (session ID, model) — **available immediately** |
| `message` | Streaming text chunks as they arrive |
| `tool_use` | Tool call requests with arguments |
| `tool_result` | Tool execution results |
| `error` | Non-fatal warnings |
| `result` | Final aggregated stats (tokens, cost) |

This is strictly better than `-o json` for the orchestrator:
- Extract `session_id` from `init` event immediately (no waiting for completion)
- Start processing partial results as chunks arrive
- Detect failures early via `error` events instead of waiting for timeout
- Can stream results to round files incrementally

```bash
# Improved Round 1: stream-json with early session_id capture
gemini -p "Research: $TOPIC" -o stream-json --approval-mode yolo | while IFS= read -r line; do
  type=$(echo "$line" | jq -r '.type // empty')
  case "$type" in
    init) SID=$(echo "$line" | jq -r '.sessionId'); echo "$SID" > "$SESSION/.gemini_sid" ;;
    message) echo "$line" | jq -r '.content // empty' >> "$SESSION/round1/gemini.md" ;;
    error) echo "WARN: $line" >&2 ;;
    result) break ;;
  esac
done
```

### Revised Constraint Summary

| Model | Stateful Interface | Protocol | Verified |
|---|---|---|---|
| **Codex** | `codex mcp-server` → `codex`/`codex-reply` by `threadId` | MCP (stdio JSON-RPC) | Yes |
| **Gemini** | `gemini --resume SID -p "..." -o stream-json` | CLI (sequential invocations, streaming JSONL) | Yes (3-turn test) |
| **Gemini** | `gemini --acp` → `session/prompt` | ACP (stdio JSON-RPC) | Broken in v0.35.1 (multiple issues) |
| **Claude** | `TeamCreate` + `SendMessage` | Native | Yes |

**All three models support stateful multi-turn conversations.** The coordination mechanism is no longer the bottleneck — the question is purely about orchestration topology.

## Recommended v1: Direct Stateful Orchestration (Variant B)

Now that Codex and Gemini are both stateful, the best v1 is to let Claude mediate a debate loop directly instead of routing everything through file-based rounds.

### Core Idea

Claude remains the team lead and explicit orchestrator:
- Codex participates through `codex` / `codex-reply` with a persistent `threadId`
- Gemini participates through `gemini -p` / `gemini --resume` with a persistent `session_id`
- Claude does its own research with WebSearch and codebase inspection
- The loop continues until the positions converge or the disagreement becomes clear enough to synthesize

### Why This Should Be v1

- It engages Codex in its strongest form: stateful, grounded, and able to defend or revise prior positions with real continuity
- It avoids the file-writing fragility of shell-managed message passing
- It avoids the extra interpretation layer of a Claude proxy agent
- It is simpler to validate than full peer-to-peer sub-agent teams

### Tool Integration

Codex must be registered as an MCP server in Claude Code's config so that `codex` and `codex-reply` appear as native tools. The skill should verify this at startup and provide setup instructions if missing:

```json
// In Claude Code MCP settings
{
  "mcpServers": {
    "codex": {
      "command": "codex",
      "args": ["mcp-server"]
    }
  }
}
```

If the MCP server is unavailable, the skill falls back to `codex exec` (one-shot, stateless) and degrades gracefully — the debate still works, Codex just loses cross-round memory.

Gemini is invoked via Bash (`gemini -p ... -o stream-json`), with `session_id` extracted from the `init` JSONL event and passed to subsequent `--resume` calls.

### High-Level Loop

```
Claude (team lead, sole orchestrator)
  │
  ├── Round 1: Independent Research (parallel where possible)
  │     codex({prompt: "Research / analyze topic X"}) → threadId
  │     gemini -p "Research / analyze topic X" -o stream-json → session_id
  │     Claude does its own research (WebSearch, codebase)
  │
  ├── Round 2: Cross-Critique
  │     codex-reply(threadId, "Gemini argues A. Respond.")
  │     gemini --resume session_id -p "Codex argues B. Respond."
  │     Claude evaluates positions and identifies agreement/disagreement
  │
  ├── Round 3 (if needed): Rebuttal
  │     codex-reply(threadId, "Gemini conceded X but insists on Y. Final position?")
  │     gemini --resume session_id -p "Codex defends Z. Final position?"
  │
  └── Synthesis: Claude produces consensus table and narrative
```

### Convergence Criteria

**Default:** Up to 3 rounds. After each round, Claude checks whether the models have reached consensus. If all three agree, stop early. If not, continue to the next round. After round 3, use simple majority (2/3) to resolve remaining disagreements.

**User override:** The skill accepts an optional convergence parameter:
- `--rounds N` — set max round count (default: 3)
- `--consensus strict` — require unanimity, flag anything without 3/3 agreement instead of majority-resolving
- `--consensus majority` — (default) resolve via 2/3 after max rounds
- `--consensus any` — stop after round 1, no debate (equivalent to `/dual-brainstorm`)

The synthesis table reflects how each item was resolved:

```
| Finding     | Codex | Gemini | Claude | Resolution              |
|-------------|-------|--------|--------|-------------------------|
| Pick X at 1 | Yes   | Yes    | Yes    | Unanimous (round 1)     |
| Pick Y at 2 | Yes   | No→Yes | Yes    | Unanimous (round 2)     |
| Pick Z at 3 | No    | Yes    | No     | Majority reject (2/3)   |
| Pick W at 3 | Yes   | No     | Yes    | Majority accept (2/3)   |
```

### Context Cost Management

Claude mediates all traffic in Variant B, so its context window accumulates every round's output. For a 3-round debate with verbose models, this can reach 15-20k tokens of debate content alone.

**Mitigations:**
- Instruct external models to keep responses concise ("under 500 words per round")
- Claude summarizes each round before injecting it into the next prompt, rather than forwarding verbatim
- The skill can use `stream-json` message events to incrementally capture Gemini output without loading the full response into Claude's context

This gives you model diversity plus true continuity for Codex and Gemini without taking on sub-agent complexity yet.

## Baseline Fallback: File-Based Message Passing with Bash Orchestrator

This remains useful as a portable baseline when MCP setup is unavailable or when you want the simplest possible implementation, but it is no longer the recommended primary design.

### Core Idea

Use the filesystem as a shared message bus. Each model writes its output to a known file path. A lightweight bash script sequences the rounds using `wait` to synchronize. Claude is **not in the loop** during the debate — it only reads the final outputs.

### Architecture

```
Claude (team lead)
  │
  ├── Launches orchestrator.sh as background Bash task
  │     │
  │     ├── Round 1 (parallel): codex & gemini research independently
  │     │     codex exec "..." → writes round1/codex.md
  │     │     gemini -p "..."  → writes round1/gemini.md
  │     │     wait (both finish)
  │     │
  │     ├── Round 2 (parallel): each reads the other's Round 1 output
  │     │     codex exec "read round1/*.md, pick top 3, critique" → round2/codex.md
  │     │     gemini -p "read round1/*.md, pick top 3, critique"  → round2/gemini.md
  │     │     wait
  │     │
  │     └── Round 3 (parallel): each reads the other's Round 2 picks
  │           codex exec "read round2/*.md, final position" → round3/codex.md
  │           gemini -p "read round2/*.md, final position"  → round3/gemini.md
  │           wait → writes DONE sentinel
  │
  ├── Meanwhile: Claude does its own research (WebSearch, codebase, etc.)
  │
  └── On DONE: reads round3/*.md, synthesizes 3-way consensus
```

### Session Directory Structure

```
/tmp/team-brainstorm-{session-id}/
├── prompt.md                  # Original topic/question
├── round1/
│   ├── codex.md               # Codex independent findings
│   └── gemini.md              # Gemini independent findings
├── round2/
│   ├── codex.md               # Codex top 3 + critique of Gemini
│   └── gemini.md              # Gemini top 3 + critique of Codex
├── round3/
│   ├── codex.md               # Codex final position after rebuttal
│   └── gemini.md              # Gemini final position after rebuttal
└── DONE                       # Sentinel file — orchestrator finished
```

### Orchestrator Script (generated by the skill at runtime)

```bash
#!/bin/bash
set -euo pipefail

SESSION="$1"        # e.g., /tmp/team-brainstorm-abc123
TOPIC="$2"          # e.g., "best Chinese restaurants near Burlingame CA"
PROMPT_FILE="$SESSION/prompt.md"

mkdir -p "$SESSION"/{round1,round2,round3}

# Detect git repo for Codex
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  GIT_FLAG=""
else
  GIT_FLAG="--skip-git-repo-check"
fi

# ─── ROUND 1: Independent Research ───────────────────────────────
echo "[round1] Starting parallel research..."

codex exec $GIT_FLAG \
  --config 'approval_policy="never"' \
  --config 'sandbox_permissions=["disk-full-read-access"]' \
  "You are a research agent. Your task: $TOPIC

Research thoroughly. Provide detailed findings with:
- Specific names, addresses, ratings, review counts
- What makes each option stand out
- Your sources

Write your complete findings to: $SESSION/round1/codex.md
IMPORTANT: You MUST write your output to that file path." 2>&1 &
CODEX_PID=$!

gemini --approval-mode yolo -p \
  "You are a research agent. Your task: $TOPIC

Research thoroughly. Provide detailed findings with:
- Specific names, addresses, ratings, review counts
- What makes each option stand out
- Your sources

Write your complete findings to: $SESSION/round1/gemini.md
IMPORTANT: You MUST write your output to that file path." 2>&1 &
GEMINI_PID=$!

wait $CODEX_PID $GEMINI_PID
echo "[round1] Complete."

# ─── ROUND 2: Cross-Critique ─────────────────────────────────────
echo "[round2] Starting cross-critique..."

CODEX_R1=$(cat "$SESSION/round1/codex.md" 2>/dev/null || echo "(no output)")
GEMINI_R1=$(cat "$SESSION/round1/gemini.md" 2>/dev/null || echo "(no output)")

codex exec $GIT_FLAG \
  --config 'approval_policy="never"' \
  --config 'sandbox_permissions=["disk-full-read-access"]' \
  "You are a debate agent. Two research agents investigated: $TOPIC

AGENT A (Codex) found:
$CODEX_R1

AGENT B (Gemini) found:
$GEMINI_R1

Your job:
1. Review BOTH sets of findings
2. Pick YOUR top 3 from the combined list
3. For each pick, explain WHY — argue your case
4. Challenge any picks from Agent B you disagree with
5. Be opinionated and direct

Write your top 3 picks with reasoning to: $SESSION/round2/codex.md" 2>&1 &
CODEX_PID=$!

gemini --approval-mode yolo -p \
  "You are a debate agent. Two research agents investigated: $TOPIC

AGENT A (Codex) found:
$CODEX_R1

AGENT B (Gemini) found:
$GEMINI_R1

Your job:
1. Review BOTH sets of findings
2. Pick YOUR top 3 from the combined list
3. For each pick, explain WHY — argue your case
4. Challenge any picks from Agent A you disagree with
5. Be opinionated and direct

Write your top 3 picks with reasoning to: $SESSION/round2/gemini.md" 2>&1 &
GEMINI_PID=$!

wait $CODEX_PID $GEMINI_PID
echo "[round2] Complete."

# ─── ROUND 3: Final Rebuttal ─────────────────────────────────────
echo "[round3] Starting final rebuttal..."

CODEX_R2=$(cat "$SESSION/round2/codex.md" 2>/dev/null || echo "(no output)")
GEMINI_R2=$(cat "$SESSION/round2/gemini.md" 2>/dev/null || echo "(no output)")

codex exec $GIT_FLAG \
  --config 'approval_policy="never"' \
  --config 'sandbox_permissions=["disk-full-read-access"]' \
  "Final round of debate on: $TOPIC

Your previous picks:
$CODEX_R2

The other agent's picks and critiques:
$GEMINI_R2

Respond to their challenges. Defend or concede your positions.
State your FINAL top 3 with brief reasoning.
Note any points of agreement and remaining disagreements.

Write your final position to: $SESSION/round3/codex.md" 2>&1 &
CODEX_PID=$!

gemini --approval-mode yolo -p \
  "Final round of debate on: $TOPIC

Your previous picks:
$GEMINI_R2

The other agent's picks and critiques:
$CODEX_R2

Respond to their challenges. Defend or concede your positions.
State your FINAL top 3 with brief reasoning.
Note any points of agreement and remaining disagreements.

Write your final position to: $SESSION/round3/gemini.md" 2>&1 &
GEMINI_PID=$!

wait $CODEX_PID $GEMINI_PID
echo "[round3] Complete."

# ─── SENTINEL ─────────────────────────────────────────────────────
touch "$SESSION/DONE"
echo "All rounds complete. Results in $SESSION/round3/"
```

### Claude's Role (Skill SKILL.md instructions)

```
Step 1: Generate and launch the orchestrator script as a single
        background Bash command with run_in_background: true.

Step 2: While the script runs (~3-5 minutes), do your OWN independent
        research on the same topic using WebSearch and any other tools.
        Do NOT read any round files until your own analysis is complete.

Step 3: When the background task completes (DONE sentinel), read:
        - round3/codex.md
        - round3/gemini.md
        Optionally read round1/ and round2/ to understand the debate arc.

Step 4: Synthesize a 3-way consensus table:

        | Finding     | Codex | Gemini | Claude | Action          |
        |-------------|-------|--------|--------|-----------------|
        | Pick X at 1 | Yes   | Yes    | Yes    | Unanimous       |
        | Pick Y at 2 | Yes   | No     | Yes    | 2/3 — accept    |
        | Pick Z at 3 | No    | Yes    | No     | 1/3 — flag      |

Step 5: Present the final consensus to the user with the debate
        narrative (who argued what, who conceded).
```

## Comparison of Baseline / Low-Complexity Approaches

| Dimension | `/dual-brainstorm` (current) | Claude Team (`TeamCreate`) | File-Based Baseline |
|---|---|---|---|
| **Models** | Codex + Gemini + Claude | Claude + Claude + Claude | Codex + Gemini + Claude |
| **Rounds** | 1 (independent) | Organic (as many as needed) | 3 (structured) |
| **Communication** | None between tools | Real-time peer DMs | File-based between rounds |
| **Claude in loop?** | Yes (sole synthesizer) | Yes (team lead, but agents self-coordinate) | No (reads final only) |
| **Debate quality** | No debate | Rich (challenge, concede, converge) | Structured (critique, rebut, finalize) |
| **Model diversity** | 3 different architectures | Single architecture | 3 different architectures |
| **Latency** | ~2 min (1 round) | ~3-5 min (organic) | ~4-6 min (3 rounds) |
| **Context cost** | Low (Claude reads 2 outputs) | High (all relay traffic) | Low (Claude reads 2-6 files) |
| **Web search** | Codex: sometimes; Gemini: yes | Native WebSearch | Codex: sometimes; Gemini: yes |
| **Setup complexity** | Low (2 Bash calls) | Medium (TeamCreate + tasks + agents) | Medium (1 script + Bash call) |

> **Note:** The table above compares low-complexity baselines. The recommended v1 is the stateful direct-orchestration loop described earlier in this document. The file-based design remains here as the simplest fallback.

## Known Limitations

### Applies to v1 (Direct Stateful Orchestration)

**1. Codex web search may be unavailable**
Codex web search availability depends on the sandbox configuration and model. It may or may not be able to browse the web for a given invocation.

**Mitigation:** Prompt Codex to do web research like the other agents. If it fails to produce web-sourced findings, it naturally falls back to reasoning over what Claude and Gemini found — acting as a critic who evaluates others' research rather than an independent researcher. No special handling needed; just don't assume it will or won't have web access.

**2. Codex MCP server availability**
The v1 design requires `codex mcp-server` registered in Claude Code's MCP config. If it's not configured, the `codex`/`codex-reply` tools won't be available.

**Mitigation:** The skill checks for tool availability at startup. If Codex MCP is missing, it falls back to `codex exec` (one-shot, stateless) via Bash. The debate still works — Codex just loses cross-round memory and each round gets the full prior context inlined in the prompt.

**3. Claude context accumulation**
Claude mediates all traffic, so its context window grows with every round. A 3-round debate with verbose models can add 15-20k tokens of debate content.

**Mitigation:** Instruct external models to keep responses under 500 words per round. Claude summarizes each round internally before injecting into the next prompt rather than forwarding verbatim.

**4. Model refusal / tool unavailability**
Either CLI tool might not be installed, might error out, or might refuse the task.

**Mitigation:** The skill degrades gracefully — if Codex fails, the debate continues as Gemini + Claude (2-way). If Gemini fails, Codex + Claude. If both fail, Claude reports the issue and offers its own single-model analysis.

**5. Response length asymmetry**
If one model produces 2000-word responses and another produces 200 words, the verbose model dominates the debate framing and the concise model's points may get lost.

**Mitigation:** Prompt each model with explicit length targets ("respond in 200-500 words"). Claude should also weight arguments by quality, not length, when synthesizing.

### Applies to Baseline Fallback (File-Based) Only

**6. File writing reliability**
Codex and Gemini must be explicitly told to write to a file path. They sometimes print to stdout instead, write to a different path, or truncate long outputs.

**Mitigation:** Capture stdout as a fallback: `codex exec ... "..." > "$SESSION/round1/codex.md" 2>&1`

**7. Prompt size limits in bash orchestrator**
By Round 3, the bash script inlines all prior round outputs into the prompt string. If Round 1 findings are verbose, Round 3 prompts may hit shell argument or token limits.

**Mitigation:** Add word limits to each round's instructions. Alternatively, instruct models to `cat` the round files themselves rather than inlining content.

**8. Codex stdin limitation**
`codex exec` does not read piped stdin — all content must be inlined in the prompt argument.

**Mitigation:** Instruct Codex to read files from disk via its sandbox permissions: `"Read the files in $SESSION/round1/ before responding..."`

## Open Questions

1. **Should we support topic-specific prompt templates?** Restaurant search vs. code architecture review vs. technology comparison need different Round 1 prompts. Could ship a few templates, or let the skill generate prompts dynamically based on the topic type.

2. **How should Claude share its own web research with the other models?** Claude has native WebSearch and Codex may or may not have web access depending on config. Should Claude inject its findings into the Round 2 prompts for both models, or keep its research private until synthesis? Sharing early makes the debate more informed; keeping it private preserves independence.

3. **Should we build the richer team version (Variant A) later?** Variant A uses Claude sub-agents (TeamCreate) with each sub-agent proxying a different external model. This gives true peer-to-peer debate semantics (via SendMessage) while still getting model diversity. Worth pursuing once v1 validates the core value prop. **See detailed analysis below.**

4. **Should Gemini ACP be revisited when fixed?** ACP `session/prompt` is broken as of v0.35.1 but the session management layer (`session/new`, `session/load`) works. If Google fixes prompt streaming, ACP would give Gemini parity with Codex's MCP — persistent process, no cold-start overhead. Worth monitoring.

## Stateful Multi-Model Designs

### Key Discovery: `codex mcp-server` Changes Everything

The original hybrid analysis assumed Codex is one-shot only. But `codex mcp-server` runs as a **persistent MCP server on stdio** and exposes two tools:

```json
{
  "tools": [
    {
      "name": "codex",
      "description": "Run a Codex session. Returns {threadId, content}.",
      "inputSchema": { "required": ["prompt"], ... }
    },
    {
      "name": "codex-reply",
      "description": "Continue a Codex conversation by threadId.",
      "inputSchema": { "required": ["prompt"], "properties": { "threadId": "..." } }
    }
  ]
}
```

This means Codex can maintain **full conversation state across turns**. The MCP server holds the session in memory — no context re-packing, no "you previously argued X" prompts. Codex actually remembers what it argued.

This eliminates the biggest gap between the hybrid and pure-Claude approaches.

### Two Stateful Variants

#### Variant A: Sub-Agent Proxy (original concept, improved with MCP)

Claude sub-agents use `TeamCreate` + `SendMessage` for peer-to-peer debate. Each sub-agent talks to its external model via the appropriate interface:

```
Claude (team lead)
  │
  ├── TeamCreate("codex-agent")
  │     MCP tools: codex, codex-reply (via codex mcp-server)
  │     Protocol: Start session with `codex` tool, continue with
  │               `codex-reply` using threadId. Relay responses
  │               to peers via SendMessage.
  │
  ├── TeamCreate("gemini-agent")
  │     Tools: Bash (for gemini -p / --resume)
  │     Protocol: Start with `gemini -p "..." -o stream-json`, capture session_id.
  │               Continue with `gemini --resume SID -p "..."` each turn.
  │               Full statefulness. Relay responses via SendMessage.
  │
  └── TeamCreate("claude-agent")  [or team lead plays this role]
        Tools: WebSearch, SendMessage
        Protocol: Native reasoning + web search.
```

**Codex sub-agent flow:**
```
Turn 1: codex({prompt: "Research best Chinese food near Burlingame"})
        → {threadId: "abc-123", content: "I found these restaurants..."}
        → SendMessage to peers with findings

Turn 2: codex-reply({threadId: "abc-123",
         prompt: "Gemini challenges your #2 pick — 3.5 stars on Yelp"})
        → {threadId: "abc-123", content: "The Yelp rating doesn't reflect..."}
        → SendMessage with defense

Turn 3: codex-reply({threadId: "abc-123",
         prompt: "Claude found that Pick X closed last month"})
        → {threadId: "abc-123", content: "Dropping X, replacing with..."}
        → SendMessage with updated list
```

Codex has **full memory** of its prior positions. It genuinely "remembers" arguing for Y and can choose to defend or concede with real continuity.

#### Variant B: Direct MCP Integration (no sub-agent wrapper)

**This is the recommended v1.** See "Recommended v1: Direct Stateful Orchestration" above for the full design, including tool integration, convergence criteria, and context cost management.

**Pros:** Simpler (no TeamCreate), cheaper (no sub-agent tokens), team lead has full visibility, all models stateful.
**Cons:** No peer-to-peer messaging, Claude mediates everything, no parallel agent execution.

### Team Semantics Comparison

| Claude Team Semantic | Variant A (Sub-Agent + MCP) | Variant B (Direct MCP) | Pure Claude Team |
|---|---|---|---|
| **Peer-to-peer messaging** | Yes (SendMessage) | No (Claude mediates) | Yes |
| **Named addressable agents** | Yes | No (single agent) | Yes |
| **Team lead orchestration** | Yes | Yes (team lead IS the agent) | Yes |
| **Organic round count** | Yes | Yes | Yes |
| **Agents challenge each other** | Yes (via DMs) | Simulated (Claude relays) | Yes |
| **Agents concede positions** | Yes | Simulated | Yes |
| **Convergence detection** | Yes (team lead monitors) | Yes (agent decides) | Yes |
| **Parallel research** | Yes (all agents start at once) | Partial (can parallel Bash) | Yes |
| **External model statefulness** | Codex: full (MCP), Gemini: full (resume) | Same | N/A (all Claude) |
| **Model diversity** | 3 models | 3 models | 1 model |
| **Context accumulation** | Full for all | Full within each model session; Claude-mediated across models | Full for all |

### What's Still Different from Pure Claude Teams

With both Codex and Gemini now stateful, the gaps narrow significantly. What remains:

**1. Latency per turn is still higher**

Each external model call adds ~10-30s per turn. In a pure Claude team, `SendMessage` is near-instant. A 5-turn debate:
- Pure Claude team: ~2-3 min
- Hybrid Variant A: ~4-6 min (Codex MCP is faster than `codex exec` but still slower than Claude-to-Claude)
- Hybrid Variant B: ~3-5 min (less overhead, sequential)

**2. Sub-agent interpretation layer (Variant A only)**

Claude sub-agents relay external model output. This adds a layer of interpretation that could:
- Soften controversial positions
- Introduce Claude's worldview bias
- Lose nuance in summarization

**Mitigation:** Prompt sub-agents to relay verbatim when output is short, summarize only when necessary.

**3. Cost**

| Approach | Estimated cost per debate |
|---|---|
| File-based (bash orchestrator) | ~$0.10-0.15 |
| Pure Claude team | ~$0.15-0.25 |
| Variant B (direct MCP) | ~$0.20-0.30 |
| Variant A (sub-agent + MCP) | ~$0.30-0.50 |

### Key Design Decision: How Much Autonomy for the Sub-Agent?

*(Applies to Variant A only)*

There's a spectrum:

**Thin proxy (minimal autonomy):**
```
Sub-agent prompt: "Call codex/codex-reply for positions.
                   Send its exact output to the other agents.
                   Do not add your own opinions."
```
- Preserves model diversity maximally
- Sub-agent is basically a protocol adapter
- Cheapest in Claude tokens

**Thick proxy (high autonomy):**
```
Sub-agent prompt: "You champion Codex's perspective. Use codex tools
                   to get Codex's raw take, then strengthen the argument.
                   Add supporting evidence. Anticipate counterarguments.
                   You may diverge from Codex if you think it missed something."
```
- Sub-agent becomes a "lawyer" for the external model
- Debate quality is higher (better-argued positions)
- But you lose some model diversity (Claude is mediating everything)

**Recommended: Medium proxy**
```
Sub-agent prompt: "Invoke codex/codex-reply to get Codex's position. Relay it
                   faithfully but concisely. When challenged by other agents,
                   use codex-reply to get Codex's actual response to the critique.
                   You may summarize but do not add arguments Codex didn't make."
```

### Example Flow: Burlingame Restaurants (Variant A with MCP)

```
1. Team lead creates 3 agents, sets topic

2. All agents research in parallel:
   - codex-agent: codex({prompt: "research..."}) → threadId=abc
     sends findings via SendMessage
   - gemini-agent: gemini -p "research..." → sends findings via SendMessage
   - claude-agent: WebSearch → sends findings via SendMessage

3. Team lead: "Everyone share your top 3 picks"
   - Each agent sends their ranked list via SendMessage

4. gemini-agent → codex-agent: "Your #2 pick has 3.5 stars, why?"
   - codex-agent: codex-reply({threadId: "abc",
       prompt: "Gemini challenges #2 pick — 3.5 stars on Yelp"})
     Codex *remembers* picking it and responds with genuine continuity
   - codex-agent → gemini-agent: relays Codex's defense

5. claude-agent → both: "Pick X closed last month"
   - codex-agent: codex-reply({threadId: "abc",
       prompt: "Claude says Pick X closed last month"})
     Codex updates its list with full memory of why it originally picked X
   - gemini-agent: gemini --resume SID -p "Claude says Pick X closed"
     Gemini updates its list with full memory of its prior positions

6. Team lead detects convergence after 3-4 exchanges. Final vote.

7. Agents send final positions. Team lead synthesizes.
```

The key difference from the pre-MCP analysis: in step 4, Codex isn't being *told* it argued for that restaurant — it actually *remembers* arguing for it. The debate is genuine.

### Comparison: All Four Approaches

| Dimension | File-Based | Pure Claude Team | Hybrid A (Sub-Agent+MCP) | Hybrid B (Direct MCP) |
|---|---|---|---|---|
| **Debate protocol** | Fixed 3 rounds | Organic SendMessage | Organic SendMessage | Orchestrated loop |
| **Model diversity** | 3 models | 1 model | 3 models | 3 models |
| **Codex statefulness** | None | N/A | Full (MCP threadId) | Full (MCP threadId) |
| **Gemini statefulness** | None | N/A | Full (resume SID) | Full (resume SID) |
| **Debate quality** | Structured but rigid | Rich and adaptive | Rich and adaptive | Good, Claude-mediated |
| **Peer-to-peer** | No | Yes | Yes | No |
| **Latency** | ~4-6 min | ~2-3 min | ~4-6 min | ~3-5 min |
| **Cost** | Low | Medium | High | Medium |
| **Implementation** | Bash script | TeamCreate + prompts | TeamCreate + MCP + Bash | MCP + Bash |
| **Failure handling** | Script fallback | Agent retry | Sub-agent fallback | Direct retry |
| **Convergence** | Fixed | Dynamic | Dynamic | Dynamic |

### Variant-Specific Risks

1. **Double interpretation (Variant A):** External model output goes through the Claude sub-agent before reaching peers. Can soften positions or introduce Claude bias.

2. **Sub-agent may "go native" (Variant A):** Over rounds, the Claude sub-agent may stop calling `codex-reply` and argue from its own reasoning.
   - **Mitigation:** Prompt must require calling the external model tool before every SendMessage. "You MUST call codex-reply before responding to any challenge."

3. **MCP server lifecycle:** `codex mcp-server` runs as a persistent process. Need to manage startup and teardown — who launches it, and when does it stop?
   - If registered in Claude Code's MCP config: managed automatically by the harness.
   - If launched per-session: the skill must start and stop it.

4. **Gemini process overhead:** Unlike Codex (persistent MCP server), Gemini spawns a new process per turn via `--resume`. Each turn has ~2-3s startup overhead. For a 5-turn debate this adds ~10-15s total — noticeable but not blocking.

### Recommendation

**Variant B for v1** (see "Recommended v1" section above). **Variant A as a stretch goal** once v1 validates the core value of multi-model stateful debate. Variant A adds peer-to-peer semantics but at the cost of sub-agent prompts, MCP + SendMessage coordination, and lifecycle management.

## Recommended Next Steps

1. **Prototype v1 with 2 rounds** on one concrete topic to validate the `codex`/`codex-reply` + Gemini `--resume` loop end to end
2. **Verify Codex MCP setup and fallback** — confirm the skill degrades gracefully to `codex exec` when the MCP server is not registered
3. **Measure output quality, latency, and cost** against both the pure Claude team approach and `/dual-brainstorm` to confirm the multi-round statefulness is paying for itself
4. **Build the SKILL.md** with convergence parameters (`--rounds`, `--consensus`) and the consensus table output format
5. **Monitor Gemini ACP** — if `session/prompt` gets fixed in a future release, evaluate switching from `--resume` to ACP for lower per-turn latency
