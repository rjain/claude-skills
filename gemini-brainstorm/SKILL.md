---
name: gemini-brainstorm
description: Run a Gemini-model brainstorming session from the command line via the Antigravity CLI (`agy -p`, non-interactive headless mode). Use when you want fast alternative ideas, critiques, and a second perspective on a plan or design. Pass a plan file path or topic as the argument.
---

# Gemini Brainstorm (via Antigravity `agy`)

Uses Google's **Antigravity CLI** (`agy --print`) to run a non-interactive brainstorming session with a Gemini model, grounded in the actual codebase.

> **Why `agy` and not `gemini`:** As of **2026-06-18**, Google discontinued the free "Login with Google" auth for Gemini Code Assist individuals / AI Pro / AI Ultra on both the IDE extensions and **gemini-cli**. Headless `gemini -p` now fails during auth with `IneligibleTierError … UNSUPPORTED_CLIENT … "Gemini Code Assist for individuals"`. Google's official migration target is the **Antigravity** suite, whose CLI is `agy`. `agy` is a native binary that reuses your existing Antigravity login (no API key, no node-version workaround), and its `--print` mode is a drop-in for headless prompting. (If you specifically need gemini-cli, the only remaining path is a Gemini **API key** from https://aistudio.google.com/apikey with `selectedAuthType: "gemini-api-key"` — but `agy` is simpler here.)

## When to Use

- As an alternative to `/codex-brainstorm` when you want a Gemini (rather than GPT) perspective
- When you want the model to read the real codebase before forming opinions
- When you have a plan file or topic you want stress-tested by a second model

## Step 1: Build the Prompt

Construct a focused prompt. For a plan file, include:
- What the app does (tech stack, one sentence)
- The three questions to answer:
  1. Any critical gaps still missing?
  2. What is the single fastest win?
  3. One unconventional alternative approach to the hardest problem?

For open-ended brainstorming, ask for: multiple angles, trade-offs, creative alternatives.

## Step 2: Run agy in the Background

```bash
# `agy` is a native binary — NO node-version / PATH workaround needed (unlike
# the old gemini-cli). It reuses your existing Antigravity login (no API key).
# `</dev/null` is REQUIRED: a backgrounded `agy -p` otherwise hangs forever
# waiting on stdin EOF (same trap as `codex exec`).
agy --add-dir "$PWD" --model "Gemini 3.1 Pro (High)" -p "$YOUR_PROMPT

THE PLAN:
$(cat $PLAN_FILE)" </dev/null 2>&1
```

Why inline the plan via `$(cat ...)` rather than piping via stdin: the inline form makes the constructed prompt visible in shell history for debugging and avoids stdin-handling quirks.

Key flags:
- `-p` / `--print` / `--prompt` — run a single prompt non-interactively and print the response (this is the headless mode)
- `--add-dir "$PWD"` — **required for codebase grounding.** Without it, `agy` runs in its own empty scratch workspace (`~/.gemini/antigravity-cli/scratch`) and reports your project as "empty." Add the directory you want it to read.
- `--model` — pick the model. Run `agy models` to list them; `"Gemini 3.1 Pro (High)"` is the Gemini-family pick. Others include `"Gemini 3.5 Flash (High)"`, `"Claude Opus 4.6 (Thinking)"`, `"GPT-OSS 120B (Medium)"`.
- **Read-only by default:** do NOT pass `--dangerously-skip-permissions`. In headless `-p` mode `agy` auto-approves only safe read tools (file reads, listings); edits/shell would need an approval it can't get non-interactively, so the session stays effectively read-only — matching gemini's old `--approval-mode plan`.
- `--print-timeout` — max wait for the print response (default `5m`). Bump it for long brainstorms.

Run as a background Bash command (`run_in_background: true`) so the main conversation stays free.

### When you genuinely need writes or shell execution

If your brainstorm needs `agy` to edit files or run shell commands, add `--dangerously-skip-permissions` (auto-approves ALL tool requests — the analog of gemini's old `yolo`), or `--sandbox` to run with terminal restrictions enabled. The broad-permission mode may trigger your harness's safety classifier; if so, whitelist the exact command in `.claude/settings.json` rather than disabling protections globally.

### Troubleshooting

**`IneligibleTierError … UNSUPPORTED_CLIENT` (this is from `gemini`, not `agy`)** — you're still calling the deprecated gemini-cli OAuth path. Switch to `agy` (this skill). See the "Why agy" note above.

**Answer ignores your code / "scratch workspace is empty"** — you forgot `--add-dir "$PWD"`. agy defaults to an empty scratch workspace; add the dir you want it to read.

**Backgrounded run never returns / 0-byte output** — missing `</dev/null`. A backgrounded `agy -p` blocks on stdin EOF until you redirect stdin from `/dev/null`.

**Auth / login errors from `agy`** — it reuses your Antigravity login. If it can't authenticate, open the Antigravity app once to refresh the session, then retry. No API key needed.

## Step 3: Do Your Own Independent Review

While agy runs, do your own analysis. Cover the same questions independently. **Do not read the agy output until your own review is complete** — the value is in having two unbiased perspectives to compare.

## Step 4: Read and Compare

Once the background command completes (wait for the task notification), read the output. Compare against your own review:
- Accept findings grounded in specific file/line references from the codebase
- Override findings that rely on assumptions the model couldn't verify
- Pay attention to disagreements — those are the most interesting

## Step 5: Update the Plan

For each accepted finding, update the plan file. For each override, note why.

## Notes

- `agy` defaults to a Gemini model; pass `--model` to choose (run `agy models` to list). Use a Gemini model here to keep this a "Gemini perspective."
- It will autonomously read files in the added directory — expect multiple tool calls and 30–120 seconds runtime.
- Unlike the old gemini-cli, `agy` needs `--add-dir` to see your project — it does not auto-read the shell's cwd.
- **Background task race condition:** when `agy` runs as a `run_in_background` Bash task, the output file may appear empty (0 bytes) briefly after the task reports completion — there can be a small delay between process exit and file flush. Always wait for the background task notification before reading output, and if it reads empty, wait 2-3s and retry before concluding the model returned nothing.
- **Graceful degradation:** if `agy` fails (timeout, not installed, auth), continue with your own independent review. A single-perspective review is still valuable.
- If `agy` is not on PATH: it ships with the Antigravity app; run `agy install` to configure shell paths, or call it at `~/.local/bin/agy`.

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
| Engine | OpenAI Codex CLI (`codex exec`) | Antigravity CLI (`agy --print`) |
| Model | GPT-family | Gemini 3.1 Pro (or any `agy models` entry) |
| Non-interactive flag | `--config 'approval_policy="never"'` | `-p` / `--print` |
| Codebase access | `sandbox_permissions=["disk-full-read-access"]` | `--add-dir "$PWD"`, read-only by default |
| Stdin | inline content in prompt arg (`</dev/null`) | inline content in prompt arg (`</dev/null`) |
| Best for | GPT-family second opinion | Gemini-family second opinion |
