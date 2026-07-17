---
name: dual-brainstorm
description: Run Codex and Gemini brainstorming sessions in parallel, then synthesize both perspectives. Pass a plan file path or topic as the argument. Launches both CLI tools simultaneously as background tasks so neither blocks the other.
---

# Dual Brainstorm: Codex + Gemini in Parallel

Runs `/codex-brainstorm` and `/gemini-brainstorm` simultaneously as background Bash commands, does an independent review in parallel, then synthesizes all three perspectives.

## Step 0: Preflight Checks

**Detect the runtime environment:**

```bash
echo "$CLAUDE_CODE_ENTRYPOINT"
```

If `claude-desktop`, you are in Claude Desktop which has a **60-second hardcoded MCP tool timeout**. This doesn't affect `codex exec` or `agy` directly, but is important context — always use CLI paths, never MCP, for these tools in Claude Desktop.

**Check tool availability:**

```bash
which codex && which agy
```

If only one tool is available, fall back to single-tool + your own review. If neither is available, abort and suggest installing them.

**Auto-mode permission (important):** In Claude Code **auto mode**, every Bash command is screened by the auto-mode classifier unless a matching `permissions.allow` rule exists. `codex exec` and `agy` are both agentic CLIs, so the classifier tends to **block them** without an explicit allow-rule — you'll see "Blocked by classifier" and the tool never runs. Add both rules to `.claude/settings.local.json` so each seat bypasses the classifier:

```json
"Bash(codex exec:*)",
"Bash(agy:*)"
```

Note: a stale `Bash(gemini:*)` rule does **not** cover `agy` (different binary). If the `agy` seat is classifier-denied while `codex` succeeds, a missing `Bash(agy:*)` rule is the cause — apply graceful degradation for this run, and tell the user to add the rule so future runs work.

**Gemini is now Antigravity's `agy`.** The old `gemini` CLI is dead for headless use — Google killed its free "Login with Google" auth on **2026-06-18**, so `gemini -p` now fails with `IneligibleTierError … UNSUPPORTED_CLIENT`. `agy` is a native binary (ships with the [Antigravity app](https://antigravity.google) at `~/.local/bin/agy`) that reuses your existing Antigravity login — no API key, no node-version workaround, no version gate. Just confirm it runs with `agy --version`. Pick a Gemini model from `agy models` (this skill uses `"Gemini 3.1 Pro (High)"`).

**Codex mode detection:** Check if `codex exec resume --help` succeeds. If so, use `--json` output to capture the thread ID for potential follow-up. Otherwise, use stateless mode.

**Always pass `--skip-git-repo-check` to Codex.** Brainstorm tasks are research-only and never modify files. Without this flag, Codex fails in non-git directories.

```bash
GIT_FLAG="--skip-git-repo-check"
```

**Known Codex config pitfall:** If the user's `~/.codex/config.toml` has `web_search = "live"` under `[features]`, `codex exec` will fail with a config parsing error. Work around with `CODEX_HOME=$(mktemp -d)` or ask the user to fix their config. Note: if using `CODEX_HOME` override and getting 401 Unauthorized, the API key lives in the original home — only override specific config keys via `--config` instead.

If a plan file path was provided as the argument, read it now so its content can be embedded in prompts.

## Model selection (optional)

Both seats default to today's behavior; override either per run via flags parsed from the skill argument (everything that is not a flag is the topic/plan path). The flags are applied inside each tool's command block in Step 1:

- `--codex-model <id>` → adds `-m <id>` to `codex exec`. Use explicit ids (`gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-5.5`), **not** the bare `gpt-5.6` alias (some codex builds lack metadata for it and reject it). Omit → your `~/.codex/config.toml` default. GPT-5.6 needs a recent codex (verified on `0.144.1`; `0.125.0` is too old).
- `--codex-effort <level>` → adds `-c 'model_reasoning_effort="<level>"'` (`minimal`/`low`/`medium`/`high`/`xhigh`; `gpt-5.6-sol` needs `medium` or higher). Omit → config default.
- `--gemini-model "<name>"` → replaces `agy --model` (**quote it** — names contain spaces; e.g. `"Gemini 3.5 Flash (High)"`). Omit → `"Gemini 3.1 Pro (High)"`.

## Step 1: Launch Both CLI Tools in Parallel

In a **single message**, fire two background Bash tool calls simultaneously:

**IMPORTANT: inline all content in the prompt argument, and redirect stdin from `/dev/null`.** Codex does NOT read piped stdin (its sandbox blocks stdin forwarding), and a backgrounded `agy -p` *hangs forever* waiting on stdin EOF unless you append `</dev/null`. So for both tools: read the plan file first, embed its contents in the prompt string, and end the command with `</dev/null`.

**Codex** (with JSONL output for structured parsing):

```bash
# Optional model overrides (empty = your codex config default):
CODEX_MODEL=""     # from --codex-model  (e.g. gpt-5.6-sol)
CODEX_EFFORT=""    # from --codex-effort (e.g. high)
CODEX_MODEL_ARGS=()
[ -n "$CODEX_MODEL" ]  && CODEX_MODEL_ARGS+=(-m "$CODEX_MODEL")
[ -n "$CODEX_EFFORT" ] && CODEX_MODEL_ARGS+=(-c "model_reasoning_effort=\"$CODEX_EFFORT\"")

codex exec $GIT_FLAG "${CODEX_MODEL_ARGS[@]}" \
  --json \
  --config 'approval_policy="never"' \
  --config 'sandbox_permissions=["disk-full-read-access"]' \
  "$PROMPT_WITH_ALL_CONTENT_INLINED" 2>&1
```

Parse the JSONL output to extract the thread ID and response:

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
            # Only capture the assistant's message text. A real run emits many
            # item.completed events: agent_message (has .text) interleaved with
            # command_execution / reasoning items (empty .text). Blindly taking
            # the last item's .text would return '' whenever Codex's final item
            # is not an agent_message, making a successful run look empty. Take
            # the LAST non-empty agent_message instead.
            if it.get('type')=='agent_message' and it.get('text'):
                msg=it['text']
    except: pass
print(msg)
")
```

**Gemini — via Antigravity `agy`** (the old `gemini` CLI is dead; see Step 0):

```bash
# `--add-dir "$PWD"` lets agy read your codebase (without it, agy runs in an
# empty scratch workspace). `</dev/null` is REQUIRED — a backgrounded `agy -p`
# otherwise hangs forever on stdin EOF. Read-only by default (no
# `--dangerously-skip-permissions` needed for a brainstorm).
GEMINI_MODEL="${GEMINI_MODEL:-Gemini 3.1 Pro (High)}"   # override via --gemini-model "<name>"
agy --add-dir "$PWD" --model "$GEMINI_MODEL" -p "$PROMPT_WITH_ALL_CONTENT_INLINED" </dev/null 2>&1
```

`agy` has no `session_id`/`-o json` output, but dual-brainstorm is single-shot so none is needed (multi-round state lives in `/team-brainstorm`).

Both use `run_in_background: true`. Both get the same prompt so their outputs are directly comparable.

Use the same prompt template for both:
```
You are a staff engineer doing a brainstorming session on this {topic} plan for a {app description}.
Tech stack: {stack}.
Read the plan below and brainstorm:
(1) any critical gaps still missing,
(2) the single fastest win after the highest-priority items are fixed,
(3) one unconventional approach to the hardest problem in the plan.
Be direct and concise. Ground your findings in specific file and line references where possible.
```

## Step 2: Do Your Own Independent Review

While both tools run, do your own analysis covering the same questions. **Do not read either output until your own review is complete.**

## Step 3: Read Both Outputs

Once both background tasks complete (you'll receive two notifications — wait for them before reading), read both output files and compare them across three dimensions:

**Agreements** — findings both tools reached independently. High-confidence; almost certainly worth acting on.

**Unique to Codex only** — validate against the codebase before accepting. May reflect GPT-family reasoning patterns.

**Unique to Gemini only** — validate against the codebase before accepting. May reflect Gemini-family reasoning patterns.

**Disagreements** — Codex and Gemini reached opposite conclusions. These are the most interesting; investigate manually.

## Step 4: Three-Way Synthesis

Compare all three perspectives (Codex, Gemini, your own review):

| Finding | Codex | Gemini | You | Action |
|---------|-------|--------|-----|--------|
| Gap X   | yes   | yes    | yes | Accept — unanimous |
| Gap Y   | yes   | no     | yes | Investigate — split |
| Gap Z   | no    | no     | yes | Keep — your finding |
| Gap W   | yes   | yes    | no  | Review — missed it |

- **3/3 agreement** -> accept without further validation
- **2/3 agreement** -> accept if grounded in a file/line reference
- **1/3** -> requires manual codebase check before accepting
- **Codex != Gemini** -> investigate the disagreement directly

## Step 5: Update the Plan

For each accepted finding, update the plan file. For overrides, note which tool raised it and why you're overriding.

## Notes

- Both tools autonomously read the codebase — expect 10-30 tool calls each, 30-120s per tool
- Fire them in the same message to get true parallelism; don't wait for one before starting the other
- **Graceful degradation:** If one tool fails (config error, timeout, not installed), continue with the other + your own review. A 2-perspective synthesis is still more valuable than a single perspective. Do not abort the entire skill because one tool failed.
- The three-way comparison is more valuable than any single tool — disagreements surface the most interesting gaps
- **Background task race condition:** Output files may appear empty (0 bytes) briefly after a background task reports completion — there can be a small delay between process exit and file flush. Always wait for the background task notification before reading output files. If an output file reads as empty, wait 2-3 seconds and retry the read before concluding the model returned nothing.
- **`agy -p` needs `</dev/null` when backgrounded** or it hangs forever on stdin EOF (same trap as `codex exec`). It also needs `--add-dir "$PWD"` to see your code — without it agy runs in an empty scratch workspace and answers from the prompt alone.
- **Environment detection:** `CLAUDE_CODE_ENTRYPOINT=claude-desktop` means 60s hardcoded MCP timeout — always use CLI paths. Any other value means Claude Code CLI where MCP is also safe.
- Expect 30-120 seconds per tool depending on codebase size and topic complexity.
