# Partial commits in plain git, safely under concurrency

Date: 2026-10-15
Ticket: [02 – Committing part of the working directory with plain git](../.scratch/agent-ready-vcs/issues/02-partial-commits-in-plain-git.md)
Method: primary sources (git-scm.com docs and the local `git help` pages of git 2.54.0, git source at commit [`6de20f6`](https://github.com/git/git/tree/6de20f6092dcf9bdb1c8efe03db4b70c82b423dd), crate sources of `gix` 0.89.0 / `gix-index` 0.57.0 / `gix-ref` 0.69.0 / `gix-lock` 26.0.0 / `gix-object` 0.66.0 / `gix-diff` 0.69.0 / `git2` 0.21.0 / `libgit2-sys` 0.18.8+1.9.7, and GitButler at commit [`03dc9bd`](https://github.com/gitbutlerapp/gitbutler/tree/03dc9bd7a3944269042dcd2b8bb9e1288622e976) from 2026-10-14). I verified the mechanics in throwaway repos with git 2.54.0 (Apple Git-157) on macOS, using the `files` ref backend (`git rev-parse --show-ref-format` → `files`), plus a small `git2` 0.21.0 Rust probe. Anything I could not verify is marked **(unverified)**.

## Summary

- **Building the commit works with plain git plumbing.** A Session can commit only the hunks it owns without touching the shared `.git/index` or the working directory. It builds a tree in a private `GIT_INDEX_FILE` (`read-tree HEAD` → `apply --cached <owned hunks>` → `write-tree`), creates the commit with an explicit parent (`commit-tree -p <old>`), and moves the branch with `update-ref <ref> <new> <old>`. That last step is a real compare-and-swap (CAS). If the race is lost, the command fails with exit 128 and changes nothing.
- **Do not use porcelain `git commit` with a temp index.** `GIT_INDEX_FILE=tmp git commit` takes its parent from HEAD at commit time but its tree from the temp index. If the temp index was built before another session committed, that other commit is silently reverted (verified).
- **After the commit, the shared index is stale.** Plumbing leaves `.git/index` untouched. `git status` then shows the committed hunk as a staged *reversal* (`MM`). The next plain `git commit` by any Session (including the Human session) silently reverts it (verified); in glossary terms, that commit is a Mixed commit. Fix: apply the same patch to the shared index (`git apply --cached`). Afterwards only the other Sessions' WIP shows as modified, and other Sessions' staging is kept (verified).
- **Locks:** a private index uses its own `<file>.lock`, so there is no contention with `.git/index.lock`. `update-ref` on the branch that HEAD points to takes **both** `refs/heads/<b>.lock` **and** `HEAD.lock` (verified, and confirmed in git source). Ref locks are retried for 100 ms by default; index locks fail immediately.
- **Session in the commit:** use a trailer (`git interpret-trailers --trailer` / `git commit --trailer`). Filter with an anchored `git log --grep='^Session-Id: X$'`, or read the value exactly with `--format=%(trailers:key=Session-Id,valueonly)`. git has no native `git log --trailer=` filter (verified).
- **Rust:** `git2` covers every step, including hunk-selective `apply_to_tree`, CAS commit, and `apply` to the index (verified empirically). `gix` has objects, a tree editor, index I/O and CAS ref transactions, but no `git apply`-compatible patch application and no index→tree. Neither runs hooks. GitButler uses gix with its own hunk applier and still uses git2 for checkout and index sync.

---

## 1. Building the commit

### The recipe (verified empirically)

Setup: `a.txt` has lines 1..20. Session A changed line 2 and Session B changed line 19 (two hunks in one file). B also changed `b.txt`. A's patch is the first hunk of `git diff a.txt`, cut out with `awk '/^@@/{n++} n<2'`.

```sh
export GIT_INDEX_FILE=$S/idx-A          # private index, outside the worktree
OLD=$(git rev-parse HEAD)
git read-tree HEAD                      # private index := HEAD's tree
git apply --cached ../A.patch           # add only A's hunk to the private index
TREE=$(git write-tree)
NEW=$(printf 'Change line 2\n\nSession-Id: A-123\n' | git commit-tree $TREE -p $OLD)
unset GIT_INDEX_FILE
git update-ref -m "session A commit" refs/heads/main $NEW $OLD   # CAS
```

Observed: `update-ref exit=0`. `git show HEAD` contains only the `-2/+2-A` hunk, with `Session-Id: A-123` in the message. The SHA-1 of `.git/index` was the same before and after (`417083bb…`), so the shared index was not touched. B's hunk (line 19) and `b.txt` stayed in the working directory.

What each step is documented to do:

- `GIT_INDEX_FILE`: "This environment variable specifies an alternate index file. If not specified, the default of `$GIT_DIR/index` is used." ([git(1)](https://git-scm.com/docs/git#Documentation/git.txt-GITINDEXFILE))
- `git read-tree <tree-ish>`: "Reads the tree information given by <tree-ish> into the index, but does not actually update any of the files it 'caches'." ([git-read-tree](https://git-scm.com/docs/git-read-tree))
- `git apply --cached`: "Apply the patch to just the index, without touching the working tree." ([git-apply](https://git-scm.com/docs/git-apply#Documentation/git-apply.txt---cached))
- `git write-tree`: "Creates a tree object using the current index… The index must be in a fully merged state." ([git-write-tree](https://git-scm.com/docs/git-write-tree))
- `git commit-tree`: "Creates a new commit object based on the provided tree object and emits the new commit object id on stdout." It also says "This is usually not what an end user wants to run directly", and parents are given with `-p`. ([git-commit-tree](https://git-scm.com/docs/git-commit-tree)) It does not move any ref: the ref stayed at `OLD` until `update-ref` ran (verified).
- `git update-ref <ref> <new> <old>`: "stores the <new-oid> in the <ref>, possibly dereferencing the symbolic refs, after verifying that the current value of the <ref> matches <old-oid>." With `-m`, the message goes into the reflog. ([git-update-ref](https://git-scm.com/docs/git-update-ref))
- To move several refs atomically, use `update-ref --stdin` with `start`/`prepare`/`commit`: "If all <ref>s can be locked with matching <old-oid>s simultaneously, all modifications are performed. Otherwise, no modifications are performed." ([git-update-ref](https://git-scm.com/docs/git-update-ref))

Patch details:

- Compute the owned hunks against **HEAD**, not against the index: `git diff HEAD -- <path>` shows "the changes you have in your working tree relative to the named <commit>" ([git-diff](https://git-scm.com/docs/git-diff)). This keeps the result independent of whatever another Session has staged in the shared index.
- A single hunk cut out of a multi-hunk diff keeps the original `index <pre>..<post>` line, so the postimage id is wrong for the partial patch. `git apply --cached` applied it anyway (verified). `--3way` uses only the *preimage* blob (see §2).
- Patches made with `-U0` need `--unidiff-zero`, and the docs discourage them. Hand-edited hunks may need `--recount`. ([git-apply](https://git-scm.com/docs/git-apply))

### Why not `GIT_INDEX_FILE=tmp git commit` (verified empirically)

Porcelain `git commit` does its own CAS on HEAD. In `update_head_with_reflog`, the expected old value is `current_head`, which `git commit` read at commit time ([sequencer.c](https://github.com/git/git/blob/6de20f6092dcf9bdb1c8efe03db4b70c82b423dd/sequencer.c#L1289-L1325), called from [builtin/commit.c](https://github.com/git/git/blob/6de20f6092dcf9bdb1c8efe03db4b70c82b423dd/builtin/commit.c#L1945)). The tree, however, comes from the index. Test: A built a temp index from HEAD₀. B then committed line 19. A then ran `GIT_INDEX_FILE=$S/idx-A2 git commit -qm "A: line 2"`:

```
9cc5806 A: line 2
c0c6b8f B: line 19
53c7f84 init
--- what A's commit did to a.txt:
@@ -16,5 +16,5 @@
-19-B
+19
```

A's commit has parent B but silently reverts B's change. The plumbing route avoids this because the parent is explicit (`-p $OLD`) and `update-ref` checks the same `$OLD`.

### Hooks

`commit-tree` and `update-ref` run no commit hooks. `pre-commit` "is invoked by git-commit(1)" ([githooks](https://git-scm.com/docs/githooks#_pre_commit)). If hooks must run, `git hook run <hook-name>` exists "for use by other scripted git commands" ([git-hook](https://git-scm.com/docs/git-hook)). Hooks such as lint-staged inspect *an* index, so it matters which `GIT_INDEX_FILE` the hook sees **(unverified: behaviour of `git hook run` under a custom `GIT_INDEX_FILE` not tested)**.

## 2. Contention: `index.lock`, `HEAD.lock`, and a lost race

### Which command takes which lock (verified by planting lock files with `touch`)

| Lock held | Command | Result |
|---|---|---|
| `.git/index.lock` | `git apply --cached` on the shared index | `fatal: Unable to create '…/.git/index.lock': File exists.` exit 128 |
| `.git/index.lock` | `git commit --allow-empty` | same error |
| `.git/index.lock` | `GIT_INDEX_FILE=$S/idx-L git read-tree HEAD` | exit 0 (no contention) |
| `.git/index.lock` | `git status` | exit 0 (its index refresh is an optional lock) |
| `.git/index.lock` | `git update-ref refs/heads/main …` | exit 0 |
| `$S/idx-L.lock` | `GIT_INDEX_FILE=$S/idx-L git read-tree HEAD` | `Unable to create '…/idx-L.lock': File exists.` (a private index locks `<file>.lock`) |
| `.git/HEAD.lock` | `git update-ref refs/heads/main $X $CUR` (HEAD → main) | `cannot lock ref 'refs/heads/main': Unable to create '…/.git/HEAD.lock': File exists.` exit 128 |
| `.git/HEAD.lock` | `git update-ref HEAD …` | same |
| `.git/refs/heads/main.lock` | `git update-ref refs/heads/main …` | `Unable to create '…/refs/heads/main.lock': File exists.` exit 128, after 0.154 s wall time |

Sources and explanation:

- **Updating a branch also takes `HEAD.lock`.** When an update targets the ref that HEAD points to, the files backend adds "an extra REF_LOG_ONLY update for HEAD" (`split_head_update` in [refs/files-backend.c](https://github.com/git/git/blob/6de20f6092dcf9bdb1c8efe03db4b70c82b423dd/refs/files-backend.c#L2495-L2545)). That update locks HEAD to write HEAD's reflog. So a Session committing onto the current branch contends with anything holding `HEAD.lock`, such as a checkout.
- **Ref-lock retry:** `core.filesRefLockTimeout` is "The length of time, in milliseconds, to retry when trying to lock an individual reference… Default is 100." `core.packedRefsTimeout` defaults to 1000. ([git-config](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corefilesRefLockTimeout)) The 0.154 s observed above fits a 100 ms retry window.
- **Index-lock retry:** none. The command fails at once (observed above). Lockfiles are created with `O_CREAT|O_EXCL`, so a second writer fails, while "Readers do not block, but they are guaranteed to see either the old contents of the file or the new contents" ([lockfile.h](https://github.com/git/git/blob/6de20f6092dcf9bdb1c8efe03db4b70c82b423dd/lockfile.h)).
- **`git status`** does not need the index lock: with `GIT_OPTIONAL_LOCKS=0`, Git skips "optional sub-operations that require taking a lock. For example, this will prevent `git status` from refreshing the index as a side effect." ([git(1)](https://git-scm.com/docs/git#Documentation/git.txt-GITOPTIONALLOCKS))
- **reftable** is a different ref backend, with different lock files. The CAS semantics of `update-ref` are documented independently of the backend, but I tested only `files` **(unverified for reftable)**.

### A lost race (verified empirically)

B and C both built commits on `OLD`. C's `update-ref` ran first:

```
--- C wins:  exit=0
--- B loses (expects OLD):
fatal: update_ref failed for ref 'refs/heads/main': cannot lock ref 'refs/heads/main': is at facb8ab… but expected 829eac3…
exit=128
--- also through the HEAD symref:
fatal: update_ref failed for ref 'HEAD': cannot lock ref 'HEAD': is at facb8ab… but expected 829eac3…
```

The ref is unchanged. The loser's commit and tree objects remain in the object database but are unreachable. `git gc` prunes such objects after a two-week grace period by default (`gc.pruneExpire`, [git-config](https://git-scm.com/docs/git-config#Documentation/git-config.txt-gcpruneExpire)). Retrying means rebuilding on the new tip: `read-tree <new tip>`, re-apply the same patch, then `commit-tree -p <new tip>` and `update-ref … <new tip>`. That retry succeeded (`retry exit=0`, linear history `B → C → A → init`).

**If the retry patch no longer applies (verified):** when the winner changed a line inside the loser's hunk *context* (winner edited line 4; loser's hunk spans lines 1–5), the retry failed:

```
--- plain --cached:   error: patch failed: a.txt:1 / error: a.txt: patch does not apply   exit=1
--- --cached --3way:  Applied patch to 'a.txt' cleanly.                                     exit=0   (result: 2-A and 4-B both present)
--- -C1:              Context reduced to (1/1) to apply fragment at 1                        exit=0
```

`--3way` works when "the patch records the identity of blobs it is supposed to apply to and we have those blobs available locally". With `--cached`, "any conflicts are left at higher stages in the cache" ([git-apply](https://git-scm.com/docs/git-apply#Documentation/git-apply.txt---3way)). `-C<n>` lowers how much context must match.

## 3. Afterwards: the shared index and the working directory

### State right after the plumbing commit (verified)

The working directory is untouched; plumbing never writes it. The shared `.git/index` still holds the *old* blob for `a.txt`, so:

```
$ git status --short
MM a.txt
 M b.txt
$ git diff --cached        # index vs new HEAD: A's hunk shows as a staged REVERSAL
-2-A
+2
$ git diff                 # worktree vs index: both A's and B's hunks still show
```

So, as things stand, the committed hunk *does not* stop showing. It appears twice, as a staged reversal and as an unstaged change.

**What this stale index causes (verified):** in a copy of that state, someone ran `git add b.txt && git commit -m "B commits b.txt"`. The resulting commit contained `b.txt` **and** `-2-A / +2`, so it silently reverted A's committed hunk. That commit is a Mixed commit in glossary terms: it contains a change (the reversal) that the committing Session does not own.

### Fix: apply the same patch to the shared index (verified)

```sh
git apply --cached ../A.patch      # on the real .git/index (takes .git/index.lock)
git status --short
#  M a.txt          <- only B's line-19 hunk remains as a diff
#  M b.txt
```

After the fix, other Sessions' WIP still shows as modified, and the committed hunk no longer shows.

Compared with `git reset -q -- a.txt` (verified): suppose the Human session had already staged B's hunk.

- Re-applying A's patch to the shared index gives `M  a.txt` with B's hunk **still staged**.
- `git reset -- a.txt` gives ` M a.txt` with **nothing staged**. B's staging is lost, though the working directory still has the change.

### Closing the window between `update-ref` and the index fix (verified)

Between the CAS and the index fix there is a window. During it, another Session's plain `git commit` can grab `index.lock`, read the new HEAD as parent, and write the stale index, which reverts A. Plain git can hold the shared index lock *across* the ref update by using git's own lockfile protocol: write `<file>.lock`, then rename it over `<file>` ([lockfile.h](https://github.com/git/git/blob/6de20f6092dcf9bdb1c8efe03db4b70c82b423dd/lockfile.h)):

```sh
set -o noclobber; cat .git/index > .git/index.lock   # O_EXCL create = take the lock
git commit -q --allow-empty -m x                     # -> fatal: Unable to create '.../index.lock': File exists.
git update-ref -m "A commit" refs/heads/main $NEW $OLD
GIT_INDEX_FILE=.git/index.lock git apply --cached ../A3.patch   # git locks index.lock.lock for this
mv .git/index.lock .git/index                        # publish + unlock
git status --short   # ->  M a.txt   (only B's hunk)
```

Two constraints come with this. While the lock is held, the Human session's `git add`/`git commit`, and any IDE that writes the index, fail at once (index locks do not retry). And a crashed holder leaves a stale `index.lock` behind (git's message: "or the lock file may be stale").

### Index stat data

An entry rewritten by `git apply --cached` gets new content but no matching stat info. In my tests `git status` was still correct. GitButler deliberately zeroes stat data when it rewrites index entries, with this comment: "it's needed to set the values to 0 here or else 1 in 40 times or so git status will report the file didn't change even though it did" ([commit_engine/index.rs L36-L40](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-workspace/src/commit_engine/index.rs#L36-L40)). I did not observe this race **(unverified)**.

## 4. Recording the Session in the commit

### Trailers

- "A trailer in its simplest form is a key-value pair with a colon as a separator. The key consists of ASCII alphanumeric characters and hyphens (-)." The trailer block must be "preceded by a blank line" and is "by definition at the end of the commit message". A block counts as trailers if it "(i) is all trailers, or (ii) contains at least one Git-generated or user-configured trailer and consists of at least 25% trailers". ([git-interpret-trailers](https://git-scm.com/docs/git-interpret-trailers))
- `--trailer <k>:<v>` adds a trailer, `--parse` extracts trailers, `--if-exists replace|add|addIfDifferent|…` handles duplicates, and `--where` sets placement (same page). `git commit --trailer "<k>:<v>"` exists too, and "The trailer.* configuration variables… can be used" ([git-commit](https://git-scm.com/docs/git-commit#Documentation/git-commit.txt---trailerlttokengtltvaluegt)).
- Verified: `printf 'Fix parser\n\nBody text.\n' | git interpret-trailers --trailer 'Session-Id: 7f3a' --trailer 'Co-authored-by: Claude <noreply@anthropic.com>'` appended both trailers after a blank line. `--if-exists replace --trailer 'Session-Id: 9999'` replaced the value. `git commit --trailer "Session-Id: abc"` also worked.
- Conventions in the wild:
  - GitHub's co-author trailer is `Co-authored-by: name <name@example.com>`, one per co-author, after an empty line ([GitHub docs](https://docs.github.com/en/pull-requests/committing-changes-to-your-project/creating-and-editing-commits/creating-a-commit-with-multiple-authors)).
  - Claude Code adds its own commit trailer, configurable or hideable via the `attribution.commit` setting ("Change or hide the trailer Claude Code adds to commits", [settings reference](https://code.claude.com/docs/en/settings-reference)).
  - Every Claude Code hook receives `session_id` ("Current session identifier") in its stdin JSON ([hooks](https://code.claude.com/docs/en/hooks)).

### Filtering by Session (verified)

Four commits: `Session-Id: 7f3a`, `Session-Id: 17f3a0`, `Session-Id: 7f3a0`, and one with no trailer.

| Query | Result |
|---|---|
| `git log --format='%h %s [%(trailers:key=Session-Id,valueonly,separator=%x2C)]'` | prints each commit's exact value: `[17f3a0]`, `[7f3a]`, `[]` |
| `git log --grep='Session-Id: 7f3a'` | matched `7f3a` **and** `7f3a0` (unanchored regex, prefix over-match) |
| `git log --grep='^Session-Id: 7f3a$'` | matched only `7f3a` (`^`/`$` anchor per line, even mid-message) |
| `git log --trailer=x` | `fatal: unrecognized argument: --trailer=x` (no native trailer filter) |
| `git log --author='session 7f3a'` | matched the commit whose `GIT_AUTHOR_NAME="Claude (session 7f3a)"` |

- `--grep=<pattern>`: "Limit the commits output to ones with a log message that matches the <pattern> regular expression", with `--all-match` to AND several patterns ([git-log](https://git-scm.com/docs/git-log#Documentation/git-log.txt---grepltpatterngt)). It searches the whole message, so a body line that happens to look like a trailer also matches.
- `%(trailers:key=<key>)`: "Matching is done case-insensitively and trailing colon is optional", plus `valueonly` and `separator` ([pretty-formats](https://git-scm.com/docs/pretty-formats)). It only looks at the parsed trailer block, so exact filtering means formatting with it and then filtering the output.

### Author / committer fields

`GIT_AUTHOR_NAME` / `GIT_AUTHOR_EMAIL` (and the `GIT_COMMITTER_*` variables) set "the author identity when creating commit or tag objects, or when writing reflogs", and they override `user.name`/`user.email` ([git(1)](https://git-scm.com/docs/git#Documentation/git.txt-GITAUTHORNAME)). `commit-tree` honours them (verified: `%an` = `Claude (session 7f3a)`, `%cn` = `Tester` from config). `git log --author=` and `--committer=` filter on them as regexes ([git-log](https://git-scm.com/docs/git-log)).

## 5. Rust support today

| Step | `git2` 0.21.0 (libgit2 bundled in libgit2-sys 0.18.8+1.9.7) | `gix` 0.89.0 |
|---|---|---|
| Private index | `Index::new()` (in-memory), `Index::open(path)`, `Index::read_tree` ([docs](https://docs.rs/git2/0.21.0/git2/struct.Index.html)) | `Repository::index_from_tree(&tree)` returns an in-memory `gix_index::File` ([source](https://docs.rs/crate/gix/0.89.0/source/src/repository/index.rs)) |
| Apply selected hunks | `Repository::apply_to_tree(tree, diff, opts)` returns an `Index`. `ApplyOptions::hunk_callback(FnMut(Option<DiffHunk>) -> bool)`: `true` applies the hunk, `false` skips it. `Repository::apply(diff, ApplyLocation::Index, …)`, where Index is "the equivalent of `git apply --cached`" ([apply.h](https://docs.rs/crate/libgit2-sys/0.18.8+1.9.7/source/libgit2/include/git2/apply.h)). `Diff::from_buffer` parses patch text. | **None.** gitoxide's status page lists `git-apply` compatibility and `stash, am and apply` as not done ([crate-status.md](https://github.com/GitoxideLabs/gitoxide/blob/8dc7afdeed215558e89f0c49fd87621d46a98307/crate-status.md)). `gix-diff` can *produce* blob diffs (`blob::UnifiedDiff`, imara-diff), but the caller must apply the hunks to the base blob itself. |
| Index → tree | `Index::write_tree_to(&repo)` | **"tree from index" not done** ([crate-status.md](https://github.com/GitoxideLabs/gitoxide/blob/8dc7afdeed215558e89f0c49fd87621d46a98307/crate-status.md)). Use the tree editor instead: `Repository::edit_tree(id)`, then `upsert(path, kind, blob_id)` / `remove` / `write()` ([source](https://docs.rs/crate/gix/0.89.0/source/src/object/tree/editor.rs)) |
| Commit object | `Repository::commit(update_ref, …)`: "If it does exist, the first parent must be the tip of this branch". Otherwise `GIT_EMODIFIED` "current tip is not the first parent" ([libgit2 commit.c](https://docs.rs/crate/libgit2-sys/0.18.8+1.9.7/source/libgit2/src/libgit2/commit.c)). `commit_create_buffer` builds a commit without writing it. | `Repository::commit(reference, …)`: "The first parent id in `parents` is expected to be the current target of `reference` and the operation will fail if it is not". `new_commit[_as]` writes a commit object without touching refs ([source](https://docs.rs/crate/gix/0.89.0/source/src/repository/object.rs)) |
| CAS ref move | `Repository::reference_matching(name, id, force, current_id, msg)`, which "will return GIT_EMODIFIED if the reference's value at the time of updating does not match". `Reference::set_target` passes the value it read as the expected old value ([libgit2 refs.c](https://docs.rs/crate/libgit2-sys/0.18.8+1.9.7/source/libgit2/src/libgit2/refs.c)) | `edit_reference(s)` with `RefEdit` + `PreviousValue::MustExistAndMatch(target)` ("The ref _must_ exist and have the given value", [gix-ref](https://docs.rs/crate/gix-ref/0.69.0/source/src/transaction/mod.rs)). `commit()` uses `MustExistAndMatch` for `HEAD` and `ExistingMustMatch` for other refs. |
| Lock behaviour | Index write with `index.lock` present fails at once: `code=Locked`, "the index is locked; this might be due to a concurrent or crashed process". libgit2's `lock_file` returns `GIT_ELOCKED` if the `.lock` exists, with no retry ([filebuf.c](https://docs.rs/crate/libgit2-sys/0.18.8+1.9.7/source/libgit2/src/util/filebuf.c)) | Ref locks honour `core.filesRefLockTimeout`/`core.packedRefsTimeout` with backoff, defaulting to 100 ms / 1000 ms ([config/cache/access.rs](https://docs.rs/crate/gix/0.89.0/source/src/config/cache/access.rs)). `gix_index::File::write` uses `Fail::Immediately` ([write.rs](https://docs.rs/crate/gix-index/0.57.0/source/src/file/write.rs)). The doc comment on `Repository::commit` still says it "fails immediately if a `reference` lock can't be acquired", but the code path goes through the config-driven timeout. |
| `HEAD.lock` on branch update | not checked **(unverified)** | `gix-ref`'s file transaction code has no HEAD special case (grep for `"HEAD"` in `store/file/transaction/` finds nothing). This suggests no `HEAD.lock` and no HEAD reflog entry when a branch is updated directly **(unverified empirically)** |
| Update shared index after commit | `Repository::apply(diff, ApplyLocation::Index, …)` (GitButler uses exactly this; see §6) | Manual: edit `State` entries and write. **Caveat:** `File::write` writes the TREE (cache-tree) extension "as-is… a stale-but-valid tree-cache can make a later commit capture outdated subtree content", so callers must `remove_tree()` first ([write.rs](https://docs.rs/crate/gix-index/0.57.0/source/src/file/write.rs), [issue #2421](https://github.com/GitoxideLabs/gitoxide/issues/2421)) |
| Trailers | parse: `git2::message_trailers_strs(msg)` (verified: `trailer Session-Id=A-git2`). Writing: no `interpret-trailers` equivalent found, so the caller formats the string. | parse: `gix_object::commit::message::BodyRef::trailers()` ([body.rs](https://docs.rs/crate/gix-object/0.66.0/source/src/commit/message/body.rs)). Writing: none found. |
| Status | `Repository::statuses` | `Repository::status()` ([source](https://docs.rs/crate/gix/0.89.0/source/src/status/mod.rs)) |
| Run hooks | **No.** libgit2 only knows the hooks directory path (`GIT_REPOSITORY_ITEM_HOOKS`) and has no hook-running code (grep of `src/`, `include/`). The separate `git2-hooks` crate exists (GitButler uses it). | **No.** "[ ] execute hooks" ([crate-status.md](https://github.com/GitoxideLabs/gitoxide/blob/8dc7afdeed215558e89f0c49fd87621d46a98307/crate-status.md)). gix only writes `*.sample` templates on `init`. |

**git2 probe (verified empirically):** a ~40-line `git2 = "=0.21.0"` program was run against a repo with A's hunk at line 2 and B's at line 19. It did `diff_tree_to_workdir(HEAD)`, then `apply_to_tree` with `hunk_callback(|h| h.old_start()==1)`, then `write_tree_to`, then `commit(Some("HEAD"), …, &[&head])`. Output:

```
committed e2aabeb9…
stale-parent commit: code=Modified msg=failed to create commit: current tip is not the first parent
reference_matching: code=Modified msg=old reference value does not match
trailer Session-Id=A-git2
index write with lock held: code=Locked msg=the index is locked; this might be due to a concurrent or crashed process
```

`git show HEAD` contained only `-2/+2-A`. As with the CLI path, `git status` then showed `MM a.txt`: the shared index was stale and needed the same fix as in §3.

**When you still have to shell out to `git`:**

- running commit hooks (`git hook run`), unless you use a third-party crate;
- `git apply`-compatible patch application from **gix** alone (git2 covers it);
- reftable repositories: gitoxide lists "[ ] support for different backends like `files` and `reftable`" ([crate-status.md](https://github.com/GitoxideLabs/gitoxide/blob/8dc7afdeed215558e89f0c49fd87621d46a98307/crate-status.md)). libgit2's reftable support is **(unverified)**.

## 6. Prior art: how GitButler builds commits from a shared working directory

All links point to commit [`03dc9bd`](https://github.com/gitbutlerapp/gitbutler/tree/03dc9bd7a3944269042dcd2b8bb9e1288622e976).

- **Selection unit.** `DiffSpec { previous_path, path, hunk_headers }`: "If `hunks` is empty, this means the current content of the file should be committed" ([diff_types.rs](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-core/src/diff_types.rs#L6-L24)). The `context_lines` used to make the specs must match the value passed when committing ([commit_create.rs L46-L49](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-workspace/src/commit/commit_create.rs#L46-L49)). Sub-hunk line selections are supported ([commit_engine/mod.rs L107-L122](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-workspace/src/commit_engine/mod.rs#L107-L122)).
- **No index at all when building the tree.** `create_tree` takes `HEAD^{tree}` as the base. `apply_worktree_changes` then:
  1. diffs the worktree,
  2. matches the requested hunks to it,
  3. runs GitButler's own `apply_hunks(old_image, new_image, hunks)` ("apply these hunks to `old_image` and return the newly constructed image"),
  4. writes the blob with `repo.write_blob`,
  5. puts it into a **gix tree editor** (`base_tree.edit()` → `upsert` → `write()`).

  ([tree/mod.rs L75-L167, L185-L372](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-core/src/tree/mod.rs#L75-L372), [hunks.rs L264-L272](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-core/src/hunks.rs#L264-L272)).
- **Other parents use a 3-way tree merge.** If the destination's tree is not HEAD's tree, the changed tree is cherry-picked onto it with `repo.merge_trees(base=HEAD tree, ours=target, theirs=changed)`. Conflicting paths become `RejectionReason::CherryPickMergeConflict`, and the commit is retried without them, so it is not an all-or-nothing failure ([tree/mod.rs L122-L155](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-core/src/tree/mod.rs#L122-L155)). Hunks that no longer match the worktree are also rejected ([tree/mod.rs L46-L47, L305-L309](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-core/src/tree/mod.rs)).
- **Commit creation does not move refs.** `create_commit` takes an explicit parent, and "No reference is touched in the process" ([commit_engine/mod.rs L103-L105](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-workspace/src/commit_engine/mod.rs#L94-L128)). Refs are moved later by the graph-rebase editor, with `RefEdit::update(…, PreviousValue::MustExistAndMatch(<value read during rebase>))` ([rebase.rs L150-L155](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-rebase/src/graph_rebase/rebase.rs#L150-L155)), applied via gix `edit_references` ([materialize.rs](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-rebase/src/graph_rebase/materialize.rs)). In the same step, `HEAD` itself is set with `PreviousValue::Any`.
- **Index and worktree sync still use git2.** `safe_checkout_from_head` "will also update the `.git/index` to match the `new_head_id^{tree}`". It does this with `git2_repo.apply(&diff, git2::ApplyLocation::Index, None)` and falls back to `index.read_tree(&tree); index.write()` ([checkout/function.rs L15-L70](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-core/src/worktree/checkout/function.rs#L15-L70)). A gix-based `sync_index_to_tree` exists too; it nulls stat data and calls `remove_tree()` before writing ([commit_engine/index.rs](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-workspace/src/commit_engine/index.rs)).
- **Cross-process concurrency is best-effort.** `exclusive_worktree_access` blocks other writers in the same process. It also "attempt[s] to obtain a best-effort inter-process file lock, but failures… are logged and ignored, so the hard guarantee remains in-process exclusivity only" ([but-ctx/src/access.rs L21-L31](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/but-ctx/src/access.rs#L21-L31)).
- **Hooks temporarily rewrite the shared index.** To run `pre-commit`, GitButler copies `.git/index` to a backup, writes the commit's tree into `.git/index`, runs the hook via the `git2-hooks` crate, and then restores the backup. A sidecar lock `index.gitbutler-hook-lock` guards this ([gitbutler-repo/src/hooks.rs L91-L170](https://github.com/gitbutlerapp/gitbutler/blob/03dc9bd7a3944269042dcd2b8bb9e1288622e976/crates/gitbutler-repo/src/hooks.rs#L91-L170)). That sidecar is not git's `index.lock`, so plain git processes are not blocked during the hook **(unverified empirically)**.

## Implications

Facts and constraints only.

1. **The commit can be built with plain git** while the shared index and working directory stay untouched: a private `GIT_INDEX_FILE`, `read-tree`/`apply --cached`/`write-tree`, `commit-tree -p <old>`, then `update-ref <ref> <new> <old>`. Two conditions make it safe: the parent is explicit, and the CAS checks that same parent.
2. **Porcelain `git commit` with a temp index is not safe** under concurrency. It reverts commits made after the temp index was built.
3. **Every plumbing commit leaves the shared `.git/index` stale.** Unless the shared index is patched afterwards, the next plain `git commit` by any Session (including the Human session) reverts the committed hunk. Patching needs `.git/index.lock`, which fails at once if anything (the human, an IDE, another Session) holds it.
4. **The window between CAS and index patch** closes only if `.git/index.lock` is held across the CAS. Plain git can do this through the lockfile-rename protocol. While it is held, every other index writer fails.
5. **Committing onto the checked-out branch locks both** `refs/heads/<b>.lock` and `HEAD.lock` in git's files backend. Ref locks retry for 100 ms by default; index locks never retry.
6. **A lost race is detectable and loses nothing** (exit 128, ref unchanged). A retry must rebuild on the new tip. If the winner touched the loser's context lines, plain `apply` fails, while `--3way` or a smaller `-C` can still apply.
7. **Plumbing, gix and libgit2 run no commit hooks.** Running them needs `git hook run` or a third-party crate. Hooks read an index, so which index they see is a design constraint.
8. **Session attribution via a trailer** is native to git (`interpret-trailers`, `commit --trailer`, `%(trailers)`). Filtering needs anchored `--grep` or parsing `%(trailers)` output, since there is no `log --trailer` filter. Claude Code hooks expose `session_id`.
9. **In Rust:** `git2` alone covers build, hunk selection, CAS and shared-index patching. `gix` lacks patch application and index→tree, so it needs a custom hunk applier plus the tree editor (GitButler's approach). gix index writes also require `remove_tree()`.
10. **GitButler** builds commits without an index (gix tree editor + its own hunk applier) and uses CAS on refs. It still relies on git2 for index/worktree sync, and its cross-process locking is only best-effort.
