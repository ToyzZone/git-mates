# Same branch or virtual branches

Type: prototype
Status: resolved
Blocked by:
Map: [Agent-ready version control: v0 spec](../map.md)
Prototype: [prototypes/same-branch-or-virtual-branches.html](../../../prototypes/same-branch-or-virtual-branches.html) (throwaway, opens by double-click)

## Question

Should sessions commit onto **one shared branch**, with every commit recording its session? Or should each session get its **own virtual branch**, as in GitButler?

Build a throwaway prototype on a toy repo that plays the user's scenario both ways:

1. Session 1 has WIP on a big feature.
2. Session 2 makes a quick fix and commits.
3. Session 1 commits later.

React to:

- What `git log` looks like in each case.
- How you'd open a PR with only one session's work.
- What happens when both sessions touch the same file.

Context:

- **Why the user leans toward virtual branches:** they want to tell which session made which commit.
- **What a shared branch gives:** that can be met on one branch by recording the session id in each commit. What one branch can't give is a separate PR per session.
- **What virtual branches cost:** direct competition with GitButler, a working directory that must be the merge of every branch, and conflicts between branches.

## Answer

**Each session commits onto its own session branch (B).** The user decided this after working through the prototype.

- **The deciding need:** throwing away everything one session did, including commits it made along the way. Typical case: open a session to research an idea. It commits as it goes, concludes the idea isn't feasible, and the whole thing goes in the bin. With session branches, that means deleting one branch. On one shared branch, its commits sit between other sessions' commits, and once pushed they need a revert on main.
- **Also gained:** a PR per session with no cherry-pick, and half-done checkpoints stay off the shared branch.
- **Costs accepted** (all visible in the prototype):
  - HEAD can't stay on the base branch, because plain `git status` would then show committed work as modified. It must sit on a commit that combines every session branch.
  - Changes on the same or adjacent lines in two session branches conflict, even when the working directory holds both fine.
  - A change built on lines another session committed can't stand on its own branch.
  - It is the same model as GitButler's virtual branches, so "why not GitButler?" must rest on other ground.
- **Rejected:** one shared branch with a `Session-Id` trailer, plus commands that rebuild one session's work on request (split it into a PR, drop it). That works while commits are unpushed. But it reconstructs a branch when asked, instead of giving you one you can see and delete.
- **Git facts checked against real git:**
  - Changes on the same or adjacent lines conflict, in both merge and cherry-pick. One untouched line between them is enough to merge cleanly.
  - On a shared branch, a stacked PR applies cleanly once the PR it depends on has merged.
  - Two branches with adjacent changes conflict on the second merge.

Recorded in [ADR 0002](../../../docs/adr/0002-a-branch-per-session.md). Prototype: [prototypes/same-branch-or-virtual-branches.html](../../../prototypes/same-branch-or-virtual-branches.html).
