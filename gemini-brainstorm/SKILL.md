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

```bash
cat $PLAN_FILE | gemini --approval-mode yolo -p "$YOUR_PROMPT" 2>&1
```

Key flags:
- `--approval-mode yolo` — fully non-interactive, auto-approves all tool calls (file reads, shell commands)
- `-p` / `--prompt` — appended to stdin input, enabling the plan file to be piped in
- `-s` / `--sandbox` — optionally add for sandboxed execution
- `-m` / `--model` — optionally specify model (e.g. `-m gemini-2.5-pro`)
- Gemini will autonomously explore the repo before answering — expect shell/file tool calls

Run as a background Bash command (`run_in_background: true`) so the main conversation stays free.

## Step 3: Do Your Own Independent Review

While Gemini runs, do your own analysis. Cover the same questions independently. **Do not read the Gemini output until your own review is complete** — the value is in having two unbiased perspectives to compare.

## Step 4: Read and Compare

Once the background command completes, read the output file. Compare against your own review:
- Accept findings grounded in specific file/line references from the codebase
- Override findings that rely on assumptions Gemini couldn't verify
- Pay attention to disagreements between Gemini and your review — those are the most interesting

## Step 5: Update the Plan

For each accepted finding, update the plan file. For each override, note why.

## Notes

- Gemini CLI uses Gemini 2.5 Pro by default (as of 2026-03)
- It will autonomously explore the repo — expect multiple tool calls and 30–120 seconds runtime
- Output goes to a temp file; use `run_in_background: true` on the Bash call
- Unlike Codex, Gemini reads the working directory automatically — no need for `sandbox_permissions` config
- If `gemini` is not on PATH: install via `npm install -g @google/gemini-cli`

## Example Prompt Template (Plan Review)

```
You are a staff engineer doing a brainstorming session on this {topic} plan for a {app description}.
Tech stack: {stack}.
Read the plan below and brainstorm:
(1) any critical gaps still missing,
(2) the single fastest win after the highest-priority items are fixed,
(3) one unconventional approach to the hardest problem in the plan.
Be direct and concise.
```

## Comparison with /codex-brainstorm

| | `/codex-brainstorm` | `/gemini-brainstorm` |
|---|---|---|
| Model | GPT (via OpenAI Codex CLI) | Gemini 2.5 Pro |
| Non-interactive flag | `--config 'approval_policy="never"'` | `--approval-mode yolo` |
| Stdin piping | `cat file \| codex exec ... "prompt"` | `cat file \| gemini -p "prompt"` |
| Codebase access | `sandbox_permissions=["disk-full-read-access"]` | Automatic (working directory) |
| Best for | GPT-family second opinion | Gemini-family second opinion |
