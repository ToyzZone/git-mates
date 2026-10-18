# Committing part of the working directory with plain git, safely under concurrency

Type: research
Status: resolved
Blocked by:
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

How can one session commit only the hunks it owns from a shared working directory onto the current branch, using plain git or a Rust git library? The commit must not disturb the shared index or other sessions' WIP. It must also stay safe when two sessions commit at the same moment.

Cover:

- **Building the commit:** a per-session `GIT_INDEX_FILE`; building a tree from HEAD plus selected hunks (`git apply --cached`, `read-tree`, `write-tree`, `commit-tree`); moving the branch with `update-ref <ref> <new> <old>` as a compare-and-swap.
- **Contention:** `index.lock` and `HEAD.lock` contention, and what happens on a lost race.
- **Afterwards:** the state of the shared `.git/index` and the working directory. Do other sessions' WIP still show as modified, and does the committed hunk stop showing?
- **Recording the session in the commit:** trailer conventions (`git interpret-trailers`), the author/committer fields, and how `git log --grep` / `--format=%(trailers)` can filter by session.
- **Rust support today:** what `gix` (gitoxide) and `git2` support for the steps above, and where shelling out to `git` is still needed.
- **Prior art:** how GitButler builds commits from a shared working directory (it is written in Rust).

Use primary sources: git documentation, gitoxide and git2 docs or source, and GitButler source.

## Answer

- **Build (verified):** a private `GIT_INDEX_FILE` → `read-tree HEAD` → `apply --cached <owned hunks>` → `write-tree` → `commit-tree -p <old>` → `update-ref <ref> <new> <old>`. This commits only the Session's hunks and never touches `.git/index` or the working directory. `update-ref` is a real CAS: a lost race gives exit 128 ("is at X but expected Y") and changes nothing. A retry rebuilds on the new tip; if the winner touched the loser's context lines, `--3way` is needed.
- **Do not use porcelain `GIT_INDEX_FILE=tmp git commit`.** It takes its parent from the current HEAD but its tree from the stale temp index, so it silently reverts the other Session's commit (verified).
- **After the commit, the shared index is stale** (`MM`: a staged reversal of the hunk). The next plain `git commit` by any Session reverts the hunk, which is a Mixed commit (verified). Fix: `git apply --cached` the same patch on `.git/index`. Then only other Sessions' WIP shows and their staging is kept. To close the gap between the CAS and the index fix, hold `.git/index.lock` across the CAS (works with plain git via the lockfile-rename protocol).
- **Locks:** a private index locks only `<file>.lock`. Updating the checked-out branch takes `refs/heads/<b>.lock` **and** `HEAD.lock`. Ref locks retry for 100 ms; index locks fail immediately.
- **Session in the commit:** a trailer (`interpret-trailers`/`commit --trailer`), filtered with anchored `log --grep='^Session-Id: X$'` or `%(trailers:key=Session-Id,valueonly)`. There is no native `log --trailer` filter; the unanchored grep over-matches prefixes.
- **Rust:** `git2` covers every step (hunk-selective `apply_to_tree`, CAS `commit`/`reference_matching`, `apply` to the index; verified). `gix` has the tree editor, CAS ref edits and index I/O, but no patch apply and no index→tree. Neither runs hooks (shell out to `git hook run`).
- **GitButler:** builds trees without an index (gix tree editor + its own `apply_hunks`, 3-way merge onto non-HEAD parents) and moves refs with `MustExistAndMatch`. It still uses git2 to sync index and worktree, and its cross-process lock is best-effort only.

Full findings: [research/partial-commits-in-plain-git.md](../../../research/partial-commits-in-plain-git.md)
