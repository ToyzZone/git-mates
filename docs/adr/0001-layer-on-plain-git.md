# Build a layer on plain git, not a new version control system

The project began as "my own version control, so multiple agents can work on the same branch". The real gap, though, is concurrency inside one working directory: no tool knows which session owns which uncommitted change. Storage, history and the network protocol are not the gap. So we keep git for history, storage and push/pull, and build only the layer that tracks Ownership per Session. That way the tool works on existing repos, GitHub and CI from day one, and agents keep the git fluency they already have.

## Considered Options

- **A new VCS (patch-based like Pijul, or a custom object store):** rejected. It would take years before it was usable on a real repo, and it would break every git-aware tool.
- **Adopt GitButler:** rejected as the product direction. It is Fair Source (FSL), not OSI open source, and agents must use its `but` CLI instead of raw git. See [research/initial-research-2026-10-14.md](../../research/initial-research-2026-10-14.md).
