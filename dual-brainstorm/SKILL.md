---
name: dual-brainstorm
description: Run Codex and Gemini brainstorming sessions in parallel, then synthesize both perspectives. Pass a plan file path or topic as the argument. Launches both CLI tools simultaneously as background tasks so neither blocks the other.
---

# Dual Brainstorm: Codex + Gemini in Parallel

Runs `/codex-brainstorm` and `/gemini-brainstorm` simultaneously as background Bash commands, does an independent review in parallel, then synthesizes all three perspectives.

## Step 1: Launch Both CLI Tools in Parallel

In a **single message**, fire two background Bash tool calls simultaneously:

**Codex:**

First detect whether the working directory is inside a git repo. Codex refuses to run in non-git directories without `--skip-git-repo-check`:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  GIT_FLAG=""
else
  GIT_FLAG="--skip-git-repo-check"
fi
```

```bash
cat $PLAN_FILE | codex exec \
  $GIT_FLAG \
  --config 'approval_policy="never"' \
  --config 'sandbox_permissions=["disk-full-read-access"]' \
  "$PROMPT" 2>&1
```

**Gemini:**
```bash
cat $PLAN_FILE | gemini --approval-mode yolo -p "$PROMPT" 2>&1
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

Once both background tasks complete (you'll receive two notifications), read both output files and compare them across three dimensions:

**Agreements** — findings both tools reached independently. High-confidence; almost certainly worth acting on.

**Unique to Codex only** — validate against the codebase before accepting. May reflect GPT-family reasoning patterns.

**Unique to Gemini only** — validate against the codebase before accepting. May reflect Gemini-family reasoning patterns.

**Disagreements** — Codex and Gemini reached opposite conclusions. These are the most interesting; investigate manually.

## Step 4: Three-Way Synthesis

Compare all three perspectives (Codex, Gemini, your own review):

| Finding | Codex | Gemini | You | Action |
|---------|-------|--------|-----|--------|
| Gap X   | ✓     | ✓      | ✓   | Accept — unanimous |
| Gap Y   | ✓     | ✗      | ✓   | Investigate — split |
| Gap Z   | ✗     | ✗      | ✓   | Keep — your finding |
| Gap W   | ✓     | ✓      | ✗   | Review — missed it |

- **3/3 agreement** → accept without further validation
- **2/3 agreement** → accept if grounded in a file/line reference
- **1/3** → requires manual codebase check before accepting
- **Codex ≠ Gemini** → investigate the disagreement directly

## Step 5: Update the Plan

For each accepted finding, update the plan file. For overrides, note which tool raised it and why you're overriding.

## Notes

- Both tools autonomously read the codebase — expect 10–30 tool calls each, 30–120s per tool
- Fire them in the same message to get true parallelism; don't wait for one before starting the other
- If one tool is unavailable (not on PATH), fall back to the other + your own review
- The three-way comparison is more valuable than any single tool — disagreements surface the most interesting gaps
