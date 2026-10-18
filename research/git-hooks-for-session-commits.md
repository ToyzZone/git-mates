# Git hooks for session commits on a shared workspace branch

Date: 2026-10-18
Ticket: [06-committing-only-own-wip](../.scratch/agent-ready-vcs/issues/06-committing-only-own-wip.md)
Method: experiments in throwaway repos under the session scratchpad (`.../scratchpad/githooks-spike`) with **git 2.54.0 (Apple Git-157)** on macOS, `files` ref backend (`git rev-parse --show-ref-format` → `files`). The user's global config sets `core.hooksPath=~/.git-hooks`, so every experiment ran with `GIT_CONFIG_GLOBAL=<spike>/gitconfig GIT_CONFIG_NOSYSTEM=1 GIT_EDITOR=true` and hooks in `.git/hooks`. Q9 comes from the docs only (`https://code.claude.com/docs/en/hooks.md`, fetched 2026-10-18).

Shared fixture (`mkws.sh`): `main` = base; `s1` = base + a change to `c.txt`; `s2` = base + a change to `d.txt`; `ws` = W = `git commit-tree $(git merge-tree --write-tree s1 s2) -p main -p s1 -p s2 -m workspace`; HEAD on `ws`.

## Summary

- **Q1, pre-commit modifying the index: VERIFIED, but not for `--only`.** The commit always uses the index as the hook left it (`git apply --cached -R`, `git reset -q -- f` and `git update-index --cacheinfo` all work, with no lock errors). `GIT_INDEX_FILE` is `.git/index` for plain commit, `.git/index.lock` for `-a` and `-i`, and `.git/next-index-<pid>.lock` for `--only`/pathspec. After plain, `-a` and `-i` commits, the stripped lines are unstaged (a stripped new file becomes untracked). After `--only`, the stripped lines are **still staged** in the real index, and pre-commit cannot fix that because `.git/index.lock` is held. Extra finding: if the hook strips everything, git still creates an **empty commit**.
- **Q2, repointing HEAD inside pre-commit: VERIFIED.** The commit lands on `ws`, `main` is unchanged, and git prints `[ws <sha>]`. The only oddity is a HEAD reflog entry with an empty message (from `symbolic-ref` without `-m`). It works only if `ws` points at the same commit HEAD pointed at. If `ws` already exists elsewhere, the commit fails with `cannot lock ref 'HEAD': is at X but expected Y`.
- **Q3, `--no-verify`: VERIFIED.** It skips pre-commit and commit-msg. prepare-commit-msg and post-commit still run.
- **Q4, post-commit moving the branch: VERIFIED.** `commit-tree` plus `update-ref refs/heads/ws W' C` gives no error. No `index.lock` is held in post-commit, and `git apply --cached` and `git update-index` on the real index work. `git status` stays clean. Hook stdout and stderr both go to git's **stderr** and appear **before** `[ws <sha>] msg / N files changed`. That summary line shows C's sha, not W'.
- **Q5, amend: VERIFIED.** prepare-commit-msg runs every time: `--no-edit` gives `$2=commit $3=HEAD`, `-m new` gives `$2=message` and no `$3`, and plain `--amend` with `GIT_EDITOR=true` gives `$2=commit $3=HEAD`. The overwritten message is used even with `--no-edit` and `-m`. W's 3 parents are kept. post-commit runs, and post-rewrite runs with arg `amend` and stdin `<old> <new>`.
- **Q6, checkout -b / switch -c: VERIFIED.** post-checkout gets `<old> <new> 1`, with old == new. "Switched to a new branch 'x'" is printed before the hook. Running `symbolic-ref HEAD refs/heads/ws` plus `branch -D x` (or `branch -m`) inside the hook works with no error, and the result is `On branch ws` with WIP untouched. `checkout -b`/`switch -c` of an existing branch fails with `fatal: a branch named 's1' already exists` before any hook runs. Caveat: with a start point other than `ws`, the snap-back leaves the index and worktree at the other commit's tree.
- **Q7, which commands fire commit hooks: VERIFIED** (table below). cherry-pick, `revert --no-edit` and each rebase pick run only prepare-commit-msg and post-commit, never pre-commit or commit-msg. Rebase also runs post-rewrite. Merge and pull run pre-merge-commit, prepare-commit-msg, commit-msg and post-merge, but **no pre-commit and no post-commit**. stash runs no commit hooks. reference-transaction fires for all of them, including stash moving `refs/heads/<current>` to the same oid.
- **Q8, reference-transaction as a guard: VERIFIED, with hazards.** A `prepared`-state hook that exits 1 on `refs/heads/ws` (unless `LAYER_INTERNAL` is set) blocks `reset --soft` and `commit`. The user sees `fatal: in 'prepared' phase, update aborted by the reference-transaction hook`, and the index and worktree are unchanged. Hazards: (a) same-oid updates must be allowed, or `git stash` and a no-op `git reset` fail; (b) `reset --mixed` and `reset --hard` rewrite the index and worktree **before** the ref update, so the guard leaves HEAD and index inconsistent, and with `--hard` **WIP is lost**.
- **Q9, PATH prepend via CLAUDE_ENV_FILE: VERIFIED.** The docs say "Each Bash command runs the file's contents as shell code before the command itself, so a line there can use anything Bash evaluates" and show a PATH edit as the example. The docs show an append (`$PATH:./node_modules/.bin`), not a prepend. A prepend uses the same mechanism. Not covered: PowerShell commands receive none of the variables if the file contains `$PATH`.

---

## 1. pre-commit modifying the index

Setup (`q1.sh`): 30-line `a.txt`, plus `c.txt` and `d.txt`, committed. WIP: "mine" = `a.txt` line 3 and `d.txt`; "other" = `a.txt` line 25, new `b.txt`, and `c.txt`. Everything is staged with `git add -A`. `.git/other.patch` is the diff of the line-25 change alone. The hook:

```sh
echo "GIT_INDEX_FILE=${GIT_INDEX_FILE-<unset>}"; ls .git/*.lock
git apply --cached -R .git/other.patch                       # strip one hunk
git reset -q -- b.txt                                         # strip a new file
git update-index --cacheinfo "100644,$(git rev-parse HEAD:c.txt),c.txt"   # strip a modified file
```

| Command | `GIT_INDEX_FILE` in hook | Locks during hook | Committed | `git status --short` afterwards |
|---|---|---|---|---|
| `git commit -m x` | `.git/index` (relative) | none | `a.txt` (line 3 only), `d.txt` | ` M a.txt`, ` M c.txt`, `?? b.txt` (stripped lines **unstaged**) |
| `git commit -a -m x` | `<abs>/.git/index.lock` | `index.lock` | same | same (unstaged) |
| `git commit -m x -- a.txt b.txt c.txt d.txt` | `<abs>/.git/next-index-96272.lock` | `index.lock`, `next-index-96272.lock` | same | `M  a.txt`, `A  b.txt`, `M  c.txt`: stripped lines **still staged** (`-25 / +25 OTHER`, `+b-other`, `-c-orig / +c-other`) |
| `git commit -i -m x a.txt b.txt c.txt d.txt` | `<abs>/.git/index.lock` | `index.lock` | same | same as plain (unstaged) |

`HEAD:a.txt` lines 3 and 25 were `3 MINE` and `25` in all four cases. None of the three index edits hit an `index.lock` error: each tool locks `$GIT_INDEX_FILE.lock`, which is a different file.

Extra probes:
- With `--only`, pre-commit tried `GIT_INDEX_FILE=<abs>/.git/index git update-index --refresh` and got `fatal: Unable to create '.../.git/index.lock': File exists.` (rc 128). The real index cannot be corrected from pre-commit in that mode. It can be corrected in post-commit (see Q4).
- If the hook strips everything (`git reset -q -- c.txt` when only `c.txt` was staged), `git commit -m x` exits 0 and creates a commit whose tree equals its parent's tree (an empty commit, no stat output). The hook must exit non-zero itself when nothing of the session's remains.

**Verdict: VERIFIED** for plain, `-a` and `-i`. For `--only`/pathspec the commit is correct, but the real index keeps the stripped lines staged.

## 2. Repointing HEAD inside pre-commit

```sh
# HEAD on main, staged d.txt change; pre-commit:
git branch ws && git symbolic-ref HEAD refs/heads/ws
git commit -m x
```

Observed:
```
[ws 6b2371d] x
 1 file changed, 1 insertion(+), 1 deletion(-)
main: 8c55048 (base 8c55048)  ws: 6b2371d  HEAD: refs/heads/ws   ws^ = 8c55048
reflog HEAD: 6b2371d HEAD@{0}: commit: x | 8c55048 HEAD@{1}: (empty message) | 8c55048 HEAD@{2}: commit (initial): base
reflog main: only "commit (initial): base"
reflog ws:   6b2371d ws@{0}: commit: x | 8c55048 ws@{1}: branch: Created from main
```
There was no error. The empty HEAD reflog message comes from `symbolic-ref` without `-m`. Use `git symbolic-ref -m <msg>` (Q6 shows `-m "snap back"` working).

Variant: `ws` already exists at a different commit (an empty commit W on top of base) and pre-commit only runs `git symbolic-ref HEAD refs/heads/ws`. Result: `fatal: cannot lock ref 'HEAD': is at e042dcb… but expected 4e05d07…`, rc 128. HEAD is left on `ws`, `main` is unchanged, and the change stays staged. git reads the parent (and the expected old value) from HEAD before the hooks run.

**Verdict: VERIFIED** when `ws` is created at the current HEAD commit. It fails safely (CAS error) when `ws` points elsewhere.

## 3. `--no-verify`

All four hooks append their name and args to a log file.

| Command | Hooks run |
|---|---|
| `git commit -a -m x` | `pre-commit`, `prepare-commit-msg .git/COMMIT_EDITMSG message`, `commit-msg .git/COMMIT_EDITMSG`, `post-commit` |
| `git commit -a --no-verify -m y` | `prepare-commit-msg … message`, `post-commit` |
| `git commit -a -n -m z` | `prepare-commit-msg … message`, `post-commit` |

**Verdict: VERIFIED.** `--no-verify` skips pre-commit and commit-msg, and prepare-commit-msg and post-commit still run. So a post-commit that rebuilds W' still runs, but the stripping in pre-commit does not.

## 4. post-commit moving the branch

On the fixture (HEAD = ws = W), the post-commit hook:

```sh
C=$(git rev-parse HEAD); W=$(git rev-parse HEAD^)
git diff $W $C > $gd/c.patch
git apply --cached -R $gd/c.patch && git apply --cached $gd/c.patch   # write real index twice
git update-index --refresh
GIT_INDEX_FILE=$gd/tmp-index: git read-tree s1; git apply --cached c.patch; T=$(git write-tree)
S1=$(git commit-tree $T -p s1 -m "$(git log -1 --format=%B $C)"); git update-ref refs/heads/s1 $S1 <old s1>
W2=$(git commit-tree $C^{tree} -p main -p s1 -p s2 -m workspace)
git update-ref -m "rebuild workspace" refs/heads/ws $W2 $C
```

Observed for `git commit -am mine`:
```
[post-commit stdout] GIT_INDEX_FILE=.git/index locks:
[post-commit stderr] hello
[post-commit] apply --cached on real index OK
[post-commit] update-index OK
[post-commit] s1 -> b5a020b
[post-commit] update-ref ws rc=0 -> 329418f
[ws 4eaef21] mine
 1 file changed, 1 insertion(+), 1 deletion(-)
```
- `git status` showed `On branch ws / nothing to commit, working tree clean`, and `ws^{tree}` equals C's tree.
- The graph shows W' (`329418f`) with parents `base`, `s1'` (`b5a020b mine` on top of `s1-work`) and `s2-work`. C (`4eaef21`) is no longer on any branch and survives only in the reflog.
- The summary line shows C's sha (`4eaef21`), not W'.
- The same results held with `--only` (`git commit -m two -- a.txt`): `GIT_INDEX_FILE=.git/index` and no locks in post-commit.

Streams: `git commit -m two -- a.txt 2>/dev/null` showed only `[ws 77942ac] two` and the stat line. `git commit … >/dev/null` showed only the hook lines. So git sends hook stdout and stderr to its own stderr, and both appear before git's stdout summary.

**Verdict: VERIFIED.**

## 5. Amend

On the fixture (HEAD = W with 3 parents `92e5632 3f56189 f3e7f69`). prepare-commit-msg logs its args and then overwrites `$1` with `OVERWRITTEN by hook`. commit-msg, post-commit and post-rewrite (args plus stdin) log too. `GIT_EDITOR=true`.

| Command | prepare-commit-msg args | Final message | Parents kept | Other hooks |
|---|---|---|---|---|
| `git commit --amend --no-edit` | `$2=commit $3=HEAD` (incoming msg `workspace`) | `OVERWRITTEN by hook` | yes | commit-msg, post-commit, `post-rewrite amend` stdin `366c4a8… 8439064…` |
| `git commit --amend -m new` | `$2=message`, no `$3` (incoming msg `new`) | `OVERWRITTEN by hook` | yes | commit-msg, post-commit, `post-rewrite amend` stdin `<old> <new>` |
| `git commit --amend` | `$2=commit $3=HEAD` | `OVERWRITTEN by hook` | yes | commit-msg, post-commit, `post-rewrite amend` |

In the third run the new sha equals the old one, because it had identical content and the same-second timestamp as the second run. post-rewrite still fired with `old == new`.

**Verdict: VERIFIED.** The hook can force the message even with `--no-edit` or `-m`. Amend keeps W's parents, and both post-commit and post-rewrite (`amend`, stdin `<old-sha> <new-sha>`) run.

## 6. `checkout -b` / `switch -c`

On the fixture, with staged and unstaged WIP in `a.txt` plus an untracked file. The post-checkout hook:
```sh
new=$(git symbolic-ref --short -q HEAD)
[ "$3" = 1 ] && [ "$new" != ws ] && { git symbolic-ref -m "snap back" HEAD refs/heads/ws; git branch -D "$new"; }
```

Observed:
```
$ git checkout -b x
Switched to a new branch 'x'
[post-checkout] args: 08c0f59… 08c0f59… 1  HEAD=refs/heads/x
Deleted branch x (was 08c0f59).
rc=0 → On branch ws; branches: main s1 s2 ws; WIP unchanged: yes; status: MM a.txt, ?? untracked.txt
$ git switch -c y        → identical (args <W> <W> 1, branch y deleted, On branch ws, WIP unchanged)
$ git checkout -b z      (hook variant: symbolic-ref + git branch -m z renamed-z) → rc=0, On branch ws, branch renamed-z exists
$ git checkout -b s1     → fatal: a branch named 's1' already exists (rc 128, hook did not run)
$ git switch -c s1       → fatal: a branch named 's1' already exists (rc 128, hook did not run)
```
HEAD reflog: `checkout: moving from ws to x`, then `snap back`. The same snap-back without `-m` leaves an empty message.

Caveat (extra probe): `git checkout -b x2 main` carries the WIP over to main's tree. Snapping HEAD back to `ws` afterwards leaves the index and worktree at main's tree, and status shows `M  c.txt` and `M  d.txt` (s1's and s2's work as staged reversals). The hook should snap back only when `$1 == $2`.

**Verdict: VERIFIED** for the case without a start point.

## 7. Which commands fire commit hooks

Every hook listed in the question logs `name args` (post-rewrite also logs stdin; reference-transaction logs state and refnames). Each command ran in a fresh repo with `GIT_EDITOR=true`.

| Command | pre-commit | prepare-commit-msg | commit-msg | post-commit | pre-merge-commit | post-merge | post-rewrite | reference-transaction |
|---|---|---|---|---|---|---|---|---|
| `git cherry-pick f` | – | yes (`… message`) | – | yes | – | – | – | yes (`refs/heads/main`, CHERRY_PICK_HEAD, AUTO_MERGE) |
| `git revert --no-edit HEAD` | – | yes (`… message`) | – | yes | – | – | – | yes |
| `git merge --no-ff b` | – | yes (`.git/MERGE_MSG merge`) | yes | – | yes | yes (`0`) | – | yes |
| `git rebase main` (2 picks) | – | yes, per pick | – | yes, per pick | – | – | yes: `rebase`, stdin 2 `old new` lines | yes (detached HEAD per pick, `refs/heads/topic` at the end) |
| `git -c pull.rebase=false pull` (merge needed) | – | yes (`MERGE_MSG merge`) | yes | – | yes | yes (`0`) | – | yes (`refs/remotes/origin/main`, `refs/heads/main`) |
| `git stash` | – | – | – | – | – | – | – | yes (`refs/stash`, ORIG_HEAD, `refs/heads/main` old == new) |
| `git stash pop` | – | – | – | – | – | – | – | yes (`refs/stash`) |

In git 2.54, reference-transaction also has a `preparing` state. There the update to the current branch is listed as `HEAD`. In `prepared` and `committed` it is listed as the resolved `refs/heads/<branch>`. Not tested: cherry-pick or revert with an editor (`-e`, or revert without `--no-edit`).

**Verdict: VERIFIED.** For the layer, plain `git commit` is the only command that runs pre-commit. cherry-pick, revert and rebase create commits on `ws` without it (post-commit still runs). merge and pull run neither pre-commit nor post-commit.

## 8. reference-transaction as a guard

```sh
#!/bin/sh   # .git/hooks/reference-transaction
[ "$1" = prepared ] || exit 0
[ -n "$LAYER_INTERNAL" ] && exit 0
while read old new ref; do
  [ "$ref" = refs/heads/ws ] && { echo "layer: direct update of refs/heads/ws refused ($old -> $new)" >&2; exit 1; }
done
```

On the fixture, with staged `a.txt` line 3 and unstaged line 9 (`MM a.txt`):

| Command | User sees | rc | Afterwards |
|---|---|---|---|
| `git reset --soft HEAD~1` | `layer: … refused (<W> -> <base>)` then `fatal: in 'prepared' phase, update aborted by the reference-transaction hook` | 128 | HEAD = ws = W, status `MM a.txt` (unchanged) |
| `git commit -m blocked` | same two lines | 128 | unchanged, staged part still staged |
| `git commit -a -m blocked-a` | same two lines | 128 | unchanged (the `index.lock` was rolled back) |
| `LAYER_INTERNAL=1 git update-ref refs/heads/ws …` | nothing | 0 | allowed |
| `git reset HEAD~1` (mixed) | `Unstaged changes after reset: …`, then the refusal | 128 | HEAD still W, but the **index was reset to base**: ` M a.txt`, `MM c.txt`, `MM d.txt` |
| `git reset --hard HEAD~1` | the refusal only | 128 | HEAD still W, but index and worktree = base: `M  c.txt`, `M  d.txt`; **the a.txt WIP is gone** (re-tested with the refined guard below: 0 WIP lines left) |
| `git stash` | `Saved working directory…`, then a refusal `(<W> -> <W>)` | 1 | the stash entry exists and the worktree was reset; it is reported as a failure |
| `git reset -q` (no-op) | refusal `(<W> -> <W>)` | 128 | — |

In the `prepared` state the hook receives `refs/heads/ws` (not `HEAD`) for a commit on `ws`, so matching on the refname works.

Refined guard: also require `old != new`. Then `git stash` succeeds (rc 0) and `git stash pop` restores the WIP. The `reset --hard` data loss remains.

**Verdict: VERIFIED** for `commit` and `reset --soft`: they are rejected cleanly and nothing is changed. Two hazards: same-oid updates must be let through, and `reset --mixed`/`--hard` (and anything else that writes the index or worktree before its ref update) is rejected only after the damage is done. The guard is a tripwire, not a transaction.

## 9. CLAUDE_ENV_FILE and PATH (docs only)

From https://code.claude.com/docs/en/hooks, section "Persist environment variables":

> SessionStart hooks have access to the `CLAUDE_ENV_FILE` environment variable, which provides a file path where you can persist environment variables for the shell commands Claude runs later in the session.

> To set individual environment variables, write `export` statements to `CLAUDE_ENV_FILE`. Use append (`>>`) to preserve variables set by other hooks:
> ```bash
> echo 'export PATH="$PATH:./node_modules/.bin"' >> "$CLAUDE_ENV_FILE"
> ```

> Each Bash command runs the file's contents as shell code before the command itself, so a line there can use anything Bash evaluates, such as the `$PATH` reference in `export PATH="$PATH:./node_modules/.bin"`.

> If any line is of another kind, such as `export PATH="$PATH:./node_modules/.bin"` with its unescaped `$PATH` […] PowerShell commands receive none of the variables […] Bash commands still receive all of them.

> `CLAUDE_ENV_FILE` is available for SessionStart, Setup, CwdChanged, and FileChanged hooks.

**Verdict: VERIFIED.** The docs say the file runs as Bash code before each Bash command and show a PATH edit. Only an append is shown, but `export PATH="/dir:$PATH"` is the same mechanism, so a `git` shim in `/dir` takes precedence in later Bash tool commands. The docs say nothing about PowerShell commands (they receive none of these variables when a line uses `$PATH`), about git run by the human's own terminal, or about git calls Claude Code makes internally.
