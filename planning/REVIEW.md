# Change Review (since last commit)

## Findings

### 1. High: Stop hook can recursively trigger itself and may execute twice per stop event
- File: `.claude/settings.json:2-13`, `independant-reviewer/.claude-plugin/hooks/hooks.json:2-13`
- Both define a `Stop` hook that runs:
  - `codex exec --full-auto "Review all changes since the last commit and write feedback to planning/REVIEW.md"`
- Risk:
  - Duplicate execution on each stop (settings hook + plugin hook).
  - Recursive re-entry if the spawned `codex exec` process also loads the same Stop hook, causing repeated child runs.
- Suggested fix:
  - Keep only one source of truth for this hook (either settings or plugin, not both).
  - Add a recursion guard (env flag, lock file, or condition) so hook-spawned runs do not trigger the same hook again.

### 2. High: Plugin source path appears incorrect relative to marketplace file
- File: `.claude-plugin/marketplace.json:10`
- Current value is `"source": "./independant-reviewer"` while the marketplace file is at `.claude-plugin/marketplace.json`.
- `./independant-reviewer` relative to `.claude-plugin/` resolves to `.claude-plugin/independant-reviewer` (does not exist), while the plugin directory actually exists at repo root `./independant-reviewer`.
- Suggested fix:
  - If paths are marketplace-relative, update to `"../independant-reviewer"`.
  - If paths are repo-root-relative by convention, document that explicitly in this repo to avoid future breakage.

### 3. Medium: README quick start references scripts that do not exist in repo
- File: `README.md:26`, `README.md:31`, `README.md:47`
- README now instructs `./scripts/start_mac.sh` and `./scripts/stop_mac.sh`, and lists `scripts/` in layout, but no `scripts/` directory exists.
- Impact: fresh setup flow fails immediately for new users.
- Suggested fix:
  - Either add the referenced scripts, or restore runnable Docker commands in Quick Start.

## Notes
- The cleanup of old reviewer agent markdown files is reasonable if plugin-based workflow is replacing it.
- Main risk in this diff is operational behavior of auto-review hooks, not documentation wording.
