# What the workspace looks like to plain git

Type: grilling
Status: resolved
Blocked by: 11
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

With one Session branch per session, the shared working directory holds every session's commits plus everyone's WIP. What does plain git see?

- **HEAD:** a commit that merges every session branch, or something else? Its branch name, and when it is rebuilt.
- **The human's plain git:** what `git status`, `git log`, `git diff`, `git commit`, `git switch` and `git push` do here. The human is a session too, so does the Human session get a session branch?
- **Session branches:** where they live (`refs/heads/...` or a private namespace) and how they are named.
- **The base branch:** how it is chosen, and what happens when the human switches branches.

## Answer

Decided with the user on 2026-10-18.

- **HEAD:** sits on a **Workspace branch**, `<tool>/workspace` (the name waits on the project name). Its tip is one **Workspace commit** made by the layer. Its parents are the base tip plus every session branch that has commits. Each rebuild rewrites it in place, so the branch always has exactly one commit on top and nothing piles up. HEAD stays on the base branch until any session first commits, then moves to the workspace branch, and no file on disk changes when it moves. A `pre-push` hook refuses to push the workspace branch.
  - Rejected: a detached HEAD (alarming `git status`, and commits made there float). Also rejected: sitting directly on a session branch while it is the only one ahead, as GitButler now does. The branch name in the prompt would flip back and forth, and the human's commits would land on an agent's branch.
- **Workspace commit's tree:** the previous workspace tree plus the changes just committed, not a fresh merge of the session branches. It always matches what is committed on disk and never fails, even when two session branches conflict with each other. That conflict only shows when a branch merges into base, and handling it is up to [Changes that can't stand on their own session branch](13-changes-that-cant-stand-alone.md).
  - Rejected: re-merging the branch tips, as GitButler does, where a conflict forces one branch out of the working directory. Recorded in [ADR 0004](../../../docs/adr/0004-workspace-commit-records-the-disk.md).
  - Rebuilt after every commit by any session, under the same lock as the session branch compare-and-swap and the shared index patch ([partial commits](02-partial-commits-in-plain-git.md)). Throwing a session away and base moving also need rebuilds. Those belong to [Throwing away all of one session's work](14-throwing-away-a-session.md) and [Keeping session branches current](17-keeping-session-branches-current.md).
- **Human session:** has its own session branch, `<tool>/human`, which lives on indefinitely. So the layer never moves the base branch: base moves only when the human merges or pulls. The cost: to ship a hand fix, the human pushes `<tool>/human` or merges it into base.
- **The human's plain git on the workspace branch:**
  - `git status` and `git diff`: "On branch `<tool>/workspace`", followed by every session's WIP with no owners shown. A view of who owns what stays in the fog.
  - `git log`: the workspace commit, then every session's commits, then base history.
  - `git commit`: allowed, so VS Code's commit button keeps working. The commit lands on the workspace branch, then a `post-commit` git hook moves it onto the Human session branch and rebuilds the workspace commit. Claude Code hooks never see the human, so this path has to use git hooks. Keeping other sessions' lines out of the human's commit is handed to [How a session commits only its own WIP](06-committing-only-own-wip.md).
  - `git push`: `pre-push` refuses the workspace branch. Session branches push like any other branch.
  - `git switch <other>`: allowed, and it means the human leaves the workspace.
    - Tested on git 2.54.0: no hook can stop a switch before the working tree changes. `reference-transaction` does fire for HEAD, but rejecting the update leaves the new files and index in place under the old HEAD, and with `-b`/`-c` the new branch has already been created. `post-checkout` runs only after the switch.
    - Nothing is lost. Commits stay on their session branches, and git refuses any switch that would overwrite WIP.
    - While HEAD is off the workspace branch, every session's commit is refused, with a message to switch back. `post-checkout` prints a warning listing the live sessions. Switching back restores everything.
- **Session branches:** plain `refs/heads/<tool>/<first 8 chars of session_id>`, so they show in `git branch` and VS Code, and push and open PRs like any other branch. A session's branch is created at its first commit, from the base tip at that moment. A session that never commits leaves no branch.
  - Rejected: a private `refs/<tool>/` namespace. A PR needs the branch under `refs/heads/` on the remote anyway.
- **An agent's `git checkout -b <name>` or `git switch -c <name>`:** gives its session branch the name `<name>`. HEAD stays on the workspace branch, and the agent sees git's usual "Switched to a new branch" output. An agent's switch to an existing branch is blocked, with a message explaining that the folder is shared. Unlike git hooks, PreToolUse can block a command before it runs.
- **Base branch:** the branch HEAD is on when the workspace is entered. The layer records it. Each shared working directory has one base. Moving the workspace to another base is part of [Keeping session branches current](17-keeping-session-branches-current.md).
- **Fact for storing state:** tested that `git branch -m` carries every `branch.<name>.*` key along, custom keys included, and that `git branch -d/-D` deletes the whole section. A session id kept in the session branch's config therefore survives renames made with plain git.
