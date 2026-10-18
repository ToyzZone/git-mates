# Throwing away all of one session's work

Type: grilling
Status: open
Blocked by: 12, 13
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

This is why we chose session branches ([ADR 0002](../../../docs/adr/0002-a-branch-per-session.md)). A session researches an idea, commits as it goes, and concludes the idea isn't feasible. The human throws everything away. How does that work?

- **Who and how:** the human only, or the session itself too? With what command, given the pitch of "no new CLI that agents must learn"?
- **What goes:** the session branch, its WIP, the new files it created, and its lines in files other sessions also touched.
- **Edge cases:** another session's branch is stacked on it, or it was already pushed.
- **Safety net:** can it be undone, like a reflog?

## Comments

- From [What the workspace looks like to plain git](12-workspace-to-plain-git.md), 2026-10-18: the workspace commit's tree builds up commit by commit and isn't re-merged from the branches. So throwing a session away means taking its changes back out of that tree, not just dropping one of its parents. The Human session has a session branch too.
