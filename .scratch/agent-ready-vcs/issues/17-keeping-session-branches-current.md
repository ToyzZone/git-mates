# Keeping session branches current

Type: grilling
Status: open
Blocked by: 13
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

The base branch moves: the human pulls, or a session's PR merges upstream. What happens to the session branches and the workspace commit ([What the workspace looks like to plain git](12-workspace-to-plain-git.md))?

- **Base moves upstream:** when and how the workspace takes in the new base tip. The human's plain `git pull` on the workspace branch has no upstream to pull from. Are session branches rebased onto the new tip, or do they keep their fork point? What happens to WIP whose lines changed upstream?
- **A session branch merges upstream:** how that is detected, squash merges included. Its branch and its parent leave the workspace commit. What happens to the Human session branch, which lives on indefinitely?
- **Back to base:** once no session branch is ahead, does HEAD return to the base branch?
- **Changing the base branch:** a human `git switch` only leaves the workspace. Is there a way to move the whole workspace onto another base, or is that out of v0?

## Comments

- From [How a session commits only its own WIP](06-committing-only-own-wip.md), 2026-10-18: the human's `git merge <base>`, or a `git pull` with an upstream, on the workspace branch runs neither pre-commit nor post-commit (verified). The layer sees the workspace branch moved and pauses commits. If merging base into the workspace is how a new base comes in, this ticket must define it.
