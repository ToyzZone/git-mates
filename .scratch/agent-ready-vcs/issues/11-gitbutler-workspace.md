# How GitButler's workspace and virtual branches work today

Type: research
Status: resolved
Blocked by:
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

GitButler already runs several branches in one working directory, and we have chosen the same model ([ADR 0002](../../../docs/adr/0002-a-branch-per-session.md)). How does it do it today?

- **The workspace:** what HEAD points at, how the workspace commit is built and rebuilt, what happens to the index, and what plain `git status`, `git log`, `git commit` and `git switch` do inside it. How GitButler recovers when someone runs plain git in its workspace anyway.
- **Dependent and adjacent hunks:** how it decides a hunk depends on, or is locked to, another branch. How it treats changes on neighbouring lines (git's merge conflicts on these). When it stacks branches.
- **Upstream:** what happens when the base branch moves, and when one branch is merged upstream.
- **Discarding a branch:** what unapplying or deleting a branch does to its commits, its WIP and its files on disk. Can it be undone (oplog)?
- **Agents today:** what replaced its Claude Code hooks after 2026-05-19, and what the `but` CLI asks of agents.
- **License:** what FSL-1.1-MIT allows if we read its design and write our own code.

Use primary sources: GitButler's source at a pinned commit and its official docs.

## Answer

- **Workspace (12):** HEAD sits on `gitbutler/workspace`, a merge of all applied branch tips, with the index forced to its tree. A `pre-commit` hook blocks `git commit`. A commit forced past it makes every `but` command stop until `but teardown` or `git reset --soft`. `git stash` and `git reset --hard` are not guarded. GitButler is retreating from this: since 2026-10-13 HEAD may sit on a normal branch, the merge commit appears only when a second branch is applied, and the hooks are called "legacy".
- **Adjacent lines (13):** the lock check is strict overlap with 0 context lines, but committing uses a 3-way merge that conflicts on touching lines, like git. So a hunk next to another branch's lines is often not shown as locked, yet committing it is rejected for the whole file. GitButler never makes a Mixed commit and never stacks automatically. Since March 2026 locks no longer force a hunk onto a branch, and two branches that conflict cannot both stay applied.
- **Upstream (12, 14):** `but pull` rebases every lane onto the new target and drops commits already upstream. Conflicts become "conflicted commits" stored in history with extra directories in the tree. WIP is 3-way merged and may get conflict markers. Integrated branches are deleted locally.
- **Discarding (14):** `but branch delete` drops a branch's commits and ref. Unapply keeps them and removes the files from disk. Uncommitted work follows a branch only if it was *assigned*, and the CLI stopped assigning in August 2026. Undo restores a whole-repo snapshot (all WIP, untracked included), so it also rewinds every other session's WIP. Branches already unapplied at snapshot time can't come back from it.
- **Agents today:** no hooks replaced the removed ones. `but agent setup` writes instructions into `CLAUDE.md` ("Use a dedicated GitButler branch for each agent session. Commit only changes that belong to that session") and a skill that forbids git writes. Agents commit with `but commit -b <branch> <hunk ids>`. The docs say "The branch routing is the agent's job". Nothing records or enforces ownership.
- **License:** FSL-1.1-MIT governs GitButler's code, not its ideas. Reading it to learn the design is a permitted "non-commercial research" use. Copying or translating its code would carry the FSL terms into our project, so only code we write ourselves can ship under MIT. Each commit becomes MIT two years after it is pushed. Patents are unverified.

Details: [research/gitbutler-workspace.md](../../../research/gitbutler-workspace.md)
