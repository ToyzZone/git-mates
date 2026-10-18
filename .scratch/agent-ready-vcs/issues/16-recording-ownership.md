# How ownership is recorded and kept current

Type: prototype
Status: open
Blocked by: 06
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

Ownership is per line, and every hook's snapshot is a checkpoint shared by all sessions ([ADR 0003](../../../docs/adr/0003-own-changes-by-the-line.md)). How is that recorded and kept current?

- **Following lines:** session 1 owns lines 40–60. Session 2 inserts 5 lines above them. Session 1 then commits half its lines, and HEAD is rebuilt. How do the records follow each step, and how are deletion points kept?
- **Storage:** where the per-line records and the snapshots live (inside `.git`, as git objects, a SQLite file, or something else), and how hook processes from many sessions update them at once without losing a write.
- **Snapshots:** what a snapshot stores so that a later diff yields line hunks (for example a tree written through a private index), and when old ones are dropped.

Build a throwaway prototype of the bookkeeping on a toy repo and react to how it behaves.

## Comments

- From [What the workspace looks like to plain git](12-workspace-to-plain-git.md), 2026-10-18: tested on git 2.54.0 that `git branch -m` carries every `branch.<name>.*` key along, custom keys included, and that `git branch -d/-D` deletes the whole section. A session id kept in the session branch's config survives renames made with plain git. An agent's `git checkout -b <name>` renames its session branch.
- From [How a session commits only its own WIP](06-committing-only-own-wip.md), 2026-10-18: when the workspace branch is moved outside the layer (for example the human's `git reset HEAD~`), every session's committed lines suddenly differ from HEAD. Snapshots must compare against the last workspace commit the layer recorded, not HEAD, or those lines get credited as WIP to whoever's call is running (or to the Human session) and swept into that session's next commit. After a commit, the committed lines stop being WIP and leave the records; the layer lock held from pre-commit to post-commit is a natural place for that update.
