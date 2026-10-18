# Own changes by the line, attributed by hooks alone

A session owns individual changed lines, not files or hunks, so a quick fix and another session's WIP can sit in one file. Ownership is recorded by Claude Code hooks alone. Edit and Write report their exact lines, and each Bash call is attributed by diffing snapshots taken before and after it. No unprivileged mechanism can tell which process wrote a file, so whatever the evidence can't pin to one agent session goes to the Human session, whose WIP no agent commits.

## Considered Options

- **Whole files:** rejected. A quick fix couldn't share a file with another session's WIP.
- **Hunks, as GitButler assigns them:** rejected. A hunk has no stable identity: two changes about six lines apart merge into one, and GitButler then gives the merged hunk to the larger one's branch without saying so.
- **A file-watching daemon:** rejected for v0. It sees hand edits sooner but still can't say who wrote them, and it is one more process to install and keep alive. A snapshot at every PreToolUse already shows a hand edit before an agent writes over it.
- **When two sessions' Bash calls overlap, guess** (the shorter or later call wins) **or lock Bash repo-wide:** rejected. A wrong guess is a silent Mixed commit. A lock makes one session wait out another's test run, and background commands escape it anyway.

## Consequences

- v0 has two known limits. A hand edit saved while an agent's Bash call runs alone is credited to that agent. A background command's writes after it launches are credited by the normal rules, so they can land with another session whose Bash call is running at the time.
- An agent's own Bash output can end up with the Human session. The agent is told, and the WIP has to be handed back.
- Details: [the ticket](../../.scratch/agent-ready-vcs/issues/05-unit-of-ownership.md), and research on [hooks](../../research/claude-code-hooks-for-ownership.md) and [detecting changes](../../research/detecting-changes-outside-hooks.md).
