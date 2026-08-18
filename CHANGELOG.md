# Changelog

## 2026-08-17

### Add `team-brainstorm` to the plugin bundle (plugin 1.2.0 → 1.3.0)

`team-brainstorm` (stateful multi-round debate across Claude, Codex, and a Gemini model) was a working skill in the repo but shipped only via individual symlink install — it was absent from the `multi-model-brainstorm` plugin and the README. Plugin users now get all four skills.

- **Plugin manifest:** added `./team-brainstorm` to the `multi-model-brainstorm` skills list; bumped plugin + marketplace version to 1.3.0; refreshed description/keywords (added `debate`).
- **README:** added a `/team-brainstorm` section; updated "three → four skills" throughout; documented the auto-mode `Bash(codex exec:*)` / `Bash(agy:*)` allow-rules and `python3` as prerequisites.

### Reliability fixes (all brainstorm skills)

- **Codex JSONL parser:** capture the last non-empty `agent_message` instead of the last `item.completed` text. Real runs interleave `agent_message` (has text) with `command_execution` items (empty text), so the old logic could return an empty string on a successful run.
- **Auto-mode:** documented that the auto-mode classifier blocks `codex exec` / `agy` without explicit allow-rules — the `gemini`→`agy` migration left `agy` unlisted, silently blocking the Gemini seat.
- **dual-brainstorm:** fixed Step 1 example snippets to match the working invocation (`</dev/null`, output-file redirect, defined `$OUTPUT`); added `python3` preflight with a `jq` fallback.

## 2026-06-28

### Migrate all brainstorm skills from `gemini` CLI to Antigravity `agy`

Google discontinued the free "Login with Google" auth for Gemini Code Assist (individuals / AI Pro / AI Ultra) on **2026-06-18**, breaking headless `gemini -p` (it now fails with `IneligibleTierError … UNSUPPORTED_CLIENT`). All skills now use Google's official replacement, the **Antigravity CLI `agy`** — a native binary that reuses the existing Antigravity login (no API key, no node-version/PATH workaround).

**All brainstorm skills (gemini-brainstorm, dual-brainstorm, team-brainstorm):**
- Replaced `gemini -p` with `agy --add-dir "$PWD" --model "Gemini 3.1 Pro (High)" -p`
- `--add-dir "$PWD"` is required for codebase grounding (agy otherwise runs in an empty scratch workspace)
- Read-only by default (no `--dangerously-skip-permissions`), matching gemini's old `--approval-mode plan`
- Added `</dev/null` to backgrounded `agy -p` calls — without it agy hangs forever on stdin EOF (same trap as `codex exec`)
- Dropped the `gemini --version` gate and `@google/gemini-cli` install steps

**team-brainstorm (stateful multi-round):**
- agy has no `-o json` / `session_id`; capture the conversation id by diffing `~/.gemini/antigravity-cli/conversations/*.db` around round 1, then resume later rounds with `agy --conversation "<UUID>"`
- Verified `--conversation <id>` resumes that specific conversation even when it is not the most recent; an unknown id silently falls back to most-recent (never pre-mint ids; treat "not found" as a failed resume)

**Docs:**
- Updated README prerequisites and skill descriptions; bumped plugin to 1.2.0

## 2026-03-31

### codex-brainstorm, gemini-brainstorm, dual-brainstorm — Reliability improvements

Applied reliability patterns from `team-brainstorm` to the three simpler brainstorm skills.

**All three skills:**
- Added background task race condition handling — wait for task notification, retry empty reads after 2-3s
- Added graceful degradation — continue with remaining tools if one fails
- Added expected runtime documentation (30-120s per tool)

**codex-brainstorm:**
- Added Step 0: Preflight Checks with environment detection (`CLAUDE_CODE_ENTRYPOINT`) and CLI resume vs stateless mode selection
- Hardcoded `--skip-git-repo-check` (removed conditional git detection — brainstorm tasks never modify files)
- Added JSONL output (`--json`) with python helpers to extract `thread_id` and response text
- Added `--output-last-message` as alternative to JSONL parsing
- Added `--ephemeral` warning (breaks resume mode)
- Added Codex config pitfall documentation (`web_search = "live"` under `[features]` causes parse error)

**gemini-brainstorm:**
- Added `-o json` structured output with `session_id` parsing for session resume
- Added `--resume` workflow documentation for multi-turn conversations
- Added warning that `--approval-mode yolo` is NOT inherited on `--resume` — must pass on every call
- Updated comparison table with session resume and structured output rows

**dual-brainstorm:**
- Added Step 0: Preflight Checks — environment detection, tool availability check, Codex mode detection
- Hardcoded `--skip-git-repo-check`
- Added JSONL output for Codex with `thread_id` parsing
- Added `-o json` output for Gemini with `session_id` parsing
- Added Codex config pitfall documentation
- Added `--approval-mode yolo` inheritance warning
- Added environment detection context (`claude-desktop` = 60s MCP timeout, always use CLI)
