# Scion: what it is and what it means for this effort

Date: 2026-10-15
Ticket: [10 What Scion is, and what it means for this effort](../.scratch/agent-ready-vcs/issues/10-scion.md)
Method: primary sources only. For Anthropic's Scion, I read the announcement post in full: [Redesigning source control at Anthropic](https://www.anthropic.com/engineering/redesigning-source-control), published 2026-10-13. I then searched every other first-party channel for "Scion" (see [Searched, nothing found](#searched-nothing-found)). For Google's Scion, I read the README, the user docs, a design doc and the `sciontool` source at commit [e134c2d5](https://github.com/GoogleCloudPlatform/scion/tree/e134c2d55cfed5658704cef5ea02f5d27686fb49) (2026-10-15). Links into the Anthropic post use text fragments, so they jump to the quoted sentence. Anything I could not confirm is marked **(unverified)**.

## Summary

- **There are two Scions. The relevant one is Anthropic's.** It is an in-house, centralized version control system. Anthropic built it to replace git for its own monorepo, and announced it in an [engineering blog post on 2026-10-13](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=Published%20Oct%2013%2C%202026). The other is **Google Cloud's Scion**, an open-source agent orchestrator. It is not from Anthropic, but it matters here (last bullet).
- **Anthropic's Scion is not a product.** There is no release, repo or license. Anthropic says it is ["by no means a suitable replacement for git for general use"](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=by%20no%20means%20a%20suitable%20replacement) and that it is ["difficult to offer Scion in a form that would be broadly useful"](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=difficult%20to%20offer%20Scion). It is not git underneath: it has its own client (`scn`), servers, protocol and storage in Spanner.
- **It solves scale, not one shared folder.** The drivers are 30,000 merges a day, a history near 100 GB and 25 million checkouts a day. Commits, merges and rebases run on the server. Checkouts are sparse or virtual. The post never mentions several sessions in one working directory, ownership, Mixed commits, Lost WIP or Clobbered edits. Agents, CI and research jobs each make their own checkout.
- **History shape: trunk-based.** Changesets land on main. Refs other than main and release branches are discouraged. There are no per-agent branches. Concurrent changes can conflict after they merge, and Claude fixes them.
- **Impact: a different layer, so neither redundant nor a competitor.** Scion replaces the server side of source control for one giant monorepo. This effort works on one local working directory on plain git. Anthropic's own advice is to keep git and improve git servers and forges. That fits [ADR 0001](../docs/adr/0001-layer-on-plain-git.md).
- **Google's Scion has the exact gap this layer fills.** Its "Shared-plain" mode mounts one git working directory into every agent with ["no per-agent isolation"](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/docs-site/src/content/docs/local/workspaces-and-sharing.md#L16-L24). Its design doc says ["No formal coordination mechanism"](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/.design/git-workspace-hybrid.md#L390-L399). The only per-agent git setup is an author name. This corrects the [initial research](initial-research-2026-10-14.md), which said every orchestrator isolates its agents.

## Which Scion is relevant

| | Anthropic's Scion | Google's Scion |
| --- | --- | --- |
| **What** | An in-house version control system that replaced git for Anthropic's monorepo | An open-source orchestrator that runs teams of agents (Claude Code, Codex, OpenCode, Antigravity) in containers |
| **Source** | [Engineering blog post](https://www.anthropic.com/engineering/redesigning-source-control), 2026-10-13 | [GoogleCloudPlatform/scion](https://github.com/GoogleCloudPlatform/scion/tree/e134c2d55cfed5658704cef5ea02f5d27686fb49) |
| **Status** | Internal only. Not released | Repo created 2026-03-10 (GitHub API). Only nightly prereleases, no stable release ([releases](https://github.com/GoogleCloudPlatform/scion/releases)). "Not an officially supported Google product" ([README](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/README.md#L167)) |
| **License** | None (not published) | Apache-2.0 ([LICENSE](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/LICENSE)) |
| **Plain git?** | No | Yes. It drives plain git in the agents' workspaces |
| **Why it matters here** | It is what the ticket asks about | Its shared-workspace mode is the scenario this effort targets, left unprotected |

The ticket says "Anthropic / Claude has announced", so Anthropic's Scion is the one the question means. It is also the only Scion that Anthropic has published anything about (see [Searched, nothing found](#searched-nothing-found)). The press covers the same post: [The Register](https://www.theregister.com/devops/2026/10/13/anthropic-reinvents-source-control-as-claude-floods-the-world-with-code/5302943), [Simon Willison](https://simonwillison.net/2026/Oct/13/redesigning-source-control-at-anthropic/). I used them only to find the post. Scion is also a discontinued Toyota car brand, which is unrelated. It was the only "scion" hit in a code search of the `anthropics` GitHub org.

## Anthropic's Scion

All quotes in this section are from [the post](https://www.anthropic.com/engineering/redesigning-source-control).

### What it is

- **An in-house VCS that replaced git.** "we built Scion, an in-house version control system, and fully migrated our monorepo from git in five months" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=we%20built%20Scion)).
- **It shipped with sibling systems.** In the same five months they also built "a code review surface (called Antique), code search, a rich source control IAM system, novel ways of managing and landing large amounts of reviewed code" ([same](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=we%20built%20Scion)).
- **Team and timeline.** "A team of seven people built both Scion and Antique" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=A%20team%20of%20seven)). Most of the code was written by Claude. Engineers "examine the intent of the change as opposed to reading every line of code".
- **Prior art they name.** Google's Piper and Meta's Sapling ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=Piper%20and%20Meta)).
- **Not offered to others.** "Scion is by no means a suitable replacement for git for general use. The concepts we've outlined here really only make sense for truly enormous monorepos" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=by%20no%20means%20a%20suitable%20replacement)). It "diverges significantly from git, meaning companies would need to migrate every piece of the development lifecycle, and it requires its own code review, code search, and CI" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=diverges%20significantly%20from%20git)).
- **No repo, release or license exists** in the `anthropics` GitHub org (115 repos listed through the GitHub API, none named Scion, Antique or scn).

### The problem it solves

- **Merge volume.** "In May 2026, Anthropic averaged around 2,500 code changes merged per day; as of September, we merged over 30,000 per day" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=over%2030%2C000%20per%20day)).
- **Repo size and checkout volume.** "The history of our main branch alone is almost 100 GB, and a compressed working copy is over 25 GB. We perform on the order of 25 million code checkouts per day for CI, before counting developer environments, agent sandboxes, and research jobs" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=almost%20100%20GB)).
- **The insight.** They used git "as a centralized system with one authoritative copy, but nearly all of our clients were agents and jobs that are online by definition" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=online%20by%20definition)).
- **Result.** The infrastructure "is now 200x smaller than our previous git mirroring infrastructure in terms of CPU and memory" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=200x%20smaller)).

So the problem is server load and checkout cost at monorepo scale. It is not several writers in one checkout.

### Mechanism

- **The server does the work.** "the server answers questions about file history and blame, creates commits, performs merges and rebases, and so on. The client maintains a bounded local object cache, lays out the working copy files on disk, monitors for filesystem changes" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=performs%20merges%20and%20rebases)).
- **Commits happen on the server.** "every commit, rebase, or merge happens on the server, so there's nothing to push" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=discourages%20refs)).
- **Changesets, not trees.** "the unit of work is a 'changeset,' an entry in a write-ahead log". It stores "a list of the files that changed relative to its first parent" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=the%20unit%20of%20work)).
- **Sparse and virtual checkouts.** The protocol "is centered around sparse checkouts ... and virtualized checkouts, which materialize files on demand" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=virtualized%20checkouts)).
- **Components.** A client, `scn`. A per-node daemon, `scion-daemon`, "that stores objects for the whole node". Regional `scion-cache` servers. Authoritative `scion-server`s backed by Spanner and cloud storage ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=scn%20client)).
- **Migration path.** A "git-to-Scion ingestion pipeline" fed the monorepo into Scion ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=ingestion%20pipeline)). Whether a git mirror is still kept is not stated **(unverified)**.
- **Access control.** It can "apply permissions to individual directories" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=permissions%20to%20individual%20directories)).

Nothing here works on a local git working directory. There is no hook layer, no git wrapper, and nothing we could reuse or would duplicate.

### History shape

- **Refs are discouraged.** "The development lifecycle in Scion discourages refs (pointers to commits, such as branches and tags), except for the main ref or release branches" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=discourages%20refs)). Moving a ref is "a compare-and-swap on a single Spanner row" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=compare%2Dand%2Dswap%20on%20a%20single%20Spanner%20row)).
- **Trunk-based.** "people only care about history or blame for commits on the main line in trunk-based development in a monorepo" (footnote 1, [post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=trunk%2Dbased%20development)).
- **Conflicts are fixed after merge.** "We haven't been able to effectively replicate the industry standard of a global repository lock on the merge queue ... This does mean that concurrent changes can conflict, but Claude can typically fix these conflicts faster than humans can even identify them" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=concurrent%20changes%20can%20conflict)).
- **Review is per change, in Antique.** The unit that gets reviewed and landed is a change, not a branch **(inferred: the post names Antique and "landing large amounts of reviewed code" but does not describe how a change is grouped for review)**.

### Overlap with our failure modes

- **Mixed commit, Lost WIP, Clobbered edit:** not mentioned. Ownership of uncommitted changes and attribution of commits to a session are not mentioned either.
- **Isolation by separate checkouts.** Checkouts are counted per CI job, developer environment, agent sandbox and research job ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=agent%20sandboxes)). Making checkouts cheap is a main goal. This suggests each agent works in its own checkout, not a shared one **(inference; the post does not say whether two agents ever share a checkout)**.
- **The only concurrency it discusses** is between changes landing on main, which is after commit. Our failure modes all happen before commit, inside one working directory.

### What Anthropic asks of git

The post ends with suggestions for git, not for a local layer: "Git platforms can provide comprehensive server-side RPCs like Scion, perhaps even in a standardized way across forges ... The negotiation protocol doesn't have to look exactly like it does today, either, and the various sparse and shallow features in git can be made to compose well" ([post](https://www.anthropic.com/engineering/redesigning-source-control#:~:text=without%20departing%20from%20the%20overall%20model)). The goal is that others "apply some of the most successful parts of Scion to git without departing from the overall model". All of it concerns git servers and fetch. None of it touches a working directory.

## Google's Scion (not Anthropic)

Read at [e134c2d5](https://github.com/GoogleCloudPlatform/scion/tree/e134c2d55cfed5658704cef5ea02f5d27686fb49).

- **What it is.** "an open-source orchestration platform for teams of AI agents". It runs "harnesses such as Claude Code, Antigravity, Codex, and OpenCode as independent processes" and lets you "Choose shared workspaces or separate Git worktrees and clones" ([README](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/README.md#L7-L11)). Written in Go.
- **Four workspace modes** ([workspaces-and-sharing.md](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/docs-site/src/content/docs/local/workspaces-and-sharing.md#L14-L74)):
  - **Shared-plain:** "One workspace directory is mounted into every agent, with no per-agent isolation".
  - **Worktree-per-agent:** a worktree on a dedicated branch per agent. "Agents cannot step on each other's uncommitted changes."
  - **Clone-per-agent:** a full clone per agent, on a `scion/<agent-name>` branch.
  - **Empty-per-agent:** a private empty directory, without git.
- **Shared-plain works with git.** For a git project, the Hub's `shared` workspace mode "selects the shared checkout" ([L87-L104](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/docs-site/src/content/docs/local/workspaces-and-sharing.md#L87-L104)). Agents detect it with `SCION_WORKSPACE_MODE=shared-plain` plus `SCION_WORKSPACE_GIT=true` ([L116-L135](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/docs-site/src/content/docs/local/workspaces-and-sharing.md#L116-L135)).
- **The only protection is advice to the agent.** In Shared-plain: "Do *not* assume a file remains unchanged between the time you read it and the time you write it", and "use `scion message` to coordinate" before disruptive changes ([L147-L150](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/docs-site/src/content/docs/local/workspaces-and-sharing.md#L147-L150)).
- **The design doc names the risks and leaves them open** ([git-workspace-hybrid.md](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/.design/git-workspace-hybrid.md#L390-L399)). Two agents "could: Edit the same file simultaneously; Run `git checkout` to different branches; Run conflicting git operations (e.g., concurrent commits)". The resolution: "No formal coordination mechanism. This is up to the grove owner and users to manage". Its non-goals include "Per-agent branch isolation within the shared workspace (agents must coordinate)" ([L55-L58](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/.design/git-workspace-hybrid.md#L55-L58)). Its listed cons include "Potential for `.git` lock contention" ([L111-L116](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/.design/git-workspace-hybrid.md#L111-L116)).
- **One shared branch.** In a shared workspace, "only one branch can be checked out at a time", so the default branch for a new agent is "the current workspace branch (e.g., "main") instead of an agent-named branch" ([L401-L407](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/.design/git-workspace-hybrid.md#L401-L407)).
- **Attribution is the author name only.** For a shared git workspace, `sciontool init` writes a credential helper and `user.name = "Scion Agent (<name>)"`, `user.email = agent@scion.dev` into each agent's own `$HOME/.gitconfig` ([init.go](https://github.com/GoogleCloudPlatform/scion/blob/e134c2d55cfed5658704cef5ea02f5d27686fb49/cmd/sciontool/commands/init.go#L3238-L3353)). Nothing else is set per agent. All agents share the working tree, `.git` and therefore the index.
- **So Mixed commit and Lost WIP are unprotected in Shared-plain.** With one shared index, one agent's `git add -A && git commit` takes in another agent's WIP. One agent's `git checkout`, `stash` or `reset --hard` can discard another's WIP **(inference from git semantics and the shared mount; not tested in Scion)**.

## Implications (facts and constraints, no decisions)

1. **Anthropic's Scion does not overlap with this effort.** It is server-side, monorepo-scale and internal. Our problem sits in one local working directory, before commit. Nothing in the post addresses it.
2. **It does not compete either.** There is nothing to install, and Anthropic says not to use it as a general git replacement. A user with Claude Code and a normal git repo is not affected.
3. **It backs ADR 0001.** Anthropic's message to everyone else is to keep git and improve git's server side ("without departing from the overall model"). Building on plain git is in line with that.
4. **Its history shape is one trunk with no per-agent branches.** Refs other than main and release branches are discouraged, and identity and review attach to each change. This is one data point for [ticket 04](../.scratch/agent-ready-vcs/issues/04-same-branch-or-virtual-branches.md). It is weak, because Scion has its own review tool (Antique), and review per change works there without branches. On GitHub a PR still needs a branch.
5. **Google's Scion offers a shared git working directory with no protection.** It uses one shared branch and puts the agent name in the commit author. It leaves Mixed commits, Lost WIP and Clobbered edits to agent instructions. That is the gap this layer targets. So "some orchestrators share a folder and need this" is now a fact, not a guess.
6. **This layer could run under Google's Scion.** Each agent runs Claude Code in its own container, and the hooks would run inside each container. Ownership state would have to live in the shared mount, for example under `.git/`, so that every container sees it. On Kubernetes that mount can be NFS, where file locking is weaker **(inference; not a v0 target, because v0 is Claude Code on one machine)**.
7. **The prior research needs one correction.** [Initial research](initial-research-2026-10-14.md) says agent orchestrators "all isolate". Google's Scion has a shared mode. It isolates by default only for git projects.

## Searched, nothing found

I searched these first-party channels for "Scion" on 2026-10-15. The only hit is the post above.

- Anthropic news and engineering: the post at [anthropic.com/engineering/redesigning-source-control](https://www.anthropic.com/engineering/redesigning-source-control). No other anthropic.com or claude.com page in site-restricted web searches.
- Claude Code docs: no hit in [code.claude.com/docs/llms.txt](https://code.claude.com/docs/llms.txt) or [llms-full.txt](https://code.claude.com/docs/llms-full.txt) (9.6 MB).
- Claude Platform docs: no hit in [platform.claude.com/llms.txt](https://platform.claude.com/llms.txt) or [docs.claude.com/llms.txt](https://docs.claude.com/llms.txt).
- Claude Code changelog: no hit in [CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md), read at version 2.1.300.
- Anthropic's GitHub org: no repo named Scion, Antique or scn among 115 repos. A code search of `org:anthropics` for "scion" found only a mention of the Toyota brand in `anthropics/claude-quickstarts`.

## Not verified

- Whether Anthropic still mirrors the monorepo to git, or whether any team at Anthropic still uses git for it.
- How Anthropic's developers and agents group work for review in Antique (per changeset, stacks, or something else).
- Whether two Anthropic agents ever share one Scion checkout. The post only implies separate checkouts.
- Whether Anthropic plans to release any part of Scion. The post says a public version "wouldn't actually be very useful", but does not rule out releasing parts.
- That Mixed commits and Lost WIP actually happen in Google Scion's Shared-plain mode. This is inferred from the shared mount and git semantics, not tested.
- The public launch date of Google's Scion. Secondary sources say 2026-04-07. The repo was created on 2026-03-10.
