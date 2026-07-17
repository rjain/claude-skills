---
name: team-brainstorm
description: "Run a multi-round debate across Claude, Codex, and Gemini on any topic. Each model maintains full conversation memory across rounds. Claude orchestrates, detects convergence, and synthesizes a consensus. Use when the user wants a diverse agent team, multi-model brainstorm, or cross-model debate. Pass a topic or plan file path as the argument."
---

# Team Brainstorm: Stateful Multi-Model Debate

Runs a structured multi-round debate between Claude, Codex (via MCP or resumable CLI), and Gemini (via the Antigravity CLI `agy`). Each model researches independently, critiques the others' findings, and defends or concedes positions across rounds. Claude orchestrates the loop and synthesizes the final consensus.

## Arguments

The argument is a topic string or a path to a plan file.

Parse optional flags from the argument if present:
- `--rounds N` — max round count (default: 3)
- `--consensus strict|majority|any` — convergence mode (default: majority)
  - `strict` — require 3/3 unanimity, flag anything without it
  - `majority` — resolve via 2/3 after max rounds
  - `any` — stop after round 1, no debate (equivalent to `/dual-brainstorm`)
- `--codex-model <id>` / `--codex-effort <level>` / `--gemini-model "<name>"` — per-seat model overrides (see **Model selection** below)

Everything that is not a flag is the topic. Example: `/team-brainstorm --rounds 2 best approach to rate limiting in our API`

## Model selection (optional)

Each model seat can be overridden per run via flags parsed from the argument (alongside `--rounds`/`--consensus`). Substitute these placeholders into **every** Codex/`agy` invocation in the rounds below; when a flag is omitted, its placeholder is empty and behaviour is byte-identical to today:

- `--codex-model <id>` → `{CODEX_M}` = `-m <id>` (else empty → your `~/.codex/config.toml` default). Use **explicit** ids: `gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-5.5` — **not** the bare `gpt-5.6` alias (some codex builds lack metadata for it and reject it). GPT-5.6 needs a recent codex (verified on `0.144.1`; `0.125.0` errors `"requires a newer version of Codex"`). On the **MCP** path, pass the id via the `codex()`/`codex-reply()` `model` parameter instead; for guaranteed model + effort control prefer the CLI-resume path.
- `--codex-effort <level>` → `{CODEX_E}` = `-c 'model_reasoning_effort="<level>"'` (else empty → config default). Levels: `minimal`/`low`/`medium`/`high`/`xhigh`; `gpt-5.6-sol` needs `medium` or higher. (CLI paths only.)
- `--gemini-model "<name>"` → `{GEMINI_MODEL}` (else `Gemini 3.1 Pro (High)`). Any `agy models` entry; **quote it** — names contain spaces. agy bakes the reasoning tier into the name (`(High)`/`(Low)`), so there is no separate effort flag.

## Step 0: Preflight Checks

Before starting the debate, verify tool availability and choose the best Codex path.

**Detect the runtime environment first.** Run this check:

```bash
echo "$CLAUDE_CODE_ENTRYPOINT"
```

If the value is `claude-desktop`, you are running inside Claude Desktop, which hardcodes a **60-second MCP tool timeout** (via `@modelcontextprotocol/sdk` `DEFAULT_REQUEST_TIMEOUT_MSEC = 60000`). This timeout is not user-configurable. **Skip the Codex MCP path entirely** and go straight to CLI resume or stateless fallback — Codex research tasks routinely take 2-5 minutes and will always time out at 60s.

If the value is empty, `cli`, or anything else, you are in Claude Code CLI, which has a much longer timeout (~27 hours default or `MCP_TOOL_TIMEOUT` env var). MCP is safe to use.

Now choose the Codex path in this order:

**Codex MCP (preferred — CLI environments only):** Check if the `codex` and `codex-reply` MCP tools are available **and** `CLAUDE_CODE_ENTRYPOINT` is NOT `claude-desktop`. If both conditions are met, Codex gets full statefulness across rounds via `threadId`. Note: Codex calls can take 30-120 seconds. The MCP tool timeout must be set high enough (`MCP_TOOL_TIMEOUT=300000` in settings.json `env`) or calls will be killed at the default 60s limit.

**Codex CLI resume (preferred for Claude Desktop, fallback elsewhere):** If MCP is skipped or unavailable, check whether `codex exec resume --help` succeeds. If it does, use `codex exec` for Round 1 and `codex exec resume {THREAD_ID}` for later rounds. This preserves Codex's own prior context without relying on MCP timeouts. This is the **recommended path for Claude Desktop**.

**Codex stateless fallback:** If `codex exec resume --help` fails, fall back to plain `codex exec` via Bash. This is the compatibility path for older Codex versions. In this mode, Codex loses cross-round memory and each round must inline summaries of prior positions.

**Version caveat:** Codex resume support first appeared in `0.35.0`. Treat any older version as stateless. The `codex exec resume --last <prompt>` convenience behavior was improved in `0.62.0`; to maximize compatibility, prefer resuming by explicit session id instead of `--last`.

Always pass `--skip-git-repo-check` to Codex. Brainstorm tasks are research-only and never modify files, so git repo context is unnecessary. Without this flag, Codex fails immediately when run from a non-git directory (common when the plan file or topic references a project outside a repo). Set:

```bash
GIT_FLAG="--skip-git-repo-check"
```

**Gemini (via Antigravity `agy`):** The old `gemini` CLI is dead for this skill — Google killed its free "Login with Google" auth on 2026-06-18, so headless `gemini -p` now fails with `IneligibleTierError … UNSUPPORTED_CLIENT`. Use **Antigravity's `agy`** instead: a native binary at `~/.local/bin/agy` that reuses your existing Antigravity login (no API key, no node-version workaround). There is no version gate to enforce — just confirm it runs:

```bash
command -v agy >/dev/null && agy --version || echo "agy not found"
```

If `agy` is missing, it ships with the Antigravity app; run `agy install` to configure paths, or call it at `~/.local/bin/agy`. Pick a Gemini model from `agy models` — this skill uses `"Gemini 3.1 Pro (High)"` to keep the seat a "Gemini" perspective. `agy -p` is **read-only by default** (no `--dangerously-skip-permissions` needed for research rounds) and needs `--add-dir "$PWD"` to see your code.

**Auto-mode permission (important):** In Claude Code **auto mode**, every Bash command is screened by the auto-mode classifier unless a matching `permissions.allow` rule exists. `codex exec` and `agy` are both agentic CLIs, so the classifier tends to **block them** without an explicit allow-rule ("Blocked by classifier", tool never runs). Add both to `.claude/settings.local.json` — `"Bash(codex exec:*)"` and `"Bash(agy:*)"` — so each seat bypasses the classifier. A stale `Bash(gemini:*)` rule does **not** cover `agy` (different binary); if the Gemini seat is classifier-denied while Codex succeeds, a missing `Bash(agy:*)` rule is the cause.

**Parser dependency:** the Codex CLI-path JSONL parser needs `python3` on PATH (`which python3`). If it's missing, extract the response with `jq` instead: `jq -Rrn '[inputs|fromjson?|select(.type=="item.completed" and .item.type=="agent_message" and (.item.text//"")!="")|.item.text]|last' "$CODEX_OUT"`. The `-Rrn '[inputs|fromjson?|...]'` form is deliberate — Codex prepends a non-JSON `Reading additional input from stdin...` line, so a plain `jq -s` slurp errors out; `fromjson?` skips unparseable lines. (Not needed on the Codex MCP path, which returns text directly.)

Unlike `gemini`, `agy` has **no `-o json` / `session_id`** output. It keeps state per *conversation*, stored on disk as `~/.gemini/antigravity-cli/conversations/<UUID>.db`. For statefulness across rounds, capture that `<UUID>` after Round 1 by diffing the conversations dir before/after the call, then resume later rounds with `agy --conversation "<UUID>"`. **Verified:** `--conversation <id>` resumes that *specific* conversation even when it is not the most recent — safe for this interleaved debate. **Caveat:** if you pass an unknown or typo'd id, `agy` prints `Warning: conversation "..." not found` and **silently falls back to the most-recent conversation** — so never pre-mint your own id (`agy` ignores it and assigns its own), and treat a "not found" warning as a failed resume. This same fallback is why bare `-c` / `--continue` ("most recent") is too fragile to rely on here.

If a plan file path was provided as the argument, read the file now so its content can be embedded in prompts.

**Known config pitfall:** If the user's `~/.codex/config.toml` has `web_search = "live"` under `[features]`, `codex exec` will fail with `Error loading config.toml: invalid type: string "live"`. Work around this by setting `CODEX_HOME` to a temp dir with a clean config, or ask the user to move `web_search` to the top level of their config (outside `[features]`).

**Important:** Never pass `--ephemeral` to `codex exec` when using resume mode — it prevents session persistence and breaks `codex exec resume`.

**Background-task stdin trap (must-fix):** `codex exec` reads its prompt from stdin in addition to the positional argument. When invoked as a background task in Claude Code's Bash tool (`run_in_background: true`), stdin is connected to the parent shell rather than a TTY, and Codex hangs forever printing `Reading additional input from stdin...` while it waits for EOF. **`agy -p` has the identical trap** (verified: a backgrounded `agy -p` without `</dev/null` hangs at the prompt indefinitely and never returns, producing a 0-byte output file). **Always redirect stdin from `/dev/null`** in every `codex exec`, `codex exec resume`, **and `agy -p`** invocation. Failure to do this manifests as a 0-byte output file with no completion notification. Apply to every Codex **and `agy`** command in this skill — Round 1, Round 2, Round 3, sanity probes, the lot.

```bash
# WRONG — hangs in background
codex exec ... "prompt" > out.jsonl

# RIGHT — closes stdin so codex uses the positional arg only
codex exec ... "prompt" </dev/null > out.jsonl
```

Create a small in-memory session record for this debate:
- `entrypoint`: value of `CLAUDE_CODE_ENTRYPOINT` (`claude-desktop` | `cli` | empty)
- `codex_mode`: `mcp` | `cli-resume` | `cli-stateless`
- `codex_thread_id` (from JSONL `thread.started` event or MCP `threadId` return)
- `gemini_conversation_id` (the `<UUID>` of the `.db` agy created in Round 1; resume later rounds with `agy --conversation "$gemini_conversation_id"`)
- `codex_model` / `codex_effort` / `gemini_model` (from the model-override flags; empty = tool defaults)
- `topic`
- per-round outputs and summaries

Report which tools are available and which Codex mode you chose. If both Codex and Gemini are missing, abort and suggest installing them.

## Step 1: Round 1 — Independent Research

All three models research the topic independently. Launch Codex and Gemini in parallel, then do your own research.

### Codex (via MCP — preferred)

Call the `codex` MCP tool. You **must** pass `sandbox` and `approval-policy` — without them Codex defaults to read-only sandbox and interactive approval (which stalls in MCP mode):

```
codex({
  prompt: "You are a research agent in a multi-model debate. Your task: {TOPIC}

Research thoroughly. Provide specific findings, evidence, and reasoning.
Be opinionated — take clear positions you're willing to defend.
Keep your response under 500 words.

{PLAN_CONTENT if applicable}",
  sandbox: "workspace-write",
  approval-policy: "never"
})
```

Save the returned `threadId` for subsequent rounds.

### Codex (via Bash — fallback)

If using CLI fallback, always prefer JSONL output so you can capture the session id and keep machine-readable logs.

#### CLI resume mode: Round 1

```bash
codex exec $GIT_FLAG {CODEX_M} {CODEX_E} \
  --json \
  --config 'approval_policy="never"' \
  --config 'sandbox_permissions=["disk-full-read-access"]' \
  "You are a research agent in a multi-model debate. Your task: {TOPIC}

Research thoroughly. Provide specific findings, evidence, and reasoning.
Be opinionated — take clear positions you're willing to defend.
Keep your response under 500 words.

{PLAN_CONTENT if applicable}" </dev/null 2>&1
```

Parse the JSONL output to extract the thread ID and final response. The output format is:

```jsonl
{"type":"thread.started","thread_id":"019d30d6-599c-78b3-a6f9-5cb4e232f4d9"}
{"type":"turn.started"}
{"type":"item.completed","item":{"id":"item_0","type":"agent_message","text":"..."}}
{"type":"turn.completed","usage":{"input_tokens":...,"output_tokens":...}}
```

Use this helper to extract both values:

```bash
CODEX_THREAD_ID=$(echo "$OUTPUT" | python3 -c "
import sys,json
for line in sys.stdin:
    line=line.strip()
    if not line: continue
    try:
        d=json.loads(line)
        if d.get('type')=='thread.started':
            print(d['thread_id']); break
    except: pass
")

CODEX_RESPONSE=$(echo "$OUTPUT" | python3 -c "
import sys,json
msg=''
for line in sys.stdin:
    line=line.strip()
    if not line: continue
    try:
        d=json.loads(line)
        if d.get('type')=='item.completed':
            it=d.get('item',{})
            # A real run emits many item.completed events: agent_message (has
            # .text) interleaved with command_execution / reasoning items (empty
            # .text). Blindly taking the last item's .text returns '' whenever
            # Codex's final item is not an agent_message, making a successful run
            # look empty. Capture the LAST non-empty agent_message instead.
            if it.get('type')=='agent_message' and it.get('text'):
                msg=it['text']
    except: pass
print(msg)
")
```

Save `CODEX_THREAD_ID` for subsequent rounds and `CODEX_RESPONSE` as Round 1 findings.

**Alternative:** Use `--output-last-message /tmp/codex-r1.txt` instead of parsing JSONL if you only need the final text. You still need `--json` piped output to get the `thread_id`.

#### Stateless compatibility mode: Round 1

If CLI resume is unavailable, use the same `codex exec` command shape above (with or without `--json`) but treat it as one-shot only. Save the final message as Round 1 findings; there is no reusable thread id in this mode.

### Gemini (via Antigravity `agy`)

Run in background via Bash. `agy` has no `session_id` to parse, so capture the **conversation id** by diffing the on-disk conversation store around the call: a fresh `-p` run creates exactly one new `~/.gemini/antigravity-cli/conversations/<UUID>.db`, and that `<UUID>` is what you resume with later.

```bash
CONV_DIR=~/.gemini/antigravity-cli/conversations
BEFORE=$(mktemp); ls "$CONV_DIR"/*.db 2>/dev/null | sort > "$BEFORE"

agy --add-dir "$PWD" --model "{GEMINI_MODEL}" -p "You are a research agent in a multi-model debate. Your task: {TOPIC}

Research thoroughly. Provide specific findings, evidence, and reasoning.
Be opinionated — take clear positions you're willing to defend.
Keep your response under 500 words.

{PLAN_CONTENT if applicable}" </dev/null 2>&1

# Capture the conversation id agy just created (the single new .db file).
GEMINI_CONVERSATION_ID=$(comm -13 "$BEFORE" <(ls "$CONV_DIR"/*.db 2>/dev/null | sort) | head -1 | xargs basename 2>/dev/null | sed 's/\.db$//')
echo "GEMINI_CONVERSATION_ID=$GEMINI_CONVERSATION_ID"
```

- `</dev/null` is **required** (background stdin trap — see Step 0) or the call hangs forever.
- `--add-dir "$PWD"` is **required** for codebase grounding; without it `agy` runs in an empty scratch workspace and reports your project as empty.
- Read-only by default — do **not** add `--dangerously-skip-permissions` for research rounds.
- The model's findings are the command's stdout (plain text, no JSON wrapper). Save both the response and `GEMINI_CONVERSATION_ID` (printed on the last line) for subsequent rounds. If the diff yields an empty id, fall back to the newest db: `basename "$(ls -t "$CONV_DIR"/*.db | head -1)" .db`.

### Claude (your own research)

While both tools run, do your own independent research on the same topic using WebSearch, codebase inspection, or any other tools you have. Cover the same ground. **Do not read Codex or Gemini output until your own research is complete.**

### After Round 1

Read both outputs. If `--consensus any` was specified, skip directly to synthesis (Step 4).

Summarize each model's position into a compact form (key findings, positions taken, evidence cited). You'll inject these summaries into Round 2 prompts.

Check for early convergence: if all three models agree on all major points already, skip to synthesis.

## Step 2: Round 2 — Cross-Critique

Each model sees the others' positions and responds. They challenge, concede, or strengthen their arguments.

### Codex (via MCP)

```
codex-reply({
  threadId: "{THREAD_ID from Round 1}",
  prompt: "Round 2 of the debate. Here are the other agents' positions:

GEMINI's findings:
{Gemini Round 1 summary}

CLAUDE's findings:
{Claude Round 1 summary}

Review their positions. For each:
1. If you agree, say so briefly and why
2. If you disagree, challenge with specific evidence
3. Revise your own position if their evidence is compelling
State your updated top findings. Under 500 words."
})
```

### Codex (via Bash — fallback)

If `codex_mode = cli-resume`, continue the same Codex session:

```bash
codex exec resume $GIT_FLAG {CODEX_M} {CODEX_E} \
  --json \
  "{CODEX_THREAD_ID}" \
  "Round 2 of the debate. Here are the other agents' positions:

GEMINI's findings:
{Gemini Round 1 summary}

CLAUDE's findings:
{Claude Round 1 summary}

Review their positions. For each:
1. If you agree, say so briefly and why
2. If you disagree, challenge with specific evidence
3. Revise your own position if their evidence is compelling
State your updated top findings. Under 500 words." </dev/null 2>&1
```

Parse the JSONL output with the same helper as Round 1. The `thread_id` will be the same across rounds — Codex remembers its full prior conversation.

If `codex_mode = cli-stateless`, use `codex exec` and inline Codex's own prior summary in the prompt: `In Round 1, you argued: {Codex Round 1 summary}. Now respond to...`

### Gemini (via Antigravity `agy`)

Resume the same conversation with `--conversation "{GEMINI_CONVERSATION_ID}"` (the id captured in Round 1). Resuming reuses the existing `.db` — do **not** re-capture an id this round.

```bash
agy --conversation "{GEMINI_CONVERSATION_ID}" --add-dir "$PWD" --model "{GEMINI_MODEL}" -p "Round 2 of the debate. Here are the other agents' positions:

CODEX's findings:
{Codex Round 1 summary}

CLAUDE's findings:
{Claude Round 1 summary}

Review their positions. For each:
1. If you agree, say so briefly and why
2. If you disagree, challenge with specific evidence
3. Revise your own position if their evidence is compelling
State your updated top findings. Under 500 words." </dev/null 2>&1
```

If the output begins with `Warning: conversation "..." not found`, the id was wrong/stale and `agy` silently fell back to the most-recent conversation — re-capture the id from Round 1 and retry rather than trusting that response.

### Claude

Review all Round 2 responses. Update your own position based on new evidence. Identify:
- **Agreements**: positions where 2+ models now align
- **Remaining disputes**: positions where models still disagree
- **Concessions**: positions any model changed from Round 1

If all three models now agree on everything, skip to synthesis. If max rounds is 2, go to synthesis.

## Step 3: Round 3 — Final Rebuttal (if needed)

Only run if there are remaining disputes after Round 2 and `--rounds` allows it.

### Codex (via MCP)

```
codex-reply({
  threadId: "{THREAD_ID}",
  prompt: "Final round. Remaining disputes:
{List of disagreements from Round 2}

Gemini's latest position: {Gemini Round 2 summary}
Claude's latest position: {Claude Round 2 summary}

Defend or concede on each disputed point. State your FINAL position on each.
Be decisive — no hedging. Under 300 words."
})
```

### Codex (via Bash — fallback)

If `codex_mode = cli-resume`, continue with `codex exec resume $GIT_FLAG {CODEX_M} {CODEX_E} "{CODEX_THREAD_ID}" ...` using the same Round 3 prompt.

If `codex_mode = cli-stateless`, use `codex exec` and inline the full debate history summary.

### Gemini (via Antigravity `agy`)

```bash
agy --conversation "{GEMINI_CONVERSATION_ID}" --add-dir "$PWD" --model "{GEMINI_MODEL}" -p "Final round. Remaining disputes:
{List of disagreements from Round 2}

Codex's latest position: {Codex Round 2 summary}
Claude's latest position: {Claude Round 2 summary}

Defend or concede on each disputed point. State your FINAL position on each.
Be decisive — no hedging. Under 300 words." </dev/null 2>&1
```

### Claude

Form your final position on each remaining dispute.

## Step 4: Synthesis

Build a consensus table showing each finding, each model's position, and how it was resolved:

```
## Consensus

| # | Finding | Codex | Gemini | Claude | Resolution |
|---|---------|-------|--------|--------|------------|
| 1 | {desc}  | ✓     | ✓      | ✓      | Unanimous (round 1) |
| 2 | {desc}  | ✓     | ✗→✓   | ✓      | Unanimous (round 2, Gemini conceded) |
| 3 | {desc}  | ✗     | ✓      | ✓      | Majority accept (2/3) |
| 4 | {desc}  | ✓     | ✓      | ✗      | Majority accept (2/3) |
| 5 | {desc}  | ✓     | ✗      | ✗      | Majority reject (2/3) |
```

Resolution rules (based on `--consensus` mode):
- **majority** (default): 2/3 agreement resolves the item. Items with no majority are flagged for user review.
- **strict**: only 3/3 unanimous items are accepted. Everything else is flagged.
- **any**: no resolution logic — just present all three perspectives side by side.

## Step 5: Present Results

Present to the user:

1. **Consensus table** (from Step 4)
2. **Debate narrative** — a brief summary of who argued what, who conceded, and what remained disputed. Keep this to 3-5 sentences.
3. **Accepted findings** — the resolved list of findings/recommendations based on the consensus mode.
4. **Flagged for review** — any items that didn't meet the consensus threshold, with each model's reasoning.

If a plan file was provided, offer to update it with the accepted findings.

## Notes

- Codex can maintain full conversation memory in two ways: MCP (`threadId`) or CLI resume (`codex exec` + `codex exec resume {THREAD_ID}`). Gemini (via `agy`) maintains memory via `agy --conversation {GEMINI_CONVERSATION_ID}`, where the id is the `<UUID>` of the conversation `.db` agy created in Round 1.
- **Environment detection:** `CLAUDE_CODE_ENTRYPOINT=claude-desktop` means you're in Claude Desktop (60s hardcoded MCP timeout — skip MCP). Any other value means Claude Code CLI (MCP is safe). This env var is always set by the host process.
- Prefer Codex MCP only in Claude Code CLI where tool timeouts are generous. In Claude Desktop, always use CLI resume — MCP will time out on any non-trivial Codex task. Only use stateless `codex exec` on older Codex installs that do not support resume.
- Runtime capability detection is more reliable than version checks alone. Use `codex exec resume --help` to decide whether CLI resume is supported.
- Codex resume support first appeared in `0.35.0`. Versions older than that should be treated as stateless. For broad compatibility, resume by explicit thread id rather than relying on `--last`.
- **Codex JSONL format:** `--json` output emits one JSON object per line. The key events are `thread.started` (contains `thread_id`), `item.completed` (contains `item.text`), and `turn.completed` (contains `usage` with token counts). The `thread_id` is a UUID that stays the same across resume calls.
- **Alternative to JSONL parsing:** Pass `--output-last-message /path/to/file.txt` to have Codex write only the final assistant message to a file. Combine with `--json` piped to a variable to get both the `thread_id` (from JSONL) and clean response text (from the file).
- If one model fails mid-debate, continue with the remaining two. A 2-model debate is still more valuable than a single perspective.
- Keep prompts to external models under 500 words per round to manage context costs and avoid token limits.
- `agy` has no `-o json` / `session_id`. Capture the conversation id by diffing `~/.gemini/antigravity-cli/conversations/*.db` before/after the Round 1 call (the one new `<UUID>.db`); subsequent rounds resume with `agy --conversation {GEMINI_CONVERSATION_ID}`. Verified that `--conversation <id>` resumes that specific conversation even when it is not the most recent. `agy` reuses your existing Antigravity login (no API key); if it can't authenticate, open the Antigravity app once to refresh the session, then retry. It needs `--add-dir "$PWD"` to read your project — it does not auto-read the shell's cwd.
- When using Codex CLI fallback, prefer `--json` output so you can capture the thread id and preserve structured logs. Persist per-round outputs and ids in a small session record so retries and later synthesis are easier.
- **`agy -p` is read-only by default** — research rounds need no approval-bypass flag, so do not pass `--dangerously-skip-permissions` (this replaces the old `gemini --approval-mode yolo`). Every `agy -p` call needs `</dev/null` (background stdin trap) and `--add-dir "$PWD"` (codebase grounding) on every round.
- **`agy --conversation` silently falls back to the most-recent conversation if the id is unknown**, printing `Warning: conversation "..." not found`. Never pre-mint your own UUID — `agy` ignores it and assigns its own. Always capture the real id from Round 1, and treat a "not found" warning as a failed resume (re-capture and retry).
- **Config pitfalls:** If `codex exec` fails with a config.toml parsing error (e.g., `web_search = "live"` under `[features]`), work around it with `CODEX_HOME=$(mktemp -d)` and a minimal config, or ask the user to fix their config. If `codex exec` fails with 401 Unauthorized using `CODEX_HOME` override, the API key is stored in the original home — use the original `CODEX_HOME` and only override specific config keys via `--config`.
- Expect ~2-4 minutes for a 3-round debate depending on model response times.
- **Background task output race condition:** When Codex and Gemini run as `run_in_background` Bash tasks, the output file may appear empty (0 bytes) briefly after the task reports completion — there can be a small delay between process exit and file flush. **Always wait for the background task notification before reading output files.** If an output file reads as empty, wait 2-3 seconds and retry the read before concluding the model returned nothing. Do NOT use the Read tool to poll — wait for the `<task-notification>` that confirms completion.
