# Hybrid Agent Teams Proposal: True Coordinated Multi-Model Debate

**Status:** Proposal
**Date:** 2026-03-29
**Builds on:** `team-brainstorm` v1 (Direct Stateful Orchestration), Claude Code Agent Teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`)

## Problem

The current `/team-brainstorm` v1 uses a single Claude session as both orchestrator and participant. Claude mediates all cross-model communication — it calls Codex MCP, runs Gemini CLI, does its own research, summarizes each round, and injects summaries into the next round's prompts. This works, but has structural limitations:

1. **No true parallelism.** Claude can run Codex and Gemini in background tasks, but it blocks waiting for results before starting the next round. Research phases overlap; debate phases don't.
2. **Single context accumulation.** Every round's output from every model flows through Claude's context window. A 3-round debate adds 15-20k tokens of relay traffic.
3. **No peer-to-peer debate.** Codex never "talks to" Gemini directly. Claude paraphrases and relays, which can soften positions, lose nuance, or introduce Claude's own bias.
4. **Orchestration complexity in a single prompt.** The SKILL.md is 350+ lines of instructions for one agent to juggle three models, parse JSONL, manage session IDs, and detect convergence — all in a single context.

## Proposal: Agent Teams as Orchestration Layer

Use Claude Code's experimental Agent Teams feature (`TeamCreate`, `SendMessage`, `TeamDelete`) to distribute the orchestration across dedicated teammates, while preserving multi-model diversity by having each teammate drive a different external model.

### Architecture

```
Claude (Team Lead)
  │
  │  TeamCreate → spawns 3 teammates
  │
  ├── "codex-researcher" (Claude teammate)
  │     Role: Drives Codex via MCP tools (codex / codex-reply)
  │     Has: codex MCP tools, SendMessage, Bash
  │     Maintains: Codex threadId across rounds
  │
  ├── "gemini-researcher" (Claude teammate)
  │     Role: Drives Gemini via CLI (gemini -p / --resume)
  │     Has: Bash, SendMessage
  │     Maintains: Gemini session_id across rounds
  │
  └── "claude-researcher" (Claude teammate)
        Role: Does Claude's own independent research
        Has: WebSearch, Grep, Glob, Read, SendMessage
        Maintains: Own findings in context
```

The team lead does NOT participate in the debate. It orchestrates: spawns teammates, sets the topic, monitors for convergence, and synthesizes the final consensus.

### Why This Is Better Than v1

| Dimension | v1 (Single Orchestrator) | Hybrid Agent Teams |
|---|---|---|
| **Parallelism** | Background tasks, Claude waits | True parallel — all teammates work simultaneously |
| **Context pressure** | All 3 models' output in one window | Each teammate holds only its own model's history |
| **Communication** | Claude relays summaries | `SendMessage` — teammates share findings directly |
| **Orchestration** | 350-line SKILL.md, one agent does everything | Distributed — each teammate has a focused role |
| **Debate fidelity** | Claude paraphrases between models | Teammates relay their model's actual words |
| **Model diversity** | 3 model families | 3 model families (preserved) |
| **Failure isolation** | One model failure can derail the whole skill | Teammate failure is contained; debate continues 2v1 |

### Why This Is Better Than Pure Agent Teams (All-Claude)

A pure Agent Teams debate (3 Claude teammates debating each other) loses the primary value of `/team-brainstorm`: **model diversity**. Three Claude instances share the same training data, the same reasoning patterns, and the same blind spots. They converge faster, but convergence on a shared bias isn't consensus — it's an echo chamber.

The hybrid preserves genuine diversity: GPT-family reasoning (Codex), Gemini's training and web search, and Claude's own perspective — coordinated through Agent Teams infrastructure.

## Detailed Design

### Step 0: Preflight

The team lead verifies prerequisites:

1. **Agent Teams enabled:** Check that `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is set. If not, abort with setup instructions.
2. **Codex available:** Check for `codex` MCP tools or `codex` CLI on PATH. Determines whether `codex-researcher` uses MCP or CLI fallback.
3. **Gemini available:** Check for `gemini` on PATH.
4. **Entrypoint detection:** Check `CLAUDE_CODE_ENTRYPOINT`. If `claude-desktop`, warn that MCP timeouts may affect Codex MCP mode and prefer CLI resume for the Codex teammate.

If both Codex and Gemini are missing, fall back to standard v1 behavior (no teams needed for single-model analysis).

Parse arguments: topic/file, `--rounds N`, `--consensus strict|majority|any`.

### Step 1: Team Creation

The team lead spawns three teammates with focused role prompts:

#### codex-researcher

```
You are the Codex representative in a multi-model debate.

TOPIC: {TOPIC}
{PLAN_CONTENT if applicable}

Your job:
1. Use the `codex` MCP tool to get Codex's research on this topic. Save the threadId.
2. Send Codex's findings to the other teammates via SendMessage.
3. When you receive challenges from other teammates, use `codex-reply` with
   the same threadId to get Codex's actual response. Relay it faithfully.
4. You are a FAITHFUL PROXY. Relay Codex's positions accurately. Do not
   add your own arguments or soften Codex's positions. You may summarize
   for brevity but never change the substance.
5. If codex MCP is unavailable, fall back to `codex exec` via Bash.
   Use `codex exec resume {THREAD_ID}` for subsequent rounds if supported.

IMPORTANT: You MUST call codex or codex-reply before every SendMessage
in a debate round. Never argue from your own reasoning — always get
Codex's actual response first.
```

#### gemini-researcher

```
You are the Gemini representative in a multi-model debate.

TOPIC: {TOPIC}
{PLAN_CONTENT if applicable}

Your job:
1. Run `gemini -p "..." -o json --approval-mode yolo` to get Gemini's
   research. Extract the session_id from the JSON output.
2. Send Gemini's findings to the other teammates via SendMessage.
3. When you receive challenges from other teammates, run
   `gemini --resume {SESSION_ID} -p "..." -o json --approval-mode yolo`
   to get Gemini's actual response. Relay it faithfully.
4. You are a FAITHFUL PROXY. Relay Gemini's positions accurately. Do not
   add your own arguments or soften Gemini's positions. You may summarize
   for brevity but never change the substance.
5. CRITICAL: Pass `--approval-mode yolo` on EVERY call including --resume.
   It is not inherited across sessions.

IMPORTANT: You MUST call gemini before every SendMessage in a debate round.
Never argue from your own reasoning — always get Gemini's actual response first.
```

#### claude-researcher

```
You are the Claude representative in a multi-model debate.

TOPIC: {TOPIC}
{PLAN_CONTENT if applicable}

Your job:
1. Do your own independent research using WebSearch, codebase inspection
   (Grep, Glob, Read), and your own reasoning.
2. Send your findings to the other teammates via SendMessage.
3. When you receive challenges from other teammates, respond with your
   own updated analysis. You may do additional research to verify claims.
4. Be opinionated — take clear positions you're willing to defend.
5. When you concede a point, say so explicitly.
```

### Step 2: Round 1 — Independent Research (Parallel)

All three teammates begin research simultaneously. This is the key advantage over v1 — no sequential blocking.

```
Team Lead → SendMessage (broadcast):
  "Round 1: Research the topic independently. Send your findings
   to all teammates when done. Keep findings under 500 words.
   Tag your message with [ROUND 1 FINDINGS]."
```

Each teammate:
1. Calls their respective model (Codex MCP, Gemini CLI, or Claude's own tools)
2. Sends findings to the other two teammates via `SendMessage`
3. Waits for the other teammates' findings

The team lead monitors for all three `[ROUND 1 FINDINGS]` messages. Once all three are in, it triggers Round 2.

If `--consensus any` was specified, skip to synthesis after Round 1.

### Step 3: Round 2 — Cross-Critique (Parallel)

```
Team Lead → SendMessage (broadcast):
  "Round 2: You've seen the other researchers' findings.
   Challenge positions you disagree with — cite specific evidence.
   Concede points where their evidence is stronger than yours.
   State your updated position. Under 500 words.
   Tag your message with [ROUND 2 POSITION]."
```

Each teammate:
1. Reviews the other two teammates' Round 1 findings (received via SendMessage)
2. Calls their model with the critique prompt, including the other models' positions
3. Sends their updated position to all teammates

**Peer-to-peer debate can happen organically here.** If `codex-researcher` directly challenges `gemini-researcher`, Gemini's teammate can respond with a follow-up `SendMessage` without waiting for the team lead. This is the true peer-to-peer dynamic that v1 cannot achieve.

The team lead checks for early convergence: if all three positions now agree, skip to synthesis.

### Step 4: Round 3 — Final Rebuttal (If Needed)

Only runs if disputes remain and `--rounds` allows it.

```
Team Lead → SendMessage (broadcast):
  "Final round. Remaining disputes:
   {List of disagreements identified by team lead}

   Defend or concede on each disputed point. State your FINAL position.
   Be decisive — no hedging. Under 300 words.
   Tag your message with [FINAL POSITION]."
```

Each teammate calls their model one last time, relaying the specific disputes to address, and sends their final position.

### Step 5: Synthesis

The team lead collects all `[FINAL POSITION]` messages (or `[ROUND 2 POSITION]` if debate ended early) and builds the consensus table:

```markdown
## Consensus

| # | Finding | Codex | Gemini | Claude | Resolution |
|---|---------|-------|--------|--------|------------|
| 1 | {desc}  | ✓     | ✓      | ✓      | Unanimous (round 1) |
| 2 | {desc}  | ✓     | ✗→✓   | ✓      | Unanimous (round 2, Gemini conceded) |
| 3 | {desc}  | ✗     | ✓      | ✓      | Majority accept (2/3) |
| 4 | {desc}  | ✓     | ✗      | ✗      | Majority reject (2/3) |
| 5 | {desc}  | ✓     | ✗      | ✓      | ⚠ No consensus — flagged for review |

## Debate Narrative

{3-5 sentence summary of who argued what, who conceded, key turning points}

## Accepted Findings

{Resolved list based on consensus mode}

## Flagged for Review

{Items that didn't meet threshold, with each model's reasoning}
```

### Step 6: Cleanup

```
Team Lead → TeamDelete
```

Tears down the team, cleans up task files and inboxes.

## Communication Protocol

### Message Format

All inter-teammate messages follow a structured format for reliable parsing:

```
[ROUND {N} {TYPE}] from {role}

{content}
```

Where `TYPE` is one of: `FINDINGS`, `POSITION`, `CHALLENGE`, `CONCESSION`, `FINAL POSITION`.

### Message Flow (3-Round Example)

```
                    codex-researcher    gemini-researcher    claude-researcher
                         │                    │                    │
Round 1     ┌────────────┤                    │                    │
(parallel)  │  codex()   │                    │                    │
            │            │   gemini -p        │                    │
            │            │                    │    WebSearch       │
            └──►[R1]─────┼────────────────────┼──────────►         │
                         │    [R1]────────────┼──────────►         │
                         │                    │    [R1]───►        │
                         │                    │                    │
Round 2     ┌────────────┤                    │                    │
(parallel)  │codex-reply │                    │                    │
            │            │  gemini --resume   │                    │
            │            │                    │   own analysis     │
            └──►[R2]─────┼────────────────────┼──────────►         │
                         │    [R2]────────────┼──────────►         │
                         │                    │    [R2]───►        │
                         │                    │                    │
                    (optional peer-to-peer challenges here)        │
                         │                    │                    │
Round 3     ┌────────────┤                    │                    │
(if needed) │codex-reply │                    │                    │
            │            │  gemini --resume   │                    │
            └──►[FINAL]──┼────────────────────┼──────────►         │
                         │   [FINAL]──────────┼──────────►         │
                         │                    │   [FINAL]─►        │
                         │                    │                    │
                    Team Lead collects FINALs → Synthesis
```

## Teammate Constraints

### The "Faithful Proxy" Problem

The biggest risk in this architecture is **sub-agent drift**: a Claude teammate that's supposed to proxy Codex starts arguing from its own reasoning instead of calling `codex-reply`. Over multiple rounds, the teammate "goes native" and the debate loses model diversity.

Mitigations:

1. **Mandatory tool call before SendMessage.** The teammate prompt requires calling the external model before every debate message. This is the primary guardrail.

2. **Verbatim relay for short responses.** If the external model's response is under 200 words, relay it exactly. Only summarize longer responses.

3. **No opinion injection.** The proxy teammate must not add "I also think..." or "Additionally..." content that didn't come from the external model.

4. **Team lead spot-checks.** The team lead can ask a proxy teammate "What did Codex actually say?" at any point to verify fidelity.

### What the Proxy Teammate IS Allowed to Do

- Summarize verbose model output for brevity (but preserve all positions and evidence)
- Format the output for readability (bullet points, structure)
- Ask the external model clarifying follow-up questions before relaying
- Report when the external model fails or refuses, and relay that information honestly

## Failure Handling

| Failure | Impact | Recovery |
|---|---|---|
| Codex MCP unavailable | codex-researcher can't call codex/codex-reply | Fall back to `codex exec` / `codex exec resume` via Bash |
| Codex CLI also unavailable | codex-researcher has no external model | Team lead reassigns: codex-researcher becomes a second Claude researcher, debate continues 2-model (Gemini + Claude) |
| Gemini CLI unavailable | gemini-researcher can't call gemini | Debate continues 2-model (Codex + Claude) |
| Teammate crashes mid-debate | One perspective lost | Team lead continues with remaining teammates. 2-model debate is still valuable. |
| Both external models unavailable | Only Claude remains | Abort team approach, fall back to single-model analysis. No team needed. |
| MCP timeout (Claude Desktop) | codex MCP calls killed at 60s | Detect `CLAUDE_CODE_ENTRYPOINT=claude-desktop` in preflight. Instruct codex-researcher to use CLI resume instead of MCP. |
| Teammate goes native (stops calling external model) | Loss of model diversity | Team lead monitors message tags. If a proxy teammate sends debate messages without a preceding tool call, send a correction: "You must call codex-reply before responding." |

## Cost Analysis

| Component | v1 (Single Orchestrator) | Hybrid Agent Teams |
|---|---|---|
| Claude tokens (orchestration) | ~15-25k (all relay traffic in one context) | ~5-8k per teammate × 3 + ~5k lead = ~25-30k total |
| Codex tokens | Same (determined by Codex, not architecture) | Same |
| Gemini tokens | Same | Same |
| **Total Claude cost** | ~$0.20-0.30 | ~$0.30-0.50 |
| **Effective parallelism** | Limited (background tasks) | Full (independent teammates) |
| **Debate quality** | Good (Claude-mediated) | Better (peer-to-peer, faithful proxy) |

The hybrid costs ~50-70% more in Claude tokens due to teammate overhead, but delivers higher debate quality and true parallelism. For a brainstorming tool where quality of diverse perspectives is the point, this is a reasonable tradeoff.

## Comparison: All Approaches

| Dimension | v1 Direct | Hybrid Agent Teams | Pure Agent Teams (All-Claude) |
|---|---|---|---|
| **Model diversity** | 3 families | 3 families | 1 family |
| **Parallelism** | Partial | Full | Full |
| **Peer-to-peer debate** | No (Claude relays) | Yes (SendMessage) | Yes (SendMessage) |
| **Context pressure** | High (single window) | Distributed | Distributed |
| **Debate fidelity** | Claude paraphrases | Faithful proxy relay | Native (no proxy needed) |
| **Blind spot correlation** | Low (3 model families) | Low (3 model families) | High (same model) |
| **Convergence speed** | Medium | Medium | Fast (but potentially false) |
| **Token cost** | ~$0.20-0.30 | ~$0.30-0.50 | ~$0.15-0.25 |
| **Implementation complexity** | Medium (one big SKILL.md) | High (team + proxy prompts) | Low (standard team pattern) |
| **Latency (3 rounds)** | ~3-5 min | ~3-4 min (parallel advantage) | ~2-3 min |
| **Prerequisite** | Codex + Gemini CLIs | Agent Teams + Codex + Gemini | Agent Teams only |

## Prerequisites

1. **Claude Code v2.1.32+** (Agent Teams support)
2. **`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`** in settings
3. **Codex CLI** on PATH (for MCP server or CLI fallback)
4. **Gemini CLI** on PATH
5. **Codex MCP server** registered in Claude Code config (recommended, not required):
   ```json
   {
     "mcpServers": {
       "codex": {
         "command": "codex",
         "args": ["mcp-server"]
       }
     }
   }
   ```

## Open Questions

1. **Should the team lead participate in the debate or only orchestrate?** Current design has a separate `claude-researcher` teammate. Alternative: the team lead does its own research and participates directly, saving one teammate's token cost. Tradeoff: cleaner separation of concerns vs. lower cost.

2. **How to handle proxy fidelity at scale?** The "faithful proxy" constraint is enforced by prompt instructions. For critical use cases, should we add programmatic verification (e.g., check that the teammate called `codex-reply` before each SendMessage)?

3. **Should teammates communicate via shared task list or direct messages?** Current design uses `SendMessage` for debate. An alternative is to use the shared task list for structured rounds (each round is a task with dependencies) and `SendMessage` for ad-hoc challenges. This would give the team lead better visibility into round completion.

4. **Can we reduce Claude token cost?** Each teammate is a full Claude session. If the proxy teammates are mostly relaying, they're paying Claude-level token costs for what is essentially a protocol adapter. Could a lighter-weight proxy reduce costs? (Current answer: no — Agent Teams teammates must be Claude sessions. The overhead is inherent to the architecture.)

5. **v1 coexistence.** Should the hybrid be a new skill (`/team-brainstorm-v2`) or a flag on the existing skill (`/team-brainstorm --agent-teams`)? A flag keeps the surface area small and lets users opt in without learning a new command. But it makes the SKILL.md even more complex with two code paths.

## Recommendation

Ship this as an **opt-in flag** on the existing `/team-brainstorm` skill:

```
/team-brainstorm --agent-teams "topic here"
```

When `--agent-teams` is passed:
- Verify `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is set
- Use the hybrid Agent Teams architecture described here
- Fall back to v1 (direct orchestration) if Agent Teams is unavailable

When `--agent-teams` is NOT passed:
- Use v1 (current behavior, no change)

This lets us validate the hybrid approach without breaking the existing skill. Once Agent Teams graduates from experimental, we can make it the default.
