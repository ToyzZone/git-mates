# What Claude Code hooks expose for tracking ownership

Type: research
Status: resolved
Blocked by:
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

What can Claude Code hooks see and do that matters for tracking which session changed what?

- **Inputs:** for each relevant event (SessionStart, SessionEnd, PreToolUse, PostToolUse, UserPromptSubmit, Stop, SubagentStop, and any others), what JSON does the hook receive? In particular: `session_id`, `cwd`, `transcript_path`, `tool_name`, and `tool_input` for Edit, Write, MultiEdit, NotebookEdit and Bash.
- **Powers:** can a PreToolUse hook deny a tool call with a message the agent sees? Can it rewrite the tool input, for example change a Bash command before it runs? Can it inject context into the conversation?
- **Identity:** do subagents share the parent's `session_id`? What happens to `session_id` on resume, `/clear` and compaction?
- **Runtime:** timeouts, latency expectations, whether hooks from different sessions run concurrently, and environment variables available (e.g. `CLAUDE_PROJECT_DIR`).
- **Distribution:** how a plugin ships hooks so a user can install them once.

Use primary sources only: the official Claude Code hooks reference and docs, plus the source or changelog where needed.

## Answer

- **Inputs:** every hook gets `session_id`, `cwd` and `transcript_path`. Tool hooks also get `tool_name`, `tool_input` and `tool_use_id`, plus `agent_id`/`agent_type` inside subagents. Edit/Write `file_path` is always absolute. `NotebookEdit` uses `notebook_path`. `MultiEdit` is not a current tool.
- **PostToolUse:** on Edit it gets `structuredPatch` (line hunks) and `originalFile`. On Bash it can get `gitOperation` (commits) and a best-effort, beta `bashEditDiff` list of changed files.
- **Powers:** PreToolUse can deny with a reason Claude sees, even in bypassPermissions mode. It can rewrite the whole input, including a Bash command, through `updatedInput`, and it can inject `additionalContext`. It fails open on timeout or crash unless `onFailure: "block"` (v2.1.295+). A mod's `tool.check` can override a non-managed deny.
- **Identity:** subagents report the parent's `session_id` and differ only by `agent_id` (implied). `/clear` and forks mint a new ID. Resume keeps the old ID, and two terminals can share it. Compaction keeps the ID (implied).
- **Bash identity:** `CLAUDE_CODE_SESSION_ID` is set in every Bash subprocess. No `agent_id` variable is set there.
- **Runtime:** hooks are synchronous by default (600 s timeout; `SessionEnd` gets a 1.5 s budget). They run in parallel, and parallel tool calls fire PostToolUse concurrently. Cross-session concurrency is not documented but follows from separate processes. Env vars: `CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`/`CLAUDE_PLUGIN_DATA`, `CLAUDE_CODE_SESSION_ID`, `CLAUDE_PID` and others.
- **Distribution:** a plugin ships `hooks/hooks.json` through a git-hosted marketplace, installed at user, project or local scope. Its `bin/` comes after the user's PATH, so it cannot shadow `git`.

Full findings, sources and the unverified list: [research/claude-code-hooks-for-ownership.md](../../../research/claude-code-hooks-for-ownership.md)
