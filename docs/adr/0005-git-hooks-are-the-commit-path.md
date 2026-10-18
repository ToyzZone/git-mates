# Git hooks are the one commit path, for agents and the human alike

Every session commits with plain `git commit`, and the layer steps in through git's own hooks. `pre-commit` strips other sessions' lines from the index being committed. `post-commit` moves the commit git made on the workspace branch onto the committer's session branch and rebuilds the workspace commit. The committer is `CLAUDE_CODE_SESSION_ID` from the environment, or the Human session when it is unset. Claude Code hooks never see the human's commits, so git hooks are needed anyway. Using them for agents too gives one code path, and it also catches git run from scripts. Claude Code's PreToolUse hook only blocks commands that must be stopped before they run, and never rewrites one.

## Considered Options

- **PreToolUse rewriting `git commit` into the layer's own command:** rejected. It has to parse shell (`cd x && git commit -m "$(cat <<EOF …)"`), it misses git run inside scripts, and the human still needs git hooks.
- **A `git` shim put first on PATH by a SessionStart hook through `CLAUDE_ENV_FILE`** (possible, per the Claude Code docs): rejected. It would control the output fully, but it sits on every git call an agent makes, `/usr/bin/git` bypasses it, and the human still needs git hooks.

## Consequences

- git prints the sha of the commit it made on the workspace branch, which post-commit replaces. The hook adds a line naming the real branch and sha.
- post-commit must split the commit again by itself, because `--no-verify`, `git commit -- <path>`, `cherry-pick` and `revert` reach it without a pre-commit that did the job. `merge` and `pull` run neither hook, so PreToolUse blocks them for agents.
- The layer depends on its git hooks being installed alongside the repo's own hooks (husky, lint-staged) and any `core.hooksPath`.
- Facts, verified on git 2.54.0: [research/git-hooks-for-session-commits.md](../../research/git-hooks-for-session-commits.md). Details: [the ticket](../../.scratch/agent-ready-vcs/issues/06-committing-only-own-wip.md).
