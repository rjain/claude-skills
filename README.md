# claude-skills

A collection of reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for multi-model collaboration on planning and design.

Each skill runs one or more AI CLI tools (Codex, Gemini) autonomously against your real codebase, then synthesizes their findings alongside Claude's own review. The result is a grounded, multi-perspective critique you can act on immediately.

## What are skills?

Skills are prompt files that extend Claude Code with slash commands. Each skill lives in its own directory containing a `SKILL.md` file. Claude Code loads them automatically and exposes them as `/skill-name` commands in any session.

## Installation

```bash
git clone https://github.com/rjain/claude-skills.git ~/sw/ai/claude-skills

# Symlink each skill into Claude's skills directory
for skill in ~/sw/ai/claude-skills/*/; do
  ln -s "$skill" ~/.claude/skills/"$(basename "$skill")"
done
```

Skills are available immediately — no restart required.

**Prerequisites:**
- `codex` — `npm install -g @openai/codex`
- `gemini` — `npm install -g @google/gemini-cli`

---

## Skills

### `/codex-brainstorm`

Runs a [Codex CLI](https://github.com/openai/codex) brainstorming session non-interactively. Codex autonomously reads your repo (grep, cat, etc.) before forming opinions, so its feedback is grounded in actual code rather than the plan description alone.

**When to use:** You have a plan file or design doc and want a GPT-family second opinion that has read the real codebase.

**Requires:** `codex` on PATH — `npm install -g @openai/codex`

**Usage:**
```
/codex-brainstorm path/to/plan.md
/codex-brainstorm "the auth token refresh flow"
```

**What happens:**
1. Claude builds a focused prompt covering three questions: gaps, fastest win, unconventional approach
2. Launches Codex in the background (`codex exec --config 'approval_policy="never"'`), auto-detecting non-git directories and adding `--skip-git-repo-check` when needed
3. Does its own independent review while Codex runs (30–120s)
4. Compares both perspectives, accepts findings grounded in file/line references
5. Updates the plan file with accepted findings

**Example output:**

```
Launching Codex in the background...
[Claude's independent review in progress]

Codex output received. Comparing perspectives...

Codex-unique findings (grounded, accepted):

- `quotaOverride` is client-writable — firestore.rules:24 lets owners
  update any field except `isAdmin` and `createdAt`. A signed-in user
  can self-upgrade their own daily query limit by writing directly to
  Firestore. Fix: add `quotaOverride` to the excluded fields list.

- `/chat` has no user-existence gate before writing quota state —
  chat.py:32 accepts any `?user=` anonymously and increments quota at
  chat.py:94 even when no context exists (gemini_service.py:134).
  Allows UID namespace pollution and junk analytics.

- Public/private state is co-mingled in `users/{uid}` — the same
  document is read publicly at index.html:225 and written by admin at
  admin.html:992. Consider splitting `public_profiles/{uid}` from
  `users/{uid}` so public reads never touch auth/quota fields.

Unconventional approach for P3 access control:
Use capability links instead of a full guest-account system. Mint
signed, short-lived "viewer passes" scoped to {portfolio_uid,
permissions, expires_at}. Require the pass on /chat (chat.py:63) and
/user-content/* (main.py:114). No viewer auth flow needed.

Plan updated: 3 new items added, 1 annotated.
```

---

### `/gemini-brainstorm`

Runs a [Gemini CLI](https://github.com/google-gemini/gemini-cli) brainstorming session non-interactively. Gemini reads the working directory autonomously with `--approval-mode yolo` before answering, providing a Gemini-family perspective on the same questions.

**When to use:** You want a Gemini second opinion, or as a complement to `/codex-brainstorm` to see where the two models agree vs. diverge.

**Requires:** `gemini` on PATH — `npm install -g @google/gemini-cli`

**Usage:**
```
/gemini-brainstorm path/to/plan.md
/gemini-brainstorm "the onboarding flow design"
```

**What happens:**
1. Claude builds the same three-question prompt
2. Launches Gemini in the background (`gemini --approval-mode yolo -p "..."`)
3. Does its own independent review while Gemini runs
4. Compares perspectives and updates the plan

**Example output:**

```
Launching Gemini in the background...
[Claude's independent review in progress]

Gemini output received. Comparing perspectives...

Gemini-unique findings (accepted):

- Context/prompt injection via uploaded files — handle_upload_file at
  admin.py:181 has no content-security scanning. A user can upload a
  text file containing system-instruction override keywords. Since
  prompt_builder.py concatenates context files verbatim, this hijacks
  LLM behavior for any visitor and abuses the owner's quota. Add a
  safety pre-flight in gemini_service.py.

- No aggregate storage quota per user — admin.py:181-253 validates
  per-file word/char count but not total bytes stored. A user can bypass
  the LLM query limit by uploading thousands of small files. Add
  `total_bytes_stored` to users/{uid} and check it in storage_service.py
  before each upload.

- GCS/Firestore delete atomicity — if storage_service.delete_file() and
  file_repository.delete_file_metadata() fail independently, you get
  orphaned GCS blobs or broken UI references. Add a compensating delete
  pattern or a background cleanup task.

Fastest win: isPublic flag — ~2 hours: one Firestore field, two if
checks in chat.py and profile_og.py, one toggle in admin.html.
Solves "I'm not ready to share yet" drop-off immediately.

Unconventional approach for onboarding:
"AI Ghostwriter Interview" — instead of asking users to upload document,
the AI initiates a 3-question chat on first login. Turns a "work" task
(finding a PDF) into a "social" task (chatting). The backend generates
profile_data.txt from the answers automatically.

Plan updated: 3 new items added, onboarding section extended.
```

---

### `/dual-brainstorm`

Runs Codex and Gemini **simultaneously** as background tasks, does an independent Claude review in parallel, then produces a three-way synthesis. This is the most valuable of the three skills — the synthesis table shows exactly which findings all three models agree on (high confidence) vs. which only one raised (needs validation).

**When to use:** Before finalizing a plan, before starting a large implementation, or any time you want the highest-confidence critique with the least manual effort.

**Requires:** Both `codex` and `gemini` on PATH. Falls back to the available tool + Claude's own review if one is missing.

**Usage:**
```
/dual-brainstorm path/to/plan.md
/dual-brainstorm "the billing and quota architecture"
```

**What happens:**
1. Fires both `codex exec` and `gemini -p` as background Bash commands in a **single message** — true parallelism, neither blocks the other. Auto-detects non-git directories and adds `--skip-git-repo-check` for Codex when needed
2. Claude does its own independent review while both run (30–120s)
3. On completion, reads both outputs and builds a three-way comparison table
4. Applies a confidence tier: 3/3 → accept immediately, 2/3 → accept if grounded in a file reference, 1/3 → manual codebase check required
5. Updates the plan file with all accepted findings, noting which tool raised each one

**Example output:**

```
Launching Codex and Gemini in parallel...
[Independent review in progress — both tools running simultaneously]

Both tools completed (Gemini: 47s, Codex: 83s). Building synthesis...

| Finding                                   | Codex | Gemini | Claude | Action              |
|-------------------------------------------|-------|--------|--------|---------------------|
| isPublic flag = fastest win after P0s     |  ✓    |   ✓    |   ✓    | Accept — unanimous  |
| quotaOverride client-writable             |  ✓    |   ✗    |   ✗    | Add — file-grounded |
| /chat no user-existence gate              |  ✓    |   ✗    |   ✗    | Add — file-grounded |
| users doc mixes public/private state      |  ✓    |   ✗    |   ✗    | Add as arch note    |
| Prompt injection via uploaded files       |  ✗    |   ✓    |   ✗    | Add — file-grounded |
| Aggregate storage quota per user          |  ✗    |   ✓    |   ✗    | Add — file-grounded |
| GCS/Firestore delete atomicity            |  ✗    |   ✓    |   ✗    | Add — file-grounded |
| Firebase token emergency revocation path |  ✗    |   ✗    |   ✓    | Keep — Claude only  |
| GCS bucket-level IAM callout              |  ✗    |   ✗    |   ✓    | Keep — Claude only  |
| Audit log coverage gaps                   |  ✗    |   ✗    |   ✓    | Keep — Claude only  |

Unanimous (3/3):
  isPublic flag is the fastest win — confirmed, already in plan.

Codex-only findings added:
  quotaOverride self-upgrade bug (billing-class, exploitable today with
  zero backend involvement), /chat UID pollution, public/private schema
  split recommendation.

Gemini-only findings added:
  prompt injection via uploads, aggregate storage quota,
  GCS/Firestore delete atomicity.

Claude-only findings kept:
  Firebase token revocation path, GCS bucket IAM callout,
  audit log coverage.

Notable: The quotaOverride self-upgrade bug was the most surprising —
Codex caught it; Gemini missed it entirely. This is why running both
matters.

Plan updated: 8 new items added across P0–P2.
```

**Why three perspectives?** Codex and Gemini have different reasoning patterns and different strengths when reading code. In practice, they disagree on roughly half their findings — running only one gives you half the picture. The synthesis table makes disagreements explicit so you can investigate the most interesting ones directly.

---

## How skills work

Each `SKILL.md` contains a YAML frontmatter block (`name`, `description`) followed by the full instructions Claude follows when the skill is invoked. Skills can use any tool available to Claude Code — Bash, Read, Write, Grep, etc.

Claude Code reads the active skill's `SKILL.md` as a prompt extension when you invoke the slash command. The skill runs in the context of your current working directory, so the tools it calls have full access to your codebase.

## License

MIT
