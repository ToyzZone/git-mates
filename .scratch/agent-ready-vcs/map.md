# Agent-ready version control: v0 spec

Label: wayfinder:map

## Destination

A v0 spec, ready to start coding, for an open-source Rust layer on plain git. The layer lets several Claude Code sessions and the human edit one shared working directory without **Mixed commits** or **Lost WIP**. The spec includes a one-line answer to "why not GitButler or jj?".

## Notes

- **Domain:** a concurrency layer on top of plain git. Use the terms in [GLOSSARY.md](../../GLOSSARY.md) (Session, Human session, Ownership, WIP, Base branch, Session branch, Workspace branch, Workspace commit, Mixed commit, Lost WIP, Clobbered edit, Cross-session breakage) and keep it updated.
- **Pitch** (README headline, agreed): *"Run many Claude Code sessions in one folder. Each one commits only its own changes, and none can wipe another's WIP."* Tagline: *"Per-session git for shared working directories."* Plain git (same repo, same GitHub), no new CLI that agents must learn, a real OSI open-source license.
- **Settled while charting:**
  - Name: **git-mates**. The binary is `git-mates`, so it runs as `git mates <cmd>`. Reserved on crates.io (placeholder 0.0.0, 2026-10-18); free on Homebrew.
  - License: dual `MIT OR Apache-2.0`, as is conventional for Rust ([LICENSE-MIT](../../LICENSE-MIT), [LICENSE-APACHE](../../LICENSE-APACHE)).
  - Build a layer on plain git, not a new VCS ([ADR 0001](../../docs/adr/0001-layer-on-plain-git.md)).
  - Written in Rust; most of the code will be written by Claude.
  - v0 supports Claude Code only, designed so other agents can be added later.
  - The human is a session: hand edits belong to the Human session.
  - v0 focuses on Mixed commit and Lost WIP.
  - A Clobbered edit is minimally handled: v0 blocks the edit and tells the agent which session owns those lines.
- **Prior research:** [research/initial-research-2026-10-14.md](../../research/initial-research-2026-10-14.md), a landscape of GitButler, jj, Pijul, orchestrators and "VCS for agents" projects, plus the open Claude Code issue #88862.
- **Research output** goes in `research/<slug>.md` at the repo root (no git repo yet, so no research branches).
- **Skills:** grilling tickets call `grilling` and `domain-modeling`; prototype tickets call `prototype`. ADRs go in `docs/adr/`.
- **Language:** the user writes in Vietnamese; talk with them in Vietnamese. Files (tickets, research, spec, glossary) are in English.
- Planning only: tickets produce decisions, not product code. Prototypes are throwaway.

## Decisions so far

<!-- one line per closed ticket: [title](issues/NN-slug.md): gist -->

- [What Claude Code hooks expose for tracking ownership](issues/01-claude-code-hooks-for-ownership.md): every hook gets `session_id`, and Edit gives exact hunks. PreToolUse can deny or rewrite a Bash command, but hooks fail open by default. Bash subprocesses get `CLAUDE_CODE_SESSION_ID`. A plugin's `bin/` can't shadow `git`. Subagents share the parent's `session_id`.
- [Committing part of the working directory with plain git, safely under concurrency](issues/02-partial-commits-in-plain-git.md): verified that a private index followed by `apply --cached`, `write-tree`, `commit-tree` and a compare-and-swap `update-ref` commits only one session's hunks. The shared index must then be patched, or the next plain `git commit` reverts the change. Never use `GIT_INDEX_FILE=tmp git commit`. A `Session-Id` trailer records the session. In Rust, `git2` covers every step and `gix` doesn't.
- [Detecting changes that bypass Claude Code hooks](issues/03-detecting-changes-outside-hooks.md): no unprivileged OS mechanism reports which process wrote a file. When two sessions' Bash calls overlap, or the human types during one, the change can't be split between them. Snapshot-diffing around each Bash call is exact for a lone call. Only a long-running watcher sees human edits as they happen. Claude Code's `bashEditDiff` is beta and best effort. GitButler removed its Claude hooks on 2026-05-19, and they never claimed Bash changes.
- [Same branch or virtual branches](issues/04-same-branch-or-virtual-branches.md): each session commits onto its own Session branch, because throwing away all of one session's work, commits included, must be cheap. Accepted costs: HEAD sits on a commit that combines the branches, branches conflict on adjacent lines, and the model is the same as GitButler's. [ADR 0002](../../docs/adr/0002-a-branch-per-session.md).
- [What Scion is, and what it means for this effort](issues/10-scion.md): Anthropic's Scion (announced 2026-10-13) is an in-house, centralized replacement for git at monorepo scale, unreleased and not git underneath. It works at a different layer from ours, so it neither competes nor makes this redundant, and it backs ADR 0001. Google Cloud's open-source Scion orchestrator has a shared-folder mode with no coordination: the exact gap this layer fills.
- [How GitButler's workspace and virtual branches work today](issues/11-gitbutler-workspace.md): HEAD sits on a merge of the applied branches. Since 2026-10-13 it sits on a normal branch when only one branch is applied. Adjacent hunks are rejected at commit time, never mixed into another branch or auto-stacked. Deleting a branch and discarding its WIP are separate steps, and undo rewinds the whole repo. Agents route their own changes with `but`, and nothing records ownership. Reading its code to learn the design is allowed; copying it would bring FSL terms with it.
- [Unit of ownership and how every change gets attributed](issues/05-unit-of-ownership.md): a session owns changed lines, keyed by `session_id` with subagents folded in. Hooks alone record them: Edit/Write patches, plus snapshots around each Bash call, with no daemon. Changes no agent can be pinned to, overlapping Bash calls included, go to the Human session. Owned lines keep their owner when Bash or a hand edit rewrites them. Two known limits: hand edits during an agent's lone Bash call, and background commands' later writes. [ADR 0003](../../docs/adr/0003-own-changes-by-the-line.md).
- [What the workspace looks like to plain git](issues/12-workspace-to-plain-git.md): HEAD moves onto a local-only workspace branch at the first session commit. Its one commit has the base and every session branch as parents. Its tree matches what is committed on disk, so session branches that conflict can both stay. The Human session gets its own branch, so the layer never moves base. The human's plain `git commit` gets moved onto that branch by a git hook. `git switch` can't be blocked, so it means leaving the workspace, and commits pause until the human switches back. Session branches are plain `refs/heads/<tool>/<id>`, and an agent's `git checkout -b` names its branch. [ADR 0004](../../docs/adr/0004-workspace-commit-records-the-disk.md).
- [How a session commits only its own WIP](issues/06-committing-only-own-wip.md): git hooks are the one commit path for agents and the human. `pre-commit` strips other sessions' lines, and `post-commit` moves git's commit onto the session branch and rebuilds the workspace commit. PreToolUse only blocks. A commit holds lines both staged and owned. Agent commits get a `Session-Id` trailer, and only an agent's first commit enters the workspace. Push is refused with the exact command to run, amend amends the session's last commit, and an agent's `reset <commit>`, `rebase`, `merge` and `pull` are blocked. If the workspace branch is moved outside the layer, it is put back when one agent did it; otherwise commits pause. [ADR 0005](../../docs/adr/0005-git-hooks-are-the-commit-path.md).

## Not yet specified

- **A view for the human** of who owns what right now (a `status`-like view). With no daemon, it takes its own snapshot when run.
- **Install and CLI surface:** shipped as a Claude Code plugin, `cargo install`, or brew? How hooks get registered in a repo: the Claude Code hooks, plus the git hooks the workspace needs (`pre-commit`, `prepare-commit-msg`, `post-commit`, `pre-push`, `post-checkout`). They must live alongside an existing `core.hooksPath`, which the user's own global config sets, and chain with the repo's own hooks. Order matters: lint-staged, for example, stashes unstaged changes during pre-commit, and in a shared working directory those are other sessions' WIP.
- **Performance budget** for hooks on large repos. A snapshot runs at every PreToolUse and PostToolUse of Edit, Write and Bash, and every commit runs pre-commit and post-commit under the layer lock.
- **Translating an agent's history commands onto its session branch** (`git reset HEAD~1` as "undo my last commit", `rebase`, `merge`, `pull`). v0 blocks them.

## Out of scope

- **Cross-session breakage** (one session's WIP breaking another's build or tests). Fixing it needs a per-session view of the files, which is beyond v0.
- **Agents other than Claude Code** (Codex, Cursor, Aider). This is a later effort; v0 only avoids ruling them out.
- **A new storage format or non-git history.** Ruled out by [ADR 0001](../../docs/adr/0001-layer-on-plain-git.md).
