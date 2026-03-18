# claude-skills

A collection of reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills.

## What are skills?

Skills are prompt files that extend Claude Code with new slash commands. Each skill lives in its own directory with a `SKILL.md` file. Claude Code automatically loads them and exposes them as `/skill-name` commands.

## Installation

Clone this repo and symlink (or copy) the skill directories into `~/.claude/skills/`:

```bash
git clone https://github.com/rjain/claude-skills.git ~/sw/claude-skills

# Symlink each skill
for skill in ~/sw/claude-skills/*/; do
  ln -s "$skill" ~/.claude/skills/"$(basename "$skill")"
done
```

Or copy instead of symlinking:

```bash
cp -r ~/sw/claude-skills/*/ ~/.claude/skills/
```

After installation, the skills are available immediately in any Claude Code session — no restart required.

## Skills

### `/codex-brainstorm`

Run a [Codex CLI](https://github.com/openai/codex) brainstorming session grounded in your actual codebase. Codex reads your repo autonomously before forming opinions.

**Usage:** `/codex-brainstorm path/to/plan.md` or `/codex-brainstorm topic`

**Requires:** `codex` on PATH — install via `npm install -g @openai/codex`

---

### `/gemini-brainstorm`

Run a [Gemini CLI](https://github.com/google-gemini/gemini-cli) brainstorming session grounded in your actual codebase. Gemini reads your repo autonomously before forming opinions.

**Usage:** `/gemini-brainstorm path/to/plan.md` or `/gemini-brainstorm topic`

**Requires:** `gemini` on PATH — install via `npm install -g @google/gemini-cli`

---

### `/dual-brainstorm`

Run Codex and Gemini in parallel, do an independent Claude review simultaneously, then synthesize all three perspectives into a three-way comparison table. The most valuable of the three skills — disagreements between models surface the most interesting gaps.

**Usage:** `/dual-brainstorm path/to/plan.md` or `/dual-brainstorm topic`

**Requires:** Both `codex` and `gemini` on PATH. Falls back gracefully if one is unavailable.

---

## How skills work

Each `SKILL.md` contains:
- A YAML frontmatter block with `name` and `description`
- Detailed instructions Claude follows when the skill is invoked

Claude Code reads the `SKILL.md` as a system prompt extension when you invoke the corresponding slash command. The skill can use any tool available to Claude Code (Bash, Read, Write, etc.).

## License

MIT
