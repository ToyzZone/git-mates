# The workspace commit records what's on disk, not a merge of session branches

HEAD sits on a local workspace branch whose single commit has the base tip and every session branch as parents. That commit's tree is not git's merge of those branches. Instead, each rebuild takes the previous tree and adds the changes just committed, so the tree always matches what is committed in the shared working directory. Two session branches that conflict with each other, for example with changes on adjacent lines, can then both stay in the folder. Their conflict only shows when one of them merges into the base branch.

## Considered Options

- **Re-merge the session branch tips at every rebuild, as GitButler does:** rejected. Git treats adjacent lines as a conflict, so the merge fails even though the working directory holds both changes fine. GitButler then drops one branch from the workspace, which removes that session's files from disk while the session is still running.

## Consequences

- To git, the workspace commit looks like a merge whose result was resolved by hand. That is intended.
- Rebuilds that remove work, such as throwing a session away or a session branch merging upstream, can't just drop a parent. They have to take that session's changes back out of the tree.
- Details: [the ticket](../../.scratch/agent-ready-vcs/issues/12-workspace-to-plain-git.md).
