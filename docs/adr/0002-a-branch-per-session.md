# Each session commits onto its own branch

When a session commits, the commit goes onto a **Session branch** of its own, not onto the branch the shared working directory is on. The deciding need is throwing away everything one session did, commits included. For example, a session researches an idea, commits as it goes, and concludes the idea isn't feasible. With a session branch, that means deleting one branch. On one shared branch, its commits sit between other sessions' commits, so removing them means reverting, or rewriting a branch the other sessions stand on. A session branch also gives one PR per session with no cherry-pick, and keeps half-done checkpoints off the shared branch.

## Considered Options

- **One shared branch, with a `Session-Id` trailer on each commit:** rejected. It keeps HEAD on a real branch and never conflicts on adjacent lines. Commands could rebuild one session's work on request, to split it into a PR or to drop it. But that only works while the commits are unpushed, and it reconstructs a branch when asked instead of giving you one you can see and delete.

## Consequences

- HEAD can't stay on the base branch, because plain `git status` would then show other sessions' committed work as modified. The working directory needs a commit that combines every session branch.
- Git treats changes on the same or adjacent lines as a conflict. So two session branches can conflict even when the working directory holds both changes fine. A change built on lines another session committed can't stand on its own branch.
- This is the same model as GitButler's virtual branches. "Why not GitButler?" must rest on other ground: an OSI license, plain git for agents, and ownership of Bash and hand edits.
- Prototype: [prototypes/same-branch-or-virtual-branches.html](../../prototypes/same-branch-or-virtual-branches.html).
