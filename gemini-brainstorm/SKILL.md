---
name: gemini-brainstorm
description: Run a Gemini CLI brainstorming session from the command line. Use when you want fast alternative ideas, critiques, and a second perspective on a plan or design — using `gemini -p` (non-interactive headless mode). Pass a plan file path or topic as the argument.
---

# Gemini CLI Brainstorm

Uses the `gemini -p` CLI tool (Google Gemini CLI) to run a non-interactive brainstorming session grounded in the actual codebase.

## When to Use

- As an alternative to `/codex-brainstorm` when you want a Gemini (rather than GPT) perspective
- When you want Gemini to read the real codebase before forming opinions (it will run shell tools autonomously with `--approval-mode yolo`)
- When you have a plan file or topic you want stress-tested by a second model

## Step 1: Build the Prompt

Construct a focused prompt. For a plan file, include:
- What the app does (tech stack, one sentence)
- The three questions to answer:
  1. Any critical gaps still missing?
  2. What is the single fastest win?
  3. One unconventional alternative approach to the hardest problem?

For open-ended brainstorming, ask for: multiple angles, trade-offs, creative alternatives.

## Step 2: Run Gemini in the Background

Use `-o json` to capture structured output including the `session_id` for potential follow-up calls:

```bash
gemini --approval-mode yolo -o json -p "$YOUR_PROMPT_WITH_PLAN_CONTENT_INLINED" 2>/dev/null
```

**Parsing the session ID** from JSON output for follow-up calls:

```bash
GEMINI_SESSION_ID=$(echo "$RESPONSE" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['session_id'])")
GEMINI_RESPONSE=$(echo "$RESPONSE" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['response'])")
```

Save `GEMINI_SESSION_ID` — if follow-up rounds are needed, resume the same session:

```bash
gemini --resume "$GEMINI_SESSION_ID" --approval-mode yolo -o json -p "$FOLLOW_UP_PROMPT" 2>/dev/null
```

Alternatively, for simpler single-shot use where you don't need the session ID:

```bash
cat $PLAN_FILE | gemini --approval-mode yolo -p "$YOUR_PROMPT" 2>&1
```

Key flags:
- `--approval-mode yolo` — fully non-interactive, auto-approves all tool calls (file reads, shell commands). **Must be passed on every call including `--resume` — it is NOT inherited across sessions.** Without it, Gemini hangs waiting for interactive approval.
- `-p` / `--prompt` — appended to stdin input, enabling the plan file to be piped in
- `-o json` — structured JSON output with `session_id` and `response` fields
- `--resume` — resume a prior session by `session_id` for multi-turn conversations
- `-s` / `--sandbox` — optionally add for sandboxed execution
- `-m` / `--model` — optionally specify model (e.g. `-m gemini-2.5-pro`)
- Gemini will autonomously explore the repo before answering — expect shell/file tool calls

Run as a background Bash command (`run_in_background: true`) so the main conversation stays free.

## Step 3: Do Your Own Independent Review

While Gemini runs, do your own analysis. Cover the same questions independently. **Do not read the Gemini output until your own review is complete** — the value is in having two unbiased perspectives to compare.

## Step 4: Read and Compare

Once the background command completes (wait for the task notification), read the output file. Compare against your own review:
- Accept findings grounded in specific file/line references from the codebase
- Override findings that rely on assumptions Gemini couldn't verify
- Pay attention to disagreements between Gemini and your review — those are the most interesting

## Step 5: Update the Plan

For each accepted finding, update the plan file. For each override, note why.

## Notes

- Gemini CLI uses Gemini 2.5 Pro by default (as of 2026-03)
- It will autonomously explore the repo — expect multiple tool calls and 30-120 seconds runtime
- Output goes to a temp file; use `run_in_background: true` on the Bash call
- Unlike Codex, Gemini reads the working directory automatically — no need for `sandbox_permissions` config
- **Minimum version: 0.24.0.** Earlier versions (e.g. 0.1.x from Homebrew) are missing `--approval-mode`, `-o json`, and `--resume`. Check with `gemini --version` before running. Install/update via `npm install -g @google/gemini-cli`
- **Background task race condition:** When Gemini runs as a `run_in_background` Bash task, the output file may appear empty (0 bytes) briefly after the task reports completion — there can be a small delay between process exit and file flush. Always wait for the background task notification before reading output files. If an output file reads as empty, wait 2-3 seconds and retry the read before concluding the model returned nothing.
- **Graceful degradation:** If Gemini fails (timeout, not installed), continue with your own independent review. A single-perspective review is still valuable.
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

## Comparison with /codex-brainstorm

| | `/codex-brainstorm` | `/gemini-brainstorm` |
|---|---|---|
| Model | GPT (via OpenAI Codex CLI) | Gemini 2.5 Pro |
| Non-interactive flag | `--config 'approval_policy="never"'` | `--approval-mode yolo` (must pass every call) |
| Input method | **No stdin** — inline all content in prompt arg | `cat file \| gemini -p "prompt"` (stdin works) |
| Codebase access | `sandbox_permissions=["disk-full-read-access"]` | Automatic (working directory) |
| Session resume | `codex exec resume {THREAD_ID}` (>= 0.35.0) | `gemini --resume {SESSION_ID}` |
| Structured output | `--json` (JSONL) | `-o json` |
| Best for | GPT-family second opinion | Gemini-family second opinion |
