# Unit of ownership and how every change gets attributed

Type: grilling
Status: resolved
Blocked by: 01, 03
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

What unit does a session own: a whole file, a hunk, or a line? How does every change get attributed to exactly one session, whether it comes through Edit/Write, through Bash, or by hand? Hand edits go to the Human session.

Also decide what happens when attribution is ambiguous. For example, two sessions' Bash commands run at the same time and a file changes. The user's scenario needs a quick fix and WIP to live side by side in the same file, which rules out whole-file ownership unless there is a good reason.

## Answer

Decided with the user in a grilling session on 2026-10-18. Recorded in [ADR 0003](../../../docs/adr/0003-own-changes-by-the-line.md).

- **Unit: the line.** Each changed line of WIP (against HEAD) belongs to exactly one session. A deleted line belongs to the session that deleted it, anchored at its position. Hunks are built only at commit time, from one session's lines.
  - A new file: its creator owns every line.
  - Deleting or renaming a file: one file-level change, owned by whoever did it. git sees a rename as a delete plus a create.
  - Binary files, mode changes, symlinks and submodule pointers: the whole file is one unit.
  - Ignored files are not WIP and have no owner. Ownership covers exactly what `git status` shows.
- **Owner key: `session_id`.** Subagents belong to their parent session; their Bash calls can't be told apart from the parent's anyway. The human stays a session, the Human session. An agent-only design (no hand edits) was considered and rejected: it removes only known limit 1 below, it can't be enforced, and it still needs an owner of last resort.
- **Sources: hooks only, no daemon.**
  - Edit, Write, NotebookEdit: `structuredPatch` in PostToolUse, plus `session_id`. Exact even when someone else types at the same moment.
  - Bash: a snapshot at PreToolUse and at PostToolUse. Lines that change in between belong to that session. Every snapshot by any session is a shared checkpoint, so only truly overlapping time is ambiguous.
  - Hand edits: a change that appears while no agent's call is running belongs to the Human session. It is found at the next snapshot, which every PreToolUse takes, so an agent always sees a hand edit before writing over it. It follows that changes no hook reports also go to the Human session (`!` shell mode and `/rewind`, if they fire no hooks: unverified).
  - `bashEditDiff` is not a source: it is beta and best effort.
- **Ambiguity:** a change made while two sessions' Bash calls overlap goes to the Human session. No guessing and no repo-wide Bash lock. The agent is told in PostToolUse `additionalContext` which files and lines went to the Human session and why. No action is asked of it.
- **Writing over owned lines:** an agent's Edit or Write on another session's lines is blocked (decided while charting). When Bash or a hand edit rewrites owned lines, they keep their owner. So a repo-wide `cargo fmt` by session 2 leaves session 1's reformatted lines with session 1, and a hand fix to an agent's line goes into the agent's commit.
- **Background commands** (`run_in_background`, or moved to the background at their timeout) claim nothing after they launch. Their later writes are credited by the normal rules.
- **Known limits of v0**, to be named in the spec:
  1. A hand edit saved while an agent's Bash call runs alone is credited to that agent.
  2. A background command that writes tracked files while another session's Bash call runs is credited to that other session.

Handed to other tickets (see their Comments):

- [Warning sessions about areas another session is editing](09-warning-about-areas-in-use.md): should an agent be blocked on Human-owned lines, given that its own output can land there?
- [WIP that changes hands](08-wip-changing-hands.md): handing WIP that went to the Human session back to the right session.
- [Guarding against Lost WIP](07-guarding-against-lost-wip.md): a Bash call deleting or renaming another session's WIP.
- [Changes that can't stand on their own session branch](13-changes-that-cant-stand-alone.md): editing a file another session just created.

Graduated from the fog: [How ownership is recorded and kept current](16-recording-ownership.md).
