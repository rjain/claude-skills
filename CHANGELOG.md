# Changelog

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
