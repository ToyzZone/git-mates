# Detecting changes that bypass Claude Code hooks

Type: research
Status: resolved
Blocked by:
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

Many changes don't come through Claude Code's Edit or Write tools. Agents use Bash (`sed -i`, code generators, formatters, package installs that rewrite lockfiles), and the human edits in an editor. What are the reliable ways to catch every change so it can be given to a session, with hand edits going to the Human session?

Compare:

- **Snapshot diff:** diff the working directory before and after each Bash tool call.
- **Continuous watching:** FSEvents on macOS, inotify on Linux, through the Rust `notify` crate.
- **git's fsmonitor or Watchman.**

For each: cost on large repos, how it respects `.gitignore`, and the race when two sessions run Bash at once (whose command changed this file?). Include how GitButler, jj (Watchman support), re_gent and Entire CLI detect changes.

Use primary sources: official docs and source code.

## Answer

- **No unprivileged mechanism can say which session changed a file.** inotify reports no process, FSEvents marks only the watcher's own events, and git fsmonitor and Watchman answer "what changed since T", not "who". Only fanotify (`CAP_SYS_ADMIN`) and macOS Endpoint Security (an entitlement Apple must grant) report the writing process. So when two sessions' Bash windows overlap, or the human types during one, ownership cannot be split, whichever approach you pick.
- **Snapshot diff around each Bash call** is exact for a lone call, and `.gitignore` comes free through git. Repeated `git status` is cheap with `core.untrackedCache` plus `core.fsmonitor`, which is built in on Linux since git 2.55. It misses background commands, which outlive PostToolUse. Keeping hunks requires saving the pre-call content.
- **Continuous watching (`notify`)** sees human edits live. It needs a long-running process, does no `.gitignore` filtering, costs one inotify watch per directory on Linux, and can drop events, so it always needs a full-rescan path.
- **Claude Code already ships a Bash snapshot diff:** `bashEditDiff` (v2.1.269+), beta, with a `shared` flag for concurrent calls. It is best effort.
- **The four tools:** GitButler watches and re-runs a full status, and never claimed Bash changes for a session (its Claude hooks were removed 2026-05-19). jj walks the tree per command and can narrow the walk with Watchman. re_gent and Entire run a full snapshot or status at Stop and credit whatever changed to the session that stops. Entire's code admits "the first to stop would claim both".

Details: [research/detecting-changes-outside-hooks.md](../../../research/detecting-changes-outside-hooks.md)
