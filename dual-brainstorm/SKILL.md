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

If `claude-desktop`, you are in Claude Desktop which has a **60-second hardcoded MCP tool timeout**. This doesn't affect `codex exec` or `gemini` CLI directly, but is important context — always use CLI paths, never MCP, for these tools in Claude Desktop.

**Check tool availability:**

```bash
which codex && which gemini
```

If only one tool is available, fall back to single-tool + your own review. If neither is available, abort and suggest installing them.

**Gemini minimum version: 0.24.0.** Check with `gemini --version`. Earlier versions (e.g. 0.1.x from Homebrew) are missing `--approval-mode`, `-o json`, and `--resume`. Install/update via `npm install -g @google/gemini-cli`.

**Codex mode detection:** Check if `codex exec resume --help` succeeds. If so, use `--json` output to capture the thread ID for potential follow-up. Otherwise, use stateless mode.

**Always pass `--skip-git-repo-check` to Codex.** Brainstorm tasks are research-only and never modify files. Without this flag, Codex fails in non-git directories.

```bash
GIT_FLAG="--skip-git-repo-check"
```

**Known Codex config pitfall:** If the user's `~/.codex/config.toml` has `web_search = "live"` under `[features]`, `codex exec` will fail with a config parsing error. Work around with `CODEX_HOME=$(mktemp -d)` or ask the user to fix their config. Note: if using `CODEX_HOME` override and getting 401 Unauthorized, the API key lives in the original home — only override specific config keys via `--config` instead.

If a plan file path was provided as the argument, read it now so its content can be embedded in prompts.

## Step 1: Launch Both CLI Tools in Parallel

In a **single message**, fire two background Bash tool calls simultaneously:

**IMPORTANT: Codex does NOT read piped stdin** (its sandbox blocks stdin forwarding). Pass all content directly in the prompt argument. If you have a plan file, read it first and embed its contents in the prompt string. Gemini's `-p` flag does work with stdin, but for consistency, prefer inlining for both.

**Codex** (with JSONL output for structured parsing):

```bash
codex exec $GIT_FLAG \
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
            msg=d.get('item',{}).get('text','')
    except: pass
print(msg)
")
```

**Gemini** (with JSON output for session ID capture):

```bash
gemini --approval-mode yolo -o json -p "$PROMPT_WITH_ALL_CONTENT_INLINED" 2>/dev/null
```

Parse the session ID for potential follow-up:

```bash
GEMINI_SESSION_ID=$(echo "$RESPONSE" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['session_id'])")
```

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
- **Gemini `--approval-mode yolo` is NOT inherited on `--resume`.** It must be passed explicitly on every call or Gemini hangs waiting for interactive approval.
- **Environment detection:** `CLAUDE_CODE_ENTRYPOINT=claude-desktop` means 60s hardcoded MCP timeout — always use CLI paths. Any other value means Claude Code CLI where MCP is also safe.
- Expect 30-120 seconds per tool depending on codebase size and topic complexity.
