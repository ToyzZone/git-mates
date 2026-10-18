# Initial research: multiple agents in one shared working directory

Date: 2026-10-14
Method: web survey of primary sources (official docs, GitHub repos, release notes, project blogs). A research agent gathered the facts. Every claim links to its source. Unverified points are marked **(unverified)**.

## The question

What already exists for letting several AI coding agent sessions (for example, several Claude Code sessions) work at the same time in **one shared working directory**, without stepping on each other? How close does each tool come?

The concrete pain: a session is working in a worktree and has uncommitted WIP. A second Claude Code session opens in the same folder to make a quick fix. The two sessions' changes then get mixed:

- Session 2's commit sweeps in session 1's WIP.
- A `stash`, `checkout` or `reset` clobbers the other session's work.
- Half-finished edits break the other session's build or tests.

## Summary

Nothing cleanly solves this scenario. **GitButler** comes closest, but only partly. Almost every other tool isolates each agent in its own worktree, workspace or container. None of them protects WIP that already exists in your current directory.

| Tool | Solves shared-directory scenario? | License |
|---|---|---|
| GitButler | Partially, and the best fit; its Claude hooks were removed 2026-05-19 (see correction below) | FSL-1.1-MIT (Fair Source, not OSI) |
| Jujutsu (jj) | No: separate workspaces only | Apache-2.0 |
| Pijul | No | GPL-2.0 |
| Worktree/container orchestrators | No: all isolate | various |
| Entire CLI | Partially: attribution, no separation | MIT |
| re_gent | Partially: audit trail, no isolation | Apache-2.0 |
| Turso AgentFS | No: no git merge-back | MIT |

## Findings

### 1. GitButler (partial, the best fit)

> **Correction (2026-10-15):** GitButler **removed** its Claude Code hooks integration on 2026-05-19 (commit [1dbe2bdb](https://github.com/gitbutlerapp/gitbutler/commit/1dbe2bdbe88e6c79da228a8fa080985340a27f3a), "Remove agent/codegen integration"). Even when it existed, it only claimed `Edit|MultiEdit|Write` hunks; changes made through Bash and by hand stayed unassigned. Today `but agentlog hook` records transcripts and dirty paths, but it has no per-session hunk ownership. The paragraph below describes the integration before it was removed. See [detecting-changes-outside-hooks.md](detecting-changes-outside-hooks.md#gitbutler).

Since about July 2025, GitButler has used Claude Code hooks to route each session's edits into a separate virtual branch inside one shared working directory, with one commit per chat round ([blog](https://blog.gitbutler.com/parallel-claude-code)). It assigns changes per hunk, which plain-git approaches can't do ([blog](https://blog.gitbutler.com/automate-your-ai-workflows-with-claude-code-hooks)). An Agents tab arrived in the GUI around September 2025 ([blog](https://blog.gitbutler.com/agents-tab)). The `but` CLI has been in technical preview since about February 2026, with JSON output and `but agent setup` for agent skills; the installer is Mac-only so far ([blog](https://blog.gitbutler.com/but-cli)).

- **License:** FSL-1.1-MIT, "Fair Source" that converts to MIT after two years. It is **not** OSI open source. 21.8k stars ([repo](https://github.com/gitbutlerapp/gitbutler)).
- **Gaps:**
  - The hooks tie one session to one branch.
  - Agents must use `but`, not raw git.
  - Its maintainers warn that sessions touching the same areas "can definitely lead to problems related to races and interference."
  - Builds and tests are not isolated ([trigger.dev, Apr 2026](https://trigger.dev/blog/parallel-agents-gitbutler)).
  - Open bug: the pre-tool hook creates empty branches for edits to ignored files ([#13316](https://github.com/gitbutlerapp/gitbutler/issues/13316)).
- **(unverified)** The official hooks docs page returned 404. Behavior when two sessions edit the same hunk is unconfirmed.

### 2. Jujutsu (jj) (no: separate workspaces only)

The working copy is a real commit, snapshotted automatically. Operations are lock-free and merge when they diverge ([repo](https://github.com/jj-vcs/jj), [concurrency docs](https://docs.jj-vcs.dev/latest/technical/concurrency/)). Each workspace has one working-copy commit, and parallel work means `jj workspace add` ([working-copy docs](https://docs.jj-vcs.dev/latest/working-copy/)).

- The docs never cover two writers in one working copy.
- **(unverified, inferred)** Both agents' edits would land in the same `@` commit with no record of who wrote what.
- Apache-2.0, 32k stars. Agent wrappers such as [wsg](https://github.com/jarvvski/wsg) also use one workspace per agent.

### 3. Pijul (no)

Patches commute and conflicts are first-class ([why Pijul](https://pijul.org/manual/why_pijul.html)). In theory that suits merging work from several writers. In practice:

- It is still 1.0.0-beta ([site](https://pijul.org/)) and GPL-2.0.
- Git support is a one-way import behind a feature flag ([docs.rs](https://docs.rs/crate/pijul/1.0.0-beta.21)).
- It has no working-copy concurrency model and no agent integration.

### 4. Agent orchestrators (no: all isolate)

| Tool | Approach | License / traction |
|---|---|---|
| Claude Code (`--worktree`, subagent `isolation: worktree`) | Worktrees; actively blocks edits into the main checkout ([docs](https://code.claude.com/docs/en/worktrees.md)) | — |
| [claude-squad](https://github.com/smtg-ai/claude-squad) | tmux + worktrees | AGPL-3.0, 8.6k stars |
| [Conductor](https://www.conductor.build/docs/concepts/workspaces-and-branches) | One worktree per workspace, Mac only | License not found **(unverified)** |
| [Crystal](https://github.com/stravu/crystal) | Worktrees; deprecated Feb 2026 for Nimbalyst | MIT, 3.1k stars |
| [container-use](https://github.com/dagger/container-use) (Dagger) | One container and branch per agent, experimental | Apache-2.0, 4.1k stars |
| Cursor | Worktrees, applied back with `/apply-worktree` ([docs](https://cursor.com/docs/configuration/worktrees)) | closed |
| GitHub Copilot app | One worktree per session ([docs](https://docs.github.com/en/copilot/concepts/agents/github-copilot-app)) | closed |

**Common gap:** each worktree has setup costs, and none of them protects WIP already sitting in your current directory.

### 5. "VCS for agents" projects

- **[Entire CLI](https://github.com/entireio/cli)** (partial: attribution without separation)
  - MIT, 5.2k stars.
  - Started by ex-GitHub CEO Thomas Dohmke with a $60M seed ([TechCrunch](https://techcrunch.com/2026/02/10/former-github-ceo-raises-record-60m-dev-tool-seed-round-at-300m-valuation/)).
  - Stores session checkpoints under `refs/entire/`.
  - Warns when a second session starts on uncommitted work, but a single commit still takes in every pending session's changes.
- **[re_gent](https://github.com/regent-vcs/regent)** (partial: audit trail without isolation)
  - Apache-2.0, about 800 stars.
  - Records each tool call; `rgt blame` shows which prompt wrote each line.
  - Tracks concurrent sessions as separate refs and is meant to run alongside git.
  - Rewind is only on the roadmap.
- **[Turso AgentFS](https://github.com/tursodatabase/agentfs)** (no)
  - MIT, 3.5k stars, beta.
  - Gives each agent a copy-on-write overlay of the directory, but no merge back into git.
- **[mcp_agent_mail](https://github.com/Dicklesworthstone/mcp_agent_mail)**
  - 2.2k stars, license not shown **(unverified)**.
  - Advisory file leases with an optional pre-commit guard.
  - The Python version is unmaintained and has been replaced by a Rust one.
- **Small projects**
  - [ruah-orch](https://github.com/ruah-dev/ruah-orch) (12 stars): worktrees plus file-lock claims.
  - [git-stint](https://github.com/rchaz/git-stint) (35 stars): one branch and worktree per session, with Claude hooks.
- **Hosted, closed source:** [Cloudflare Artifacts](https://blog.cloudflare.com/artifacts-git-for-agents-beta) and [Relace Repos](https://docs.relace.ai/docs/repos/overview). Both are git-compatible storage for agents; neither addresses a shared local directory.

## Evidence the problem is real and unsolved

- **[Claude Code #88862](https://github.com/anthropics/claude-code/issues/88862)**, opened 2026-08-22. Still open, with no maintainer response. It measures exactly this failure:
  - 3 to 6 sessions share one git index.
  - Commits take in another session's files.
  - There is no clean way to commit your own edit to a file that also holds someone else's WIP.
- **[claudectl #58](https://github.com/mercurialsolo/claudectl/issues/58)** only added a *warning* when sessions share a directory.

## Gaps nobody covers well (candidate differentiation)

1. **WIP adoption:** assign the uncommitted changes already in the directory to session 1 before session 2 starts. Every existing tool assumes a clean start.
2. **Per-session commits in plain git:** a session-scoped `GIT_INDEX_FILE` plus per-hunk attribution from PreToolUse/PostToolUse diffs. Open source and git-native, unlike GitButler, with no `but` lock-in.
3. **Guarding destructive git commands:** block or rewrite `stash`, `checkout`, `reset` and `clean` when another session owns dirty hunks. Nothing enforces this today. #88862 notes that the harness is "the only layer that sees the write."
4. **Same-file and same-hunk edits:** detect two sessions editing the same lines, and enforce leases at write time rather than only advising.
5. **Build and test interference:** run tests against one session's view (for example, a temporary overlay or worktree built from base plus that session's hunks) while the edits stay in the shared directory.
6. **One record across agents** (Claude Code, Codex, Cursor) under a permissive OSI license.

## What this means for the project (provisional)

- The clearest differentiator from GitButler is a **layer on top of plain git, under a real open-source license, with no new CLI that agents must learn**.
- Which gap to lead with depends on which failure actually hurts most: mixed commits, lost WIP, overwritten edits, or broken builds. That question is still open.

## Not verified

- GitButler's official hooks docs (404).
- GitButler's behavior when two sessions edit the same hunk.
- jj's behavior with two writers in one workspace (inferred, not documented).
- Conductor's license.
- mcp_agent_mail's license.
- Exact dates of the GitButler blog posts (only relative dates are shown).
