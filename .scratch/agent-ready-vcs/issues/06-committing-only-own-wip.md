# How a session commits only its own WIP

Type: grilling
Status: resolved
Blocked by: 01, 02, 04, 05, 12
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

An agent will type its usual `git add . && git commit -m "..."`. How does it end up committing only the WIP it owns, onto its own Session branch ([ADR 0002](../../../docs/adr/0002-a-branch-per-session.md)), with what HEAD shows kept in step?

- **Where we step in:** a PreToolUse hook that rewrites the Bash command, a `git` shim on PATH, or something else?
- **What the agent sees:** do the output and errors look like normal git?
- **How the session is recorded on the commit.**

The pitch promises "no new CLI that agents must learn", so plain `git` commands must keep working.

## Comments

- From [What the workspace looks like to plain git](12-workspace-to-plain-git.md), 2026-10-18: this ticket now also covers the Human session and the other git commands that ticket decided.
  - The human's plain `git commit` lands on the workspace branch, and a `post-commit` git hook moves it onto the Human session branch. What keeps other sessions' lines out of that commit, for example when VS Code's "commit all" stages everyone's WIP? Claude Code hooks never see the human.
  - Git hooks also run for an agent's `git commit`, and `CLAUDE_CODE_SESSION_ID` is in their environment. That makes them a candidate place to step in for agents too.
  - An agent's `git checkout -b <name>` or `git switch -c <name>` names its session branch, and its switch to an existing branch is blocked. Design how these are caught alongside commit, and decide what an agent's plain `git push` pushes.
  - Edge case: `git commit --amend` right after a commit, when HEAD is the workspace commit.

## Answer

Decided with the user on 2026-10-18. Recorded in [ADR 0005](../../../docs/adr/0005-git-hooks-are-the-commit-path.md). Facts verified on git 2.54.0: [research/git-hooks-for-session-commits.md](../../../research/git-hooks-for-session-commits.md).

- **Where we step in: git hooks, for every session.** The committer is `CLAUDE_CODE_SESSION_ID` from the hook's environment, or the Human session when it is unset. Agents and the human share one code path, git run from scripts is covered, and no shell command is parsed or rewritten.
  - `pre-commit` strips every line the committer doesn't own from the index being committed. If nothing is left, it fails the commit; git would otherwise make an empty one.
  - Git makes commit C on the workspace branch as usual.
  - `post-commit` rebuilds C's change onto the committer's session branch, then rebuilds the workspace commit with C's tree, so no file on disk changes. It also splits C again: everything in C except lines owned by another session belongs to the committer, and the rest goes back to WIP. This covers `--no-verify` (it skips pre-commit, not post-commit), `git commit -- <path>` (its stripped lines stay staged, because `index.lock` is held during pre-commit) and `cherry-pick`/`revert` (they run post-commit but not pre-commit).
  - PreToolUse only blocks, and only what must be stopped before it runs. It never rewrites a command.
  - Rejected: PreToolUse rewriting `git commit` into the layer's own command (it parses shell and misses git inside scripts), and a `git` shim put first on PATH through `CLAUDE_ENV_FILE` (possible, but it sees only agents and `/usr/bin/git` bypasses it). Both still need git hooks for the human, so either would make two paths.
- **What a commit holds:** lines that are both staged and owned by the committer. `git add .` commits all of its WIP, `git add <file>` its lines in that file, and `-a` all its WIP in tracked files. Other sessions' lines that got staged, for example by `git add .`, are unstaged after the commit. With none of its own lines staged, the commit is refused, naming the sessions that own the staged lines.
- **Session on the commit:** a `Session-Id: <session_id>` trailer on agent commits only, added when the session branch commit is built. Once a session branch merges and is deleted, its config is gone, and the trailer is the only link from a commit to its conversation (`claude --resume <id>`). Human commits carry none.
- **Entering the workspace:** only an agent's first commit does it. That commit's `pre-commit` creates the workspace branch at HEAD and points HEAD at it, so git's commit lands there and base stays put (verified). Until then, the human's commit is a normal commit on base, holding Human lines only. This narrows the "first session commit" of [What the workspace looks like to plain git](12-workspace-to-plain-git.md).
- **Push:** `pre-push` refuses the workspace branch and gives the exact plain command to run instead, for example "Your commits are on `fix-login`. Run `git push -u origin fix-login`." For the human, it lists the session branches.
- **Amend** means amending the last commit on the committer's session branch. Unless `-m` is given, `prepare-commit-msg` swaps in that commit's message for the workspace commit's, and `post-commit` folds the newly staged lines into it. With no commit of its own, amend is refused. A pushed commit then needs a force-push, as in plain git. This covers VS Code's "Commit (Amend)" too.
- **`git checkout -b <name>` / `git switch -c <name>`, by an agent or the human:** `post-checkout` sees the old and new commit are the same, points HEAD back at the workspace branch and deletes the new branch. No file changes. The name is used exactly as typed. With no session branch yet, the name is kept for the first commit. A second `-b` renames the branch again. git itself refuses `-b` with an existing name.
- **History commands from an agent:** PreToolUse blocks `git reset <commit>`, `rebase`, `merge` and `pull`, saying the folder is shared and naming the agent's branch. To undo a commit, the message points to `git revert <sha>`. `cherry-pick` and `revert` go through post-commit like any commit. `git reset` without a commit (unstaging) is allowed. Translating these commands onto the session branch is left in the fog.
- **What the agent sees after a commit:** git's own output, plus one line from the hook printed before it: "Committed to branch `fix-login` as def4567. The `[<tool>/workspace …]` line below is the shared folder's commit; refer to def4567." The sha git prints is C's, which gets replaced.
- **The workspace branch moved outside the layer:** no hook can stop the human's `reset`, `rebase` or `merge` cleanly. `reference-transaction` can reject the ref update, but `reset --mixed` and `--hard` have already rewritten the index or files by then (verified). So every snapshot and every post-commit checks that the workspace branch still ends in the last workspace commit the layer made.
  - Moved during one agent's lone Bash call, for example a history command inside a script that slipped past PreToolUse: the layer puts the branch and index back at once, leaving files alone, and tells that agent in PostToolUse `additionalContext`. The command wasn't allowed, so no one's intent is undone.
  - Moved by the human, or during overlapping Bash calls: commits pause for every session, as when HEAD leaves the workspace. Each refused commit says how to resume: the human runs `git reset <sha>`, a mixed reset that keeps the files and makes the index match. Agents can't run it (blocked above), so they pass it on to the human. A known limit of v0.
  - A plain commit sitting directly on the last workspace commit, left there by a crashed post-commit, is moved onto its session branch without pausing.
- **Two sessions committing at once:** a layer lock is held from `pre-commit` to the end of `post-commit`, because git releases `index.lock` before post-commit runs. A second commit waits a few seconds. The lock records the git process's pid and counts as stale once that process is dead, since a commit aborted after pre-commit never reaches post-commit.

Handed to other tickets (see their Comments):

- [Changes that can't stand on their own session branch](13-changes-that-cant-stand-alone.md): where a commit whose lines can't apply onto the session branch tip is stopped.
- [How ownership is recorded and kept current](16-recording-ownership.md): compare against the recorded workspace commit, not HEAD, once the workspace branch has moved.
- [Guarding against Lost WIP](07-guarding-against-lost-wip.md): a `reset --hard` that slips past PreToolUse, and the human's, which no hook can stop.
- [Keeping session branches current](17-keeping-session-branches-current.md): the human merging base into the workspace branch.
