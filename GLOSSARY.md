# Agent-ready version control

A git-compatible layer that lets several sessions edit one working directory at the same time without mixing or losing each other's changes.

## Language

### Who changes things

**Session**:
One writer making changes in the shared working directory: either an agent session or the human session.
_Avoid_: writer, lane, agent (when the human is meant too)

**Agent session**:
A session driven by an AI coding agent, such as one Claude Code session together with its subagents.
_Avoid_: bot, worker

**Human session**:
The session standing for the person at the keyboard; it owns every change that no agent session owns.
_Avoid_: default session, unattributed changes

### What gets changed

**Shared working directory**:
One checkout in which more than one session edits files at the same time.
_Avoid_: shared worktree, "same branch" (when the directory is meant)

**Ownership**:
The link between each changed line in the shared working directory and the single session it belongs to.
_Avoid_: attribution, assignment, hunk ownership

**Base branch**:
The branch that session branches start from and that their work is meant to land on, such as `main`.
_Avoid_: target, trunk

**Session branch**:
The git branch that holds one session's commits, started from the base branch. The human session has one too.
_Avoid_: virtual branch (GitButler's term), lane

**Workspace branch**:
The local branch the shared working directory sits on once any session has committed. It combines the base branch with every session branch.
_Avoid_: integration branch (GitButler's old name)

**Workspace commit**:
The single commit at the tip of the workspace branch, made by the layer, recording everything the sessions have committed so far.
_Avoid_: merge commit (it is not git's merge of the session branches)

**WIP**:
Changes in the shared working directory that a session owns and has not yet committed.
_Avoid_: dirty files, uncommitted changes (they hide who owns them)

### What goes wrong

**Mixed commit**:
A commit made by one session that contains changes owned by another session.

**Lost WIP**:
One session's action discarding another session's WIP.

**Clobbered edit**:
Two sessions changing the same lines, with one silently overwriting the other.

**Cross-session breakage**:
One session's WIP breaking another session's build or tests.
