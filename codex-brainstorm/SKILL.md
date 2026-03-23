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

## Step 1: Build the Prompt

Construct a focused prompt. For a plan file, include:
- What the app does (tech stack, one sentence)
- The three questions to answer:
  1. Any critical gaps still missing?
  2. What is the single fastest win?
  3. One unconventional alternative approach?

For open-ended brainstorming, ask for: multiple angles, trade-offs, creative alternatives.

## Step 2: Run Codex in the Background

First detect whether the working directory is inside a git repo:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  GIT_FLAG=""
else
  GIT_FLAG="--skip-git-repo-check"
fi
```

Then launch Codex:

```bash
cat $PLAN_FILE | codex exec \
  $GIT_FLAG \
  --config 'approval_policy="never"' \
  --config 'sandbox_permissions=["disk-full-read-access"]' \
  "$YOUR_PROMPT" 2>&1
```

Key flags:
- `--skip-git-repo-check` — required when the working directory is not a git repo (Codex refuses to run otherwise)
- `approval_policy="never"` — fully non-interactive, no approval prompts
- `sandbox_permissions=["disk-full-read-access"]` — lets Codex read the codebase freely
- Pipe the plan file via stdin so Codex has the context without you duplicating it in the prompt
- Codex will autonomously grep/read the repo to ground its feedback in real code

Run as a background Bash command so the main conversation stays free.

## Step 3: Do Your Own Independent Review

While Codex runs, do your own analysis. Cover the same questions independently. **Do not read the Codex output until your own review is complete** — the value is in having two unbiased perspectives to compare.

## Step 4: Read and Compare

Once the background command completes, read the output file. Compare against your own review:
- Accept findings that are grounded in specific file/line references
- Override findings that rely on assumptions Codex couldn't verify
- Flag any finding where Codex and your review disagree — those are the most interesting

## Step 5: Update the Plan

For each accepted finding, update the plan file. For each override, note why.

## Notes

- Codex uses `gpt-5.4` by default (as of 2026-03) with high reasoning effort
- It will autonomously explore the repo — expect 10–30 tool calls and 30–120 seconds runtime
- Output goes to a temp file; use `run_in_background: true` on the Bash call so the main context isn't blocked
- If `codex` is not on PATH: `which codex` — it installs via npm as `codex-cli`

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
