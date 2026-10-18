# What Scion is, and what it means for this effort

Type: research
Status: resolved
Blocked by:
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

Anthropic / Claude has announced something called **Scion**. What is it, and does it change this effort?

- **What it is:** product, feature, or open-source project? Who ships it, its release status (announced, preview, GA), date, license, and whether it is plain git underneath.
- **The problem it solves:** does it let several Claude Code sessions (and the human) work in one shared working directory? Or does it isolate them (worktrees, sandboxes, separate clones)?
- **Overlap with our failure modes:** how it handles Mixed commit, Lost WIP and Clobbered edit, ownership of changes, and how commits are attributed to a session.
- **History shape:** does it commit onto one shared branch or one branch per session/agent? This feeds the open ticket [Same branch or virtual branches](04-same-branch-or-virtual-branches.md).
- **Mechanism:** hooks, a git wrapper, a daemon, a VCS of its own? Anything we can reuse or must avoid duplicating.
- **Impact on the destination:** does Scion make this layer redundant, complementary, or a direct competitor? What one line should the spec give for "why not Scion?", next to "why not GitButler or jj?"

Use primary sources: Anthropic's announcement, official docs, the repo and its source.

## Answer

- **What it is:** Anthropic's Scion is an in-house, centralized version control system. It replaced git for Anthropic's own monorepo and was announced in an engineering blog post on 2026-10-13. It is not a product: there is no release, repo or license, and Anthropic calls it "by no means a suitable replacement for git for general use". It is not git underneath. It has its own client (`scn`), a per-node daemon, cache servers, and storage in Spanner.
- **Problem and overlap:** it solves scale (30,000 merges a day, 25 million checkouts a day). Commits, merges and rebases run on the server, and checkouts are sparse or virtual. The post never mentions several sessions in one working directory, Mixed commits, Lost WIP, Clobbered edits or ownership. Agents and CI jobs each get their own checkout.
- **History shape:** trunk-based. Changesets land on main, refs other than main and release branches are discouraged, and there are no per-agent branches. Conflicts between concurrent changes are fixed after merge. This is a weak data point for ticket 04, because Scion has its own per-change review tool (Antique), while a GitHub PR still needs a branch.
- **Impact:** a different layer, so neither redundant nor a competitor. Nothing in it can be reused or would be duplicated. Anthropic's advice is to keep git and improve git servers, which backs ADR 0001. Spec line: *"Why not Scion? Anthropic's Scion is an unreleased, in-house replacement for git's server side at monorepo scale. It does nothing for sessions sharing one folder."*
- **The other Scion:** Google Cloud's Scion (Apache-2.0, nightly prereleases only) is an orchestrator that runs Claude Code and other agents in containers. Its "Shared-plain" mode mounts one git working directory into every agent with "no per-agent isolation". Its design doc says "No formal coordination mechanism", and it only sets a per-agent commit author name. That is the exact gap this layer fills. It also corrects the initial research, which said every orchestrator isolates.

Details: [research/scion.md](../../../research/scion.md)
