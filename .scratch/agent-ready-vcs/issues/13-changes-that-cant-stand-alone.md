# Changes that can't stand on their own session branch

Type: grilling
Status: open
Blocked by: 05, 11
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

Git treats changes on the same or adjacent lines as a conflict. So one session's branch can conflict with another's even when the working directory holds both changes fine (prototype walkthrough 2). And a change that edits lines another session already committed can't stand on its own branch (walkthrough 4). What does v0 do?

- **Options:** stack the later session's branch on the earlier one automatically; block the edit or the commit; lock the change to the other branch (GitButler's approach, which makes a Mixed commit); or let the branches conflict and resolve it at merge time.
- **What the agent and the human are told.**
- **The margin:** keep git's rule (one untouched line between changes is enough), or have ownership leave a wider gap.

Prototype: [prototypes/same-branch-or-virtual-branches.html](../../../prototypes/same-branch-or-virtual-branches.html).

## Comments

- From [Unit of ownership and how every change gets attributed](05-unit-of-ownership.md), 2026-10-18: lines a session writes in a file that another session created belong to the writer, but the file doesn't exist on the writer's session branch. Cover this case too.
- From [What the workspace looks like to plain git](12-workspace-to-plain-git.md), 2026-10-18: the workspace commit's tree is the previous tree plus the changes just committed, not a merge of the session branches. So two session branches that conflict can both stay in the working directory. Unlike in GitButler, "let the branches conflict" no longer means one of them has to leave.
- From [How a session commits only its own WIP](06-committing-only-own-wip.md), 2026-10-18: `pre-commit` runs before git makes the commit, so it is where a commit whose lines can't apply onto the session branch tip can be refused with a message. `cherry-pick` and `revert` skip pre-commit, so for them the check can only happen in `post-commit`, after git has already made the commit on the workspace branch. Decide what happens in both places.
