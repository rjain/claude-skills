---
name: codex-brainstorm
description: Run a Codex CLI brainstorming session from the command line. Use when you want fast alternative ideas, critiques, and a second perspective on a plan or design — using `codex exec` (no MCP server required). Pass a plan file path or topic as the argument.
---

# Codex CLI Brainstorm

Uses the `codex exec` CLI tool (OpenAI Codex CLI, not the MCP server) to run a non-interactive brainstorming session grounded in the actual codebase.

## When to Use

- When the `mcp__validate-plans-and-brainstorm-ideas__codex` MCP server is unavailable
- When you want Codex to read the real codebase before forming opinions (it will run `rg`, `cat`, `sed` etc. autonomously)
- When you have a plan file or topic you want stress-tested by a second model

## Step 0: Preflight Checks

**Detect the runtime environment first:**

```bash
echo "$CLAUDE_CODE_ENTRYPOINT"
```

If `claude-desktop`, you are in Claude Desktop which has a **60-second hardcoded MCP tool timeout**. This doesn't affect `codex exec` CLI directly, but is important context if you're considering MCP alternatives. CLI exec is the reliable path in all environments.

**Choose the Codex execution mode:**

1. **CLI resume (preferred):** Check if `codex exec resume --help` succeeds. If so, use `codex exec` for the initial call and `codex exec resume {THREAD_ID}` for any follow-ups. This preserves conversation context across calls.
2. **CLI stateless (fallback):** If resume is unavailable (Codex < 0.35.0), use plain `codex exec`. Each call is isolated.

Report which mode you chose.

**Always pass `--skip-git-repo-check`:** Brainstorm tasks are research-only and never modify files, so git repo context is unnecessary. Without this flag, Codex fails immediately in non-git directories.

```bash
GIT_FLAG="--skip-git-repo-check"
```

**Auto-mode permission:** In Claude Code **auto mode**, every Bash command is screened by the auto-mode classifier unless a matching `permissions.allow` rule exists. `codex exec` is an agentic CLI, so the classifier tends to **block it** ("Blocked by classifier", tool never runs) without an explicit allow-rule. Add `"Bash(codex exec:*)"` to `.claude/settings.local.json` so it bypasses the classifier.

**Parser dependency:** the JSONL response parser below needs `python3` on PATH (`which python3`). If it's missing, extract the response with `jq` instead: `jq -Rrn '[inputs|fromjson?|select(.type=="item.completed" and .item.type=="agent_message" and (.item.text//"")!="")|.item.text]|last' "$CODEX_OUT"`. The `-Rrn '[inputs|fromjson?|...]'` form is deliberate — Codex prepends a non-JSON `Reading additional input from stdin...` line, so a plain `jq -s` slurp errors out; `fromjson?` skips unparseable lines.

**Known config pitfall:** If the user's `~/.codex/config.toml` has `web_search = "live"` under `[features]`, `codex exec` will fail with `Error loading config.toml: invalid type: string "live"`. Work around this by setting `CODEX_HOME` to a temp dir with a clean config, or ask the user to move `web_search` to the top level of their config (outside `[features]`). Note: if `codex exec` fails with 401 Unauthorized using `CODEX_HOME` override, the API key is stored in the original home — use the original `CODEX_HOME` and only override specific config keys via `--config`.

## Step 1: Build the Prompt

Construct a focused prompt. For a plan file, include:
- What the app does (tech stack, one sentence)
- The three questions to answer:
  1. Any critical gaps still missing?
  2. What is the single fastest win?
  3. One unconventional alternative approach?

For open-ended brainstorming, ask for: multiple angles, trade-offs, creative alternatives.

## Model selection (optional)

By default Codex uses whatever model your `~/.codex/config.toml` specifies. To override for a single run, parse these flags from the skill's argument (everything that is not a flag is the topic/plan path):

- `--codex-model <id>` → adds `-m <id>` to `codex exec`. Use an **explicit** model id, e.g. `gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, or `gpt-5.5`. Avoid the bare `gpt-5.6` alias — some codex builds lack local metadata for it and reject it on ChatGPT-account auth.
- `--codex-effort <level>` → adds `-c 'model_reasoning_effort="<level>"'` (`minimal`/`low`/`medium`/`high`/`xhigh`; newer models add `max`). Note `gpt-5.6-sol` requires `medium` or higher.

Omit either flag to keep your config default. GPT-5.6 tiers need a recent codex — verified working on `0.144.1`; `0.125.0` is too old (fails with `"requires a newer version of Codex"`).

## Step 2: Run Codex in the Background

**IMPORTANT: Codex does NOT read piped stdin.** Its sandbox blocks stdin forwarding, so `cat file | codex exec` silently loses the input. Instead, pass all content directly in the prompt argument. If you have a plan file, read it first and embed its contents in the prompt string.

**Never pass `--ephemeral`** when using resume mode — it prevents session persistence.

Prefer JSONL output (`--json`) so you can capture the session id and structured logs:

```bash
# Optional per-run model controls (empty = your codex config defaults):
CODEX_MODEL=""     # e.g. gpt-5.6-sol   (from --codex-model)
CODEX_EFFORT=""    # e.g. high          (from --codex-effort)
MODEL_ARGS=()
[ -n "$CODEX_MODEL" ]  && MODEL_ARGS+=(-m "$CODEX_MODEL")
[ -n "$CODEX_EFFORT" ] && MODEL_ARGS+=(-c "model_reasoning_effort=\"$CODEX_EFFORT\"")

codex exec $GIT_FLAG "${MODEL_ARGS[@]}" \
  --json \
  --config 'approval_policy="never"' \
  --config 'sandbox_permissions=["disk-full-read-access"]' \
  "$YOUR_PROMPT_WITH_PLAN_CONTENT_INLINED" 2>&1
```

**Parsing JSONL output:** The output emits one JSON object per line. Key events are `thread.started` (contains `thread_id`), `item.completed` (contains `item.text`), and `turn.completed` (contains token usage).

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

Save `CODEX_THREAD_ID` for follow-up calls (if using resume mode) and `CODEX_RESPONSE` as the brainstorm findings.

**Alternative:** Use `--output-last-message /tmp/codex-output.txt` instead of parsing JSONL if you only need the final text. You still need `--json` piped output to get the `thread_id`.

Key flags:
- `--skip-git-repo-check` — always pass for brainstorm tasks
- `approval_policy="never"` — fully non-interactive, no approval prompts
- `sandbox_permissions=["disk-full-read-access"]` — lets Codex read the codebase freely
- `--json` — JSONL output for structured parsing and session resumption
- Codex will autonomously grep/read the repo to ground its feedback in real code

Run as a background Bash command so the main conversation stays free.

## Step 3: Do Your Own Independent Review

While Codex runs, do your own analysis. Cover the same questions independently. **Do not read the Codex output until your own review is complete** — the value is in having two unbiased perspectives to compare.

## Step 4: Read and Compare

Once the background command completes (wait for the task notification), read the output file. Compare against your own review:
- Accept findings that are grounded in specific file/line references
- Override findings that rely on assumptions Codex couldn't verify
- Flag any finding where Codex and your review disagree — those are the most interesting

## Step 5: Update the Plan

For each accepted finding, update the plan file. For each override, note why.

## Notes

- With no `--codex-model`/`--codex-effort`, Codex uses your `~/.codex/config.toml` default (model + `model_reasoning_effort`). Use explicit GPT-5.6 tier ids (`gpt-5.6-sol`/`-terra`/`-luna`), not the bare `gpt-5.6` alias
- It will autonomously explore the repo — expect 10-30 tool calls and 30-120 seconds runtime
- Output goes to a temp file; use `run_in_background: true` on the Bash call so the main context isn't blocked
- If `codex` is not on PATH: `which codex` — it installs via npm as `codex-cli`
- **Background task race condition:** When Codex runs as a `run_in_background` Bash task, the output file may appear empty (0 bytes) briefly after the task reports completion — there can be a small delay between process exit and file flush. Always wait for the background task notification before reading output files. If an output file reads as empty, wait 2-3 seconds and retry the read before concluding the model returned nothing.
- **Graceful degradation:** If Codex fails (config error, timeout, not installed), continue with your own independent review. A single-perspective review is still valuable.
- Expect 30-120 seconds runtime depending on codebase size and topic complexity.

## Example Prompt Template (Plan Review)

```
You are a staff engineer doing a brainstorming session on this {topic} plan for a {app description}.
Tech stack: {stack}.
Read the plan below and brainstorm:
(1) any critical gaps still missing,
(2) the single fastest win after the highest-priority items are fixed,
(3) one unconventional approach to the hardest problem in the plan.
Be direct and concise. Ground your findings in specific file and line references where possible.
```
