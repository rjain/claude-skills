# claude-skills

A collection of reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for multi-model collaboration on planning and design.

Each skill runs one or more AI CLI tools (Codex, and a Gemini model via Antigravity's `agy`) autonomously against your real codebase, then synthesizes their findings alongside Claude's own review. The result is a grounded, multi-perspective critique you can act on immediately. They range from single-shot review (`/codex-brainstorm`, `/gemini-brainstorm`, `/dual-brainstorm`) to a stateful multi-round debate (`/team-brainstorm`).

## What are skills?

Skills are prompt files that extend Claude Code with slash commands. Each skill lives in its own directory containing a `SKILL.md` file. Claude Code loads them automatically and exposes them as `/skill-name` commands in any session.

## Installation

There are two ways to install: as a **plugin** (one install, all three skills) or as **individual skills** (pick what you need). Both give you the same slash commands.

### Option A: Plugin (recommended for most users)

A plugin is a versioned bundle tracked in `settings.json`. One command installs all three skills at once.

In a Claude Code session, run:
```
/plugin marketplace add rjain/claude-skills
```

This registers the repo. You'll then be prompted to browse and enable the `multi-model-brainstorm` plugin, which contains `/codex-brainstorm`, `/gemini-brainstorm`, and `/dual-brainstorm`. Choose whether to enable it at project level (current project only) or global level (all sessions including desktop app).

### Option B: Individual Skills

A skill is a single `SKILL.md` file that defines one slash command. Symlink only the skills you want directly into Claude's skills directory — no plugin system involved.

```bash
git clone https://github.com/rjain/claude-skills.git ~/claude-skills

# Symlink the skills you want (team-brainstorm is standalone — not in the plugin bundle)
ln -s ~/claude-skills/codex-brainstorm  ~/.claude/skills/codex-brainstorm
ln -s ~/claude-skills/gemini-brainstorm ~/.claude/skills/gemini-brainstorm
ln -s ~/claude-skills/dual-brainstorm   ~/.claude/skills/dual-brainstorm
ln -s ~/claude-skills/team-brainstorm   ~/.claude/skills/team-brainstorm
```

Skills are available immediately — no restart required.

### Plugin vs. Individual Skills

| | Plugin | Individual Skills |
|---|---|---|
| **Install** | One command installs all three | Symlink each skill separately |
| **Updates** | Versioned; update via marketplace | Manual `git pull` |
| **Scope** | Project or global `settings.json` | Global (`~/.claude/skills/`) or project (`.claude/skills/`) |
| **Flexibility** | All-or-nothing bundle | Pick only the skills you need |
| **Desktop app** | Only if installed globally (see below) | Works in desktop if symlinked to `~/.claude/skills/` |

### Global vs. Project-Level Installation

> **Caution:** Think carefully about where you install.

- **Project-level** (`.claude/settings.json` or `.claude/skills/` inside a project) — skills are only available when Claude Code is opened in that project directory. The Claude desktop app won't see them.
- **Global** (`~/.claude/settings.json` or `~/.claude/skills/`) — skills are available in every session, including the desktop app.

**If you installed the plugin at project level** and want it everywhere, either:
1. Add `"multi-model-brainstorm@claude-skills": true` to your global `~/.claude/settings.json` under `enabledPlugins`, or
2. Switch to individual skill symlinks into `~/.claude/skills/` (Option B above).

For most users, **global installation** is simpler — you get the skills everywhere without thinking about which project you're in.

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — `npm install -g @anthropic-ai/claude-code`
- `codex` — `npm install -g @openai/codex`
- `agy` (Antigravity CLI) — ships with the [Antigravity app](https://antigravity.google); provides the Gemini model. (The old `gemini` CLI is no longer usable headlessly — Google discontinued its free "Login with Google" auth on 2026-06-18.)
- `python3` — used by the Codex JSONL parser in `/dual-brainstorm` and `/team-brainstorm` (preinstalled on macOS and most Linux). If it's ever unavailable, the skills fall back to a `jq` one-liner.

### Auto-mode permission (required if you run Claude Code in auto mode)

In **auto mode**, every Bash command is screened by an auto-mode classifier, and agentic CLIs like `codex exec` and `agy` get **blocked** ("Blocked by classifier", the tool never runs) unless you allowlist them. Add these to your project `.claude/settings.local.json` (or user settings) so each seat bypasses the classifier:

```json
{
  "permissions": {
    "allow": [
      "Bash(codex exec:*)",
      "Bash(agy:*)"
    ]
  }
}
```

Notes:
- A stale `Bash(gemini:*)` rule does **not** cover `agy` — it's a different binary. If the Gemini seat is classifier-denied while Codex succeeds, a missing `Bash(agy:*)` rule is the cause.
- This is only needed in auto mode. In the default (interactive) mode you'll simply be prompted to approve each command instead.

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

Runs a Gemini-model brainstorming session non-interactively via [Antigravity's `agy` CLI](https://antigravity.google) (`agy --print`). It reads the working directory (passed with `--add-dir`) before answering, providing a Gemini-family perspective on the same questions. Replaces the old `gemini` CLI, whose free login Google discontinued on 2026-06-18.

**When to use:** You want a Gemini second opinion, or as a complement to `/codex-brainstorm` to see where the two models agree vs. diverge.

**Requires:** `agy` (Antigravity CLI) on PATH — ships with the [Antigravity app](https://antigravity.google)

**Usage:**
```
/gemini-brainstorm path/to/plan.md
/gemini-brainstorm "the onboarding flow design"
```

**What happens:**
1. Claude builds the same three-question prompt
2. Launches the Gemini model in the background (`agy --add-dir "$PWD" -p "..." </dev/null`)
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

**Requires:** Both `codex` and `agy` (Antigravity CLI) on PATH. Falls back to the available tool + Claude's own review if one is missing.

**Usage:**
```
/dual-brainstorm path/to/plan.md
/dual-brainstorm "the billing and quota architecture"
```

**What happens:**
1. Fires both `codex exec` and `agy -p` as background Bash commands in a **single message** — true parallelism, neither blocks the other. Auto-detects non-git directories and adds `--skip-git-repo-check` for Codex when needed
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

### `/team-brainstorm`

Runs a **stateful, multi-round debate** across Claude, Codex, and a Gemini model. Where `/dual-brainstorm` is single-shot (each tool answers once), team-brainstorm keeps all three models in conversation: each researches independently in Round 1, then **critiques the others' positions** and concedes or defends across later rounds. Each model retains its own full memory between rounds — Codex via `codex exec resume`, Gemini via `agy --conversation`, Claude natively — so positions genuinely evolve. Claude orchestrates the loop, detects convergence, and synthesizes a consensus.

> **Note:** team-brainstorm is a **standalone skill** — it is not part of the `multi-model-brainstorm` plugin bundle. Install it individually by symlinking it (see [Option B](#option-b-individual-skills)): `ln -s "$PWD/team-brainstorm" ~/.claude/skills/team-brainstorm`.

**When to use:** Higher-stakes questions where you want positions stress-tested, not just collected — architecture decisions, trade-off analysis, or any topic where surfacing *why* models disagree (and watching one concede) is more valuable than a static side-by-side. Works on code topics or any research question.

**Requires:** `codex` and `agy` on PATH, plus `python3` (Codex JSONL parsing). Falls back to a two-model debate if one tool is missing. In auto mode, the same `Bash(codex exec:*)` / `Bash(agy:*)` allow-rules apply (see [Prerequisites](#prerequisites)).

**Usage:**
```
/team-brainstorm path/to/plan.md
/team-brainstorm "best approach to rate limiting in our API"
/team-brainstorm --rounds 2 --consensus strict "the cache invalidation design"
```

**Flags:**
- `--rounds N` — max debate rounds (default: 3)
- `--consensus strict|majority|any` — `strict` requires 3/3 unanimity; `majority` (default) resolves 2/3 after the final round; `any` stops after Round 1 (equivalent to `/dual-brainstorm`)
- `--codex-model` / `--codex-effort` / `--gemini-model` — per-seat model overrides

**What happens:**
1. **Preflight** — detects the runtime and picks the best Codex path (MCP in Claude Code CLI; CLI-resume in Claude Desktop, which has a 60s MCP timeout)
2. **Round 1** — all three research independently and take clear positions
3. **Round 2+** — each model sees the others' positions and challenges, concedes, or strengthens its own; Claude tracks agreements, disputes, and concessions
4. **Synthesis** — a consensus table showing each model's position per finding and how it resolved (unanimous / majority / flagged), plus a short narrative of who conceded what

**Why a debate, not just parallel answers?** Independent answers can be confidently wrong in the same way. Forcing the models to react to each other surfaces the reasoning behind disagreements and lets a stronger argument actually change a position — which a one-shot synthesis can't capture.

---

## How skills work

Each `SKILL.md` contains a YAML frontmatter block (`name`, `description`) followed by the full instructions Claude follows when the skill is invoked. Skills can use any tool available to Claude Code — Bash, Read, Write, Grep, etc.

Claude Code reads the active skill's `SKILL.md` as a prompt extension when you invoke the slash command. The skill runs in the context of your current working directory, so the tools it calls have full access to your codebase.

## License

MIT
