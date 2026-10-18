# Guarding against Lost WIP

Type: grilling
Status: open
Blocked by: 01, 05
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

Sometimes a session runs a git command that would discard WIP owned by another session. Examples: `git stash`, `git checkout -- <file>`, `git restore`, `git reset --hard`, `git clean`. What should v0 do?

- **(a) Block** the command with a message to the agent.
- **(b) Narrow** the command so it only acts on the running session's own WIP.

Also decide exactly which commands and flags are on the guarded list, and whether the Human session gets the same protection or an override.

## Comments

- From [Unit of ownership and how every change gets attributed](05-unit-of-ownership.md), 2026-10-18: when Bash or a hand edit rewrites another session's lines, they keep their owner. But a Bash call that deletes those lines, or deletes or renames a file holding another session's WIP, can't be stopped in advance (its paths aren't known) and shows up only afterwards, in the snapshot diff. Decide what v0 does then, alongside the git commands already listed.
- From [How a session commits only its own WIP](06-committing-only-own-wip.md), 2026-10-18: an agent's history command that slips past PreToolUse (for example `git reset --hard` inside a script) is noticed only afterwards, when a snapshot sees the workspace branch moved. The layer then puts the branch and index back, but files `--hard` overwrote are already gone. If snapshots keep file contents ([16](16-recording-ownership.md)), the snapshot taken before that call could restore them: decide whether v0 does. The human's `reset --hard` can't be stopped by any hook: `reference-transaction` rejects only after the files are rewritten (verified, [research](../../../research/git-hooks-for-session-commits.md)).
