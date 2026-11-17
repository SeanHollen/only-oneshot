# only-oneshot (Claude Code skill)

Public repo: SeanHollen/only-oneshot. `~/.claude/skills/only-oneshot` is a symlink to `skills/only-oneshot/` here, so edits take effect locally right away.

## Git
- **Commit and push after every change** (Sean's standing ask, 2026-11-17). Force-push and deleting branches or the repo still need Sean to name that exact action.
- Public repo: keep personal details out (names, home paths, private projects). The skill says "the user", never Sean.

## Testing
Test changes by running the skill in a fresh headless session (`claude -p --dangerously-skip-permissions --output-format json "/only-oneshot ..."` in a new folder, then `--resume <session_id>` for follow-up rounds), and check every claim against the test project's git history (which files changed per round, manifest, worktrees cleaned up).
- `total_cost_usd` in that JSON is API list price, not a charge: Sean runs on his subscription. Report usage, not dollars.
- Past test projects: `~/code/one-shot-test` (single file, tip calculator) and `~/code/oneshot-modular-test` (modular habit tracker). Logs in `~/code/oneshot-modular-test-logs/`.
