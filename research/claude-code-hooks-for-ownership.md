# What Claude Code hooks expose for tracking ownership

Date: 2026-10-14
Ticket: [01-claude-code-hooks-for-ownership](../.scratch/agent-ready-vcs/issues/01-claude-code-hooks-for-ownership.md)
Method: read the official Claude Code docs (code.claude.com, fetched 2026-10-14 as Markdown), the Agent SDK type reference, and the official changelog. Each claim links to its source. Points not stated in a primary source are marked **(unverified)**. Points that the docs only imply are marked **(implied)**.

Version context: the docs describe Claude Code up to **v2.1.299** (released 2026-10-14). The CLI installed on this machine is **v2.1.271**, so some features below (marked with a version) are not available locally yet.

## Summary

- **Every hook gets `session_id`, `cwd` and `transcript_path`.** Tool hooks also get `tool_name`, `tool_input` and `tool_use_id`. For `Write`, `Edit` and `Read`, `tool_input.file_path` is always absolute. Inside a subagent, hooks also get `agent_id` and `agent_type`.
- **`MultiEdit` is not a current tool.** No current doc and no changelog entry mentions it. `NotebookEdit` uses `notebook_path`, not `file_path`.
- **A PreToolUse hook can deny a call with a reason Claude sees.** It can also rewrite the whole tool input, including a Bash `command`, through `updatedInput`, and it can add context for Claude. A deny is honored even in `bypassPermissions` mode.
- **PostToolUse on `Edit` gets the structured patch with line numbers and the original file.** PostToolUse on `Bash` can get a best-effort list of files the command changed (`bashEditDiff`, beta) and any git commit it made (`gitOperation`).
- **`CLAUDE_CODE_SESSION_ID` is set in every Bash tool subprocess and hook process.** It matches the hook's `session_id`. So a `git` that an agent runs through Bash can tell which session ran it, with no hook involved.
- **Session ID lifecycle:**
  - `/clear` and forks (`/branch`, `--fork-session`) start a **new** session ID.
  - Resume keeps the resumed ID.
  - Compaction keeps the session, so it should keep the ID **(implied)**.
  - Subagents report the **parent's** `session_id` and are told apart by `agent_id` **(implied)**.
- **Hooks are synchronous by default.**
  - Timeouts: 600 s for command hooks, 30 s on `UserPromptSubmit`, and a 1.5 s shared budget on `SessionEnd`.
  - A timed-out or crashed PreToolUse hook **fails open** unless it sets `onFailure: "block"` (v2.1.295+).
  - Matching hooks run in parallel. Parallel tool calls fire PostToolUse concurrently.
- **A plugin ships hooks in `hooks/hooks.json`** and is installed from a git-hosted marketplace at user, project or local scope. A plugin's `bin/` comes after the user's `PATH`, so it **cannot shadow `git`**.

## 1. Inputs: what JSON each event receives

### Common fields (every event)

Source: [Hooks reference: Common input fields](https://code.claude.com/docs/en/hooks#common-input-fields)

| Field | Meaning |
|---|---|
| `session_id` | Current session identifier |
| `prompt_id` | UUID of the user prompt being processed. Absent until the first user input |
| `transcript_path` | Path to the session transcript (JSONL). It is written asynchronously and can lag the in-memory conversation |
| `cwd` | Working directory when the hook fires. It follows Claude into a worktree and after `cd` ([Reference scripts by path](https://code.claude.com/docs/en/hooks#reference-scripts-by-path)) |
| `scratchpad_dir` | Session scratchpad path (v2.1.257+) |
| `permission_mode` | `default`, `plan`, `acceptEdits`, `auto`, `dontAsk` or `bypassPermissions`. Not every event gets it |
| `effort` | `{ "level": ... }`, on events inside a tool-use context |
| `hook_event_name` | Name of the event |
| `agent_id`, `agent_type` | `agent_id` identifies the subagent or in-process teammate the hook fires in. `agent_type` is the agent name. Both are present inside a subagent. `agent_type` alone is present when the session runs with `--agent` |

Hooks from settings files, managed policy and plugins **also fire inside subagents**. A subagent's tool calls fire the same `PreToolUse`/`PostToolUse` hooks, and the input carries `agent_id`/`agent_type` ([Hook locations](https://code.claude.com/docs/en/hooks#hook-locations); [Sub-agents: define hooks](https://code.claude.com/docs/en/sub-agents#define-hooks-for-subagents)). The Agent SDK type confirms the base shape `{session_id, transcript_path, cwd, prompt_id?, permission_mode?, effort?, agent_id?, agent_type?}` ([SDK TypeScript: BaseHookInput](https://code.claude.com/docs/en/agent-sdk/typescript#basehookinput)).

### Per-event extras that matter for ownership

| Event | Extra input | Source |
|---|---|---|
| `SessionStart` | `source` (`startup`, `resume`, `clear`, `compact` or `fork`), optional `model`, `agent_type`, `session_title`. On resume or fork it also gets `seconds_since_last_response`, `context_tokens` and related fields (v2.1.251+) | [SessionStart input](https://code.claude.com/docs/en/hooks#sessionstart-input) |
| `SessionEnd` | `reason` (`clear`, `resume`, `logout`, `prompt_input_exit` or `other`) | [SessionEnd](https://code.claude.com/docs/en/hooks#sessionend) |
| `UserPromptSubmit` | `prompt`, optional `session_title`. It also fires for scheduled tasks, background-subagent reports and messages from other sessions | [UserPromptSubmit](https://code.claude.com/docs/en/hooks#userpromptsubmit) |
| `PreToolUse` | `tool_name`, `tool_input`, `tool_use_id`, plus `mcp_server` provenance for MCP tools (v2.1.274+) | [PreToolUse input](https://code.claude.com/docs/en/hooks#pretooluse-input) |
| `PostToolUse` | `tool_input`, `tool_response` (the tool's **structured** output object), `tool_use_id`, `duration_ms` | [PostToolUse input](https://code.claude.com/docs/en/hooks#posttooluse-input) |
| `PostToolUseFailure` | `error`, `is_interrupt`, `duration_ms` | [PostToolUseFailure input](https://code.claude.com/docs/en/hooks#posttoolusefailure-input) |
| `PostToolBatch` | `tool_calls[]`, each with `tool_name`, `tool_input`, `tool_use_id` and a serialized `tool_response`. Fires once per parallel batch | [PostToolBatch](https://code.claude.com/docs/en/hooks#posttoolbatch) |
| `SubagentStart` | `agent_id`, `agent_type` | [SubagentStart](https://code.claude.com/docs/en/hooks#subagentstart) |
| `SubagentStop` | `agent_id`, `agent_type`, `agent_transcript_path`, `last_assistant_message`, `stop_hook_active`, `background_tasks`, `session_crons` | [SubagentStop](https://code.claude.com/docs/en/hooks#subagentstop) |
| `Stop` | `stop_hook_active`, `last_assistant_message`, `background_tasks[]` (in-flight shell, subagent and other tasks), `session_crons[]` | [Stop input](https://code.claude.com/docs/en/hooks#stop-input) |
| `PreCompact` / `PostCompact` | `trigger` (`manual` or `auto`); `custom_instructions` / `compact_summary` | [PreCompact](https://code.claude.com/docs/en/hooks#precompact), [PostCompact](https://code.claude.com/docs/en/hooks#postcompact) |
| `CwdChanged` | `old_cwd`, `new_cwd` | [CwdChanged](https://code.claude.com/docs/en/hooks#cwdchanged) |
| `FileChanged` | `file_path`, `event` (`change`, `add` or `unlink`). Detected by a filesystem watcher, whatever wrote the file, but only for watched paths | [FileChanged](https://code.claude.com/docs/en/hooks#filechanged) |
| `WorktreeCreate` / `WorktreeRemove` | `name` / `worktree_path` | [WorktreeCreate](https://code.claude.com/docs/en/hooks#worktreecreate) |

The docs list more events (`Setup`, `PermissionRequest`, `PermissionDenied`, `Notification`, `TaskCreated`, `TaskCompleted`, `TeammateIdle`, `ConfigChange`, `InstructionsLoaded`, `DirectoryAdded`, model-switch, elicitation and others) in the [event table](https://code.claude.com/docs/en/hooks#hook-lifecycle). None of them adds file-change information.

### `tool_input` per file-changing tool

| Tool | `tool_input` fields | Source |
|---|---|---|
| `Edit` | `file_path` (absolute), `old_string`, `new_string`, `replace_all` | [Hooks: Edit](https://code.claude.com/docs/en/hooks#edit); [SDK FileEditInput](https://code.claude.com/docs/en/agent-sdk/typescript#edit) |
| `Write` | `file_path` (absolute), `content` | [Hooks: Write](https://code.claude.com/docs/en/hooks#write) |
| `NotebookEdit` | `notebook_path`, `cell_id?`, `new_source`, `cell_type?`, `edit_mode?` (`replace`, `insert` or `delete`) | [SDK NotebookEditInput](https://code.claude.com/docs/en/agent-sdk/typescript#notebookedit) |
| `Bash` | `command`, `description?`, `timeout?`, `run_in_background?` (the SDK type also has `dangerouslyDisableSandbox?`) | [Hooks: Bash](https://code.claude.com/docs/en/hooks#bash); [SDK BashInput](https://code.claude.com/docs/en/agent-sdk/typescript#bash) |
| `PowerShell` | Same fields as Bash. On Windows without Git Bash, Bash is not registered at all, so match `Bash\|PowerShell` | [Hooks: PowerShell](https://code.claude.com/docs/en/hooks#powershell) |
| `MultiEdit` | **Not a current tool.** It is absent from the [tools reference](https://code.claude.com/docs/en/tools-reference) list, the hooks reference and the SDK types, and it is never mentioned in the [changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md). When or how it was removed is **(unverified)** | — |

- The guarantee that paths are absolute covers `Write`, `Edit` and `Read`. Claude Code expands `~` and relative paths before hooks run. On Windows, paths arrive with backslashes ([PreToolUse input](https://code.claude.com/docs/en/hooks#pretooluse-input)).
- Whether `NotebookEdit.notebook_path` is always absolute is **(unverified)**.

Other tools can also change files in the working directory: `Monitor` (it runs commands), `Workflow` (it runs subagents) and MCP tools ([tools reference](https://code.claude.com/docs/en/tools-reference)).

### `tool_response` on PostToolUse

PostToolUse receives the tool's structured `Output` object. PostToolBatch receives the serialized text the model sees ([PostToolBatch note](https://code.claude.com/docs/en/hooks#posttoolbatch-input)). The shapes come from the [SDK output types](https://code.claude.com/docs/en/agent-sdk/typescript#tool-output-types):

- **Edit** (`FileEditOutput`): `filePath`, `oldString`, `newString`, `originalFile` (the full file before the edit, or null), `structuredPatch[]` (`oldStart`, `oldLines`, `newStart`, `newLines`, `lines[]`), `userModified`, `replaceAll`, and optional `gitDiff` (`filename`, `status`, `additions`, `deletions`, `patch`, `repository`).
- **Write** (`FileWriteOutput`): `type` (`create` or `update`), `filePath`, `content`, `structuredPatch[]`, `originalFile`, `gitDiff?`.
- **NotebookEdit** (`NotebookEditOutput`): `notebook_path`, `cell_id?`, `edit_mode`, `original_file`, `updated_file`, and other fields.
- **Bash** (`BashOutput`): `stdout` (stdout and stderr merged), `stderr` (tool notices only), `interrupted`, `backgroundTaskId?` and other fields, plus `gitOperation?`. `gitOperation` covers a commit (`sha`, `kind` of committed, amended or cherry-picked, and `branch`), a push, a merge or rebase, and PR actions.
- **Bash `bashEditDiff`** (v2.1.269+, public beta) is in `tool_response.bashEditDiff` ([Hooks: Bash](https://code.claude.com/docs/en/hooks#bash)). It has:
  - `changedFiles[]`: absolute paths, at most 200.
  - `files[]`: diffs of up to 5 files.
  - `moreFiles`, `unavailable`, `skipped`. `skipped` is set for commands that move the working tree, such as `git checkout` and `git stash`.
  - `shared`: set when another Bash call, such as a subagent's, ran in the same repo at the same time, so some listed changes may be that call's.
  - It is recorded only in auto or `bypassPermissions` mode unless `bashEditDiffEnabled` (or `CLAUDE_CODE_BASH_EDIT_DIFF=1`) turns it on everywhere. Background and read-only commands carry no diff. The docs call it "best effort … Use the list to find what to review, not to enforce a policy."

### Concrete JSON examples

**PreToolUse on Bash.** Copied verbatim from the [Hooks reference: Common input fields](https://code.claude.com/docs/en/hooks#common-input-fields):

```json
{
  "session_id": "abc123",
  "prompt_id": "550e8400-e29b-41d4-a716-446655440000",
  "transcript_path": "/home/user/.claude/projects/.../transcript.jsonl",
  "cwd": "/home/user/my-project",
  "scratchpad_dir": "/tmp/claude-1000/-home-user-my-project/abc123/scratchpad",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test",
    "description": "Run test suite",
    "timeout": 120000,
    "run_in_background": false
  },
  "tool_use_id": "toolu_01ABC123..."
}
```

**PreToolUse on Edit.** The docs give no full Edit payload. This example is built from the documented common fields and the [Edit field table](https://code.claude.com/docs/en/hooks#edit). It shows the shape only; it was not captured from a real run. `agent_id` and `agent_type` appear only when the call comes from a subagent.

```json
{
  "session_id": "abc123",
  "transcript_path": "/home/user/.claude/projects/.../abc123.jsonl",
  "cwd": "/home/user/my-project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "agent_id": "def456",
  "agent_type": "general-purpose",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/home/user/my-project/src/app.ts",
    "old_string": "original text",
    "new_string": "replacement text",
    "replace_all": false
  },
  "tool_use_id": "toolu_01..."
}
```

**PostToolUse.** The docs' only full example is for `Write`, copied verbatim from [PostToolUse input](https://code.claude.com/docs/en/hooks#posttooluse-input):

```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../.claude/projects/.../00893aaf-19fa-41d2-8238-13269b9b3ca0.jsonl",
  "cwd": "/Users/...",
  "permission_mode": "default",
  "hook_event_name": "PostToolUse",
  "tool_name": "Write",
  "tool_input": { "file_path": "/path/to/file.txt", "content": "file content" },
  "tool_response": { "filePath": "/path/to/file.txt", "type": "create" },
  "tool_use_id": "toolu_01ABC123...",
  "duration_ms": 12
}
```

For **PostToolUse on Edit**, `tool_response` follows `FileEditOutput`. This shape is built from the SDK type; the values are placeholders:

```json
"tool_response": {
  "filePath": "/home/user/my-project/src/app.ts",
  "oldString": "original text",
  "newString": "replacement text",
  "originalFile": "<full file content before the edit>",
  "structuredPatch": [
    { "oldStart": 12, "oldLines": 1, "newStart": 12, "newLines": 1, "lines": ["<diff lines>"] }
  ],
  "userModified": false,
  "replaceAll": false
}
```

For **PostToolUse on Bash**, `tool_response` follows `BashOutput`, plus `bashEditDiff` when recording is on. Again the shape is built from the docs and the values are placeholders:

```json
"tool_response": {
  "stdout": "<merged stdout+stderr>",
  "stderr": "",
  "interrupted": false,
  "gitOperation": { "commit": { "sha": "<sha>", "kind": "committed", "branch": "main" } },
  "bashEditDiff": {
    "changedFiles": ["/home/user/my-project/src/app.ts"],
    "files": [{ "filePath": "/home/user/my-project/src/app.ts", "hunks": ["..."] }],
    "moreFiles": 0,
    "shared": false
  }
}
```

**SubagentStop.** Copied verbatim from [SubagentStop input](https://code.claude.com/docs/en/hooks#subagentstop-input). Note that `session_id` is the parent's and the subagent transcript sits under the parent's ID:

```json
{
  "session_id": "abc123",
  "transcript_path": "~/.claude/projects/.../abc123.jsonl",
  "cwd": "/Users/...",
  "permission_mode": "default",
  "hook_event_name": "SubagentStop",
  "stop_hook_active": false,
  "agent_id": "def456",
  "agent_type": "Explore",
  "agent_transcript_path": "~/.claude/projects/.../abc123/subagents/agent-def456.jsonl",
  "last_assistant_message": "Analysis complete. Found 3 potential issues...",
  "background_tasks": [],
  "session_crons": []
}
```

## 2. Powers: deny, rewrite, inject

### Deny with a message the agent sees: yes

- There are two ways to deny ([PreToolUse decision control](https://code.claude.com/docs/en/hooks#pretooluse-decision-control)):
  - Print `hookSpecificOutput.permissionDecision: "deny"` with `permissionDecisionReason`; the reason is "shown to Claude".
  - Exit with code 2; stderr becomes the denial reason that Claude sees ([Exit code 2](https://code.claude.com/docs/en/hooks#exit-code-2)).
- The other decisions are `allow`, `ask` (prompts the user, labeled `[plugin:<name>]` or `[settings]`) and `defer` (only with `-p`; pauses the run for an external UI).
- With several hooks, precedence is **deny > defer > ask > allow**. Every matching hook still runs to completion, and a sibling's side effects still happen ([hooks guide: combine results](https://code.claude.com/docs/en/hooks-guide#combine-results-from-multiple-hooks)).
- A PreToolUse `deny` works in **every permission mode, including `bypassPermissions` / `--dangerously-skip-permissions`**. An `allow` cannot override deny rules from settings ([hooks guide: hooks and permission modes](https://code.claude.com/docs/en/hooks-guide#hooks-and-permission-modes)).
- Caveat: an installed **mod** that handles `tool.check` can approve a call that a non-managed PreToolUse hook blocked ([same section](https://code.claude.com/docs/en/hooks-guide#hooks-and-permission-modes); [mods: events](https://code.claude.com/docs/en/plugins/mods/events)).

### Rewrite the tool input: yes

- `hookSpecificOutput.updatedInput` **replaces the entire input object**, so include the unchanged fields too.
- Permission rules and Bash auto-background eligibility are checked against the rewritten input. Pair it with `allow` to auto-approve, or with `ask` to show the user the modified input ([PreToolUse decision control](https://code.claude.com/docs/en/hooks#pretooluse-decision-control)).
- So a hook can rewrite a Bash `command` before it runs. `PermissionRequest` hooks can rewrite the input too ([PermissionRequest decision control](https://code.claude.com/docs/en/hooks#permissionrequest-decision-control)).
- When several hooks return `updatedInput`, **the last to finish wins, and the order is non-deterministic** ([hooks guide: limitations](https://code.claude.com/docs/en/hooks-guide#limitations)).
- `UserPromptSubmit` cannot replace the prompt ([Decision control](https://code.claude.com/docs/en/hooks#decision-control)).

### Rewrite the tool output: yes, but only what Claude sees

PostToolUse `updatedToolOutput` replaces the result Claude sees. The value must match the tool's output schema, or Claude Code ignores it. It cannot undo the action ([PostToolUse decision control](https://code.claude.com/docs/en/hooks#posttooluse-decision-control)).

### Inject context into the conversation: yes

- `hookSpecificOutput.additionalContext` reaches Claude as a system reminder.
- Events that accept it: `SessionStart`, `SubagentStart` (into the subagent), `UserPromptSubmit`, `PreToolUse`/`PostToolUse`/`PostToolUseFailure`/`PostToolBatch` (next to the tool result), `Stop`/`SubagentStop` (keeps the turn going) and `PostModelSwitch` ([Add context for Claude](https://code.claude.com/docs/en/hooks#add-context-for-claude)).
- Plain stdout also becomes context on `SessionStart`, `UserPromptSubmit`, `UserPromptExpansion` and `PostModelSwitch` ([Exit code 0](https://code.claude.com/docs/en/hooks#exit-code-0)).
- Limits:
  - The cap is 10,000 characters per string. Longer output is spilled to a file and replaced with a 2,000-character preview ([JSON output](https://code.claude.com/docs/en/hooks#json-output)).
  - Write the text as factual statements. Imperative "system" phrasing can trigger Claude's prompt-injection defenses.
  - On resume, saved mid-session context is replayed, not recomputed ([Add context for Claude](https://code.claude.com/docs/en/hooks#add-context-for-claude)).
- On PostToolUse/PostToolUseFailure, exiting 2 shows stderr to Claude even though the tool already ran ([Exit code 2 per event](https://code.claude.com/docs/en/hooks#exit-code-2-behavior-per-event)).

### Other relevant powers

- **`Stop` / `SubagentStop`** can block the turn from ending (`decision: "block"` plus `reason`). The cap is 8 consecutive continuations ([Stop decision control](https://code.claude.com/docs/en/hooks#stop-decision-control)).
- **`PostToolBatch`** can stop the agentic loop before the next model call ([PostToolBatch decision control](https://code.claude.com/docs/en/hooks#posttoolbatch-decision-control)).
- **`SessionStart`** can persist environment variables for every later Bash command by appending `export` lines to `$CLAUDE_ENV_FILE`. `CLAUDE_ENV_FILE` is available only to SessionStart, Setup, CwdChanged and FileChanged hooks ([Persist environment variables](https://code.claude.com/docs/en/hooks#persist-environment-variables)).
- **`FileChanged`** sees changes made by anything, including the human. Its limits ([FileChanged](https://code.claude.com/docs/en/hooks#filechanged)):
  - It watches only literal filenames from the matcher plus absolute paths returned as `watchPaths`.
  - It has no decision control.
  - Whether it can watch a whole directory tree is **(unverified)**.
- **Not covered by hooks:**
  - Files referenced with `@` are inserted without a tool call, so no PreToolUse fires ([PreToolUse warning](https://code.claude.com/docs/en/hooks#pretooluse)).
  - Whether `!` shell-mode commands typed by the user fire PreToolUse is **(unverified)**. The docs say only that they run "without going through Claude" ([interactive mode](https://code.claude.com/docs/en/interactive-mode#shell-mode-with-prefix)).
  - Whether `/rewind` "Restore code" fires any hook is **(unverified)**; no doc mentions one. Checkpointing can restore files that another concurrent session also edited ([checkpointing limitations](https://code.claude.com/docs/en/checkpointing#limitations)).

## 3. Identity: what `session_id` means

| Situation | `session_id` behavior | Source |
|---|---|---|
| New session | New ID. `SessionStart` with `source: "startup"` | [SessionStart](https://code.claude.com/docs/en/hooks#sessionstart) |
| Subagent (Agent tool) | Hooks fire with `agent_id` and `agent_type` added. `session_id` is the **parent's** **(implied)**: the SubagentStop example shows the parent's `session_id`, and subagent transcripts live under `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl`. No doc says so in one sentence | [SubagentStop input](https://code.claude.com/docs/en/hooks#subagentstop-input); [Sub-agents: resume](https://code.claude.com/docs/en/sub-agents#resume-subagents) |
| Resumed subagent | Runs again "under the same ID" (`agent_id`) | [Sub-agents: resume](https://code.claude.com/docs/en/sub-agents#resume-subagents) |
| `/clear` | **New session ID.** `SessionEnd` (`reason: "clear"`), then `SessionStart` (`source: "clear"`). `CLAUDE_CODE_SESSION_ID` "is updated on `/clear`" | [env vars](https://code.claude.com/docs/en/env-vars); [Sessions: manage context](https://code.claude.com/docs/en/sessions#manage-context-within-a-session) |
| Resume (`--resume <id>`, `--continue`, `/resume`) | Keeps the resumed session's ID. "On `--resume <session-id>` it receives the resumed ID, matching hooks and Bash." `SessionStart` with `source: "resume"`. Interactive `/resume` first fires `SessionEnd` (`reason: "resume"`) for the session you leave | [env vars](https://code.claude.com/docs/en/env-vars); [SessionEnd](https://code.claude.com/docs/en/hooks#sessionend) |
| Fork (`/branch`, `--fork-session`, `/fork`, moving to background) | **New session ID**. `SessionStart` with `source: "fork"` (v2.1.214+; before that it reported `resume`). `/branch` switches the *running process* to the new ID. In-flight background subagents and Bash keep running, and their output goes to the new branch | [Sessions: branch](https://code.claude.com/docs/en/sessions#branch-a-session); [SessionStart](https://code.claude.com/docs/en/hooks#sessionstart) |
| Compaction (manual, auto, idle) | `PreCompact`/`PostCompact`, then `SessionStart` with `source: "compact"`. No `SessionEnd` reason exists for compaction, and the docs say compaction "doesn't end your session". So the **ID stays the same (implied)** | [SessionStart](https://code.claude.com/docs/en/hooks#sessionstart); [context window](https://code.claude.com/docs/en/context-window) |
| Same session resumed in two terminals | Both processes share one ID: "messages from both interleave into one transcript" | [Sessions: branch](https://code.claude.com/docs/en/sessions#branch-a-session) |
| In-process agent-team teammate | Has its own `agent_id` (v2.1.290 changed it to the agent ID) | [Common input fields](https://code.claude.com/docs/en/hooks#common-input-fields); [changelog 2.1.290](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |
| Split-pane teammate | A separate Claude Code process. Whether it gets its own `session_id` is **(unverified)** | [Agent teams](https://code.claude.com/docs/en/agent-teams) |

Environment variables that carry the identity ([env vars reference](https://code.claude.com/docs/en/env-vars)):

- **`CLAUDE_CODE_SESSION_ID`** is set in Bash and PowerShell tool subprocesses, hook subprocesses and stdio MCP servers. For Bash and hooks it "matches the `session_id` field in the hook JSON input". It was added to Bash in v2.1.132 ([changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)).
- **No environment variable exposing `agent_id` to Bash subprocesses is documented.** A `git` that a subagent runs sees only the parent's session ID.
- **`CLAUDE_CODE_CHILD_SESSION=1`** is set only in subprocesses that Claude Code itself spawns (Bash, PowerShell, Monitor, hooks, status line).
- **`CLAUDECODE=1`** is also set by IDE extensions in their integrated terminals, so it **cannot** tell a human's IDE terminal apart from an agent.
- **`CLAUDE_PID`** is Claude Code's own PID in tool and hook subprocesses (v2.1.214+).

To list the sessions that are live on the machine (`sessionId`, `pid`, `cwd`, `status`, `state`), run `claude agents --json`. It is "the supported way to read session state from outside Claude Code" ([agent view: list sessions as JSON](https://code.claude.com/docs/en/agent-view#list-sessions-as-json)).

## 4. Runtime

- **Blocking.** Hooks block Claude until they finish, unless `async: true` is set ([Run hooks in the background](https://code.claude.com/docs/en/hooks#run-hooks-in-the-background)). Async hooks cannot decide anything, and their output arrives on the next turn. With `-p`, async hooks still running at teardown are killed.
- **Timeouts** ([Common fields](https://code.claude.com/docs/en/hooks#common-fields)):
  - `command`, `http` and `mcp_tool` default to 600 s.
  - `UserPromptSubmit`, `PreModelSwitch` and `PostModelSwitch` default to 30 s; `MessageDisplay` to 10 s.
  - `prompt` hooks default to 30 s and `agent` hooks to 60 s.
  - `SessionEnd` hooks share a **1.5 s budget**. A per-hook `timeout` in settings raises it, up to 60 s, but **timeouts on plugin hooks do not raise it**. `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` overrides it ([SessionEnd](https://code.claude.com/docs/en/hooks#sessionend)).
- **Fail-open by default.**
  - A PreToolUse command or HTTP hook that times out, crashes, cannot start or prints invalid JSON is a *non-blocking error*, and the tool call proceeds.
  - Exit 1 does not block; only exit 2 or a JSON deny does.
  - `onFailure: "block"` (v2.1.295+) turns any of these failures into a block ([Timeouts](https://code.claude.com/docs/en/hooks#timeouts); [Block the action when a hook fails](https://code.claude.com/docs/en/hooks#block-the-action-when-a-hook-fails)).
  - A hook that forks a background process must print its decision first and redirect the child's output. Otherwise its decision may be ignored ([Hooks that start a background process](https://code.claude.com/docs/en/hooks#hooks-that-start-a-background-process)).
- **Latency guidance.** The docs give no numeric latency target. They say "SessionStart runs on every session, so keep these hooks fast" ([SessionStart](https://code.claude.com/docs/en/hooks#sessionstart)) and that a stuck `UserPromptSubmit` hook "stalls the session" ([UserPromptSubmit](https://code.claude.com/docs/en/hooks#userpromptsubmit)). The handler-level `if` filter exists to avoid "process spawn overhead" ([How a hook resolves](https://code.claude.com/docs/en/hooks#how-a-hook-resolves)). Durations per hook appear in the debug log as `Hooks: … (31ms)` lines (v2.1.296+) ([Debug hooks](https://code.claude.com/docs/en/hooks#debug-hooks)).
- **Concurrency inside one session:**
  - "All matching hooks run in parallel" ([Hook handler fields](https://code.claude.com/docs/en/hooks#hook-handler-fields)).
  - "`PostToolUse` fires once per tool, which means it fires concurrently when Claude makes parallel tool calls" ([PostToolBatch](https://code.claude.com/docs/en/hooks#posttoolbatch)).
  - Background subagents run concurrently with the main conversation ([Sub-agents](https://code.claude.com/docs/en/sub-agents)), and their tool calls fire the same hooks.
  - `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default 10) limits "read-only tools and subagents" running in parallel ([env vars](https://code.claude.com/docs/en/env-vars)). Whether write tools from one agent ever run in parallel is **(unverified)**.
- **Concurrency across sessions.** The docs do not mention it. Each session is its own Claude Code process with its own hook processes, so nothing serializes hooks from different sessions. This is a reasonable inference, not documented **(unverified)**. The SDK hooks guide warns that "completion order is non-deterministic" ([SDK hooks](https://code.claude.com/docs/en/agent-sdk/hooks)).
- **Working directory and environment.**
  - Command hooks run in the current directory with Claude Code's environment. If that directory was deleted, they fall back to the start directory, then the project root, then home, then temp ([Hook handler fields](https://code.claude.com/docs/en/hooks#hook-handler-fields)).
  - A hook inherits the parent environment minus `OTEL_*` variables. With `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`, credentials are removed as well ([Common input fields](https://code.claude.com/docs/en/hooks#common-input-fields)).
  - Command hooks have no controlling terminal (no `/dev/tty`).
- **Environment variables available to hooks:**

  | Variable | What it holds |
  |---|---|
  | `CLAUDE_PROJECT_DIR` | The project root where the session started. It stays fixed even when `cwd` moves into a worktree |
  | `CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA`, `CLAUDE_PLUGIN_OPTION_<KEY>` | Plugin install directory, plugin persistent data directory, and plugin user-config values |
  | `CLAUDE_CODE_SESSION_ID` | The session ID (see section 3) |
  | `CLAUDECODE`, `CLAUDE_CODE_CHILD_SESSION` | Markers that the process was spawned by Claude Code (see section 3 for how they differ) |
  | `CLAUDE_PID` | Claude Code's own PID |
  | `CLAUDE_EFFORT` | Current effort level |
  | `CLAUDE_ENV_FILE` | Path for persisting variables (four events only) |
  | `CLAUDE_CODE_REMOTE`, `CLAUDE_CODE_BRIDGE_SESSION_ID` | Cloud session and Remote Control markers |
  | `CLAUDE_CODE_MESSAGING_SOCKET`, `CLAUDE_CODE_MESSAGING_TOKEN` | Cross-session messaging socket and token |

  Sources: [Exec form and shell form](https://code.claude.com/docs/en/hooks#exec-form-and-shell-form), [Reference scripts by path](https://code.claude.com/docs/en/hooks#reference-scripts-by-path), [env vars](https://code.claude.com/docs/en/env-vars). There is no `$CLAUDE_MODEL`.
- **Trust.**
  - In an interactive session, settings-file hooks (user settings included) wait until the folder's workspace-trust dialog is accepted.
  - `-p` and SDK sessions treat the folder as trusted ([Workspace trust](https://code.claude.com/docs/en/hooks#workspace-trust)).
  - Whether *plugin* hooks are also held back until trust is **(unverified)**.
- **Kill switches.**
  - `disableAllHooks` turns hooks off, but not managed hooks unless set at the managed level.
  - Managed `allowManagedHooksOnly` blocks user, project and plugin hooks, except hooks from plugins force-enabled by managed settings ([Hook locations](https://code.claude.com/docs/en/hooks#hook-locations); [Disable or remove hooks](https://code.claude.com/docs/en/hooks#disable-or-remove-hooks)).
  - `--bare` / `CLAUDE_CODE_SIMPLE` turns off hook auto-discovery entirely ([env vars](https://code.claude.com/docs/en/env-vars)).

## 5. Distribution: shipping hooks in a plugin

- **Where the hooks go.** Put them in `hooks/hooks.json` at the plugin root, under a top-level `"hooks"` key with the same shape as `settings.json`. A `hooks` key in the manifest also works ([plugin components: Hooks](https://code.claude.com/docs/en/plugins/components#hooks)). Plugin hooks merge with user and project hooks. They register as soon as a session loads the plugin, without waiting for a skill to be used.
- **Plugin paths.**
  - Refer to bundled scripts through `${CLAUDE_PLUGIN_ROOT}`. The path changes on every plugin update, so keep no state there.
  - Use `${CLAUDE_PLUGIN_DATA}` (`~/.claude/plugins/data/<id>/`) for state and dependencies that must survive updates.
  - Both are exported to hook processes ([path variables and persistent data](https://code.claude.com/docs/en/plugins/components#path-variables-and-persistent-data)).
  - Prefer exec form (`command` plus `args`) for placeholder paths ([Exec form and shell form](https://code.claude.com/docs/en/hooks#exec-form-and-shell-form)).
- **Install.** The user adds a marketplace once (`claude plugin marketplace add owner/repo`) and installs (`claude plugin install name@marketplace`). Since v2.1.292 one command does both: `claude plugin install name --marketplace owner/repo`. A marketplace is just a `.claude-plugin/marketplace.json` in a git repo ([publish](https://code.claude.com/docs/en/plugins/publish)). Auto-update is a per-marketplace setting on the user's side.
- **Scopes** ([install: choose a scope](https://code.claude.com/docs/en/plugins/install#choose-an-install-scope)):
  - *User* scope writes `~/.claude/settings.json` and covers all projects on the machine.
  - *Project* scope writes the committed `.claude/settings.json`. Committing it turns the plugin on for collaborators, but each collaborator still runs `claude plugin install … --scope project` once.
  - *Local* scope writes `.claude/settings.local.json`.
  - Local overrides project, and project overrides user.
- **Executables.** Files in a plugin's `bin/` are on the Bash tool's `PATH` while the plugin is enabled. They are **appended after the user's `PATH`**, "so a plugin can't shadow `git`" ([Executables](https://code.claude.com/docs/en/plugins/components#executables)). claude.ai and Cowork refuse plugins with a top-level `bin/`.
- **Restrictions.**
  - Plugin *subagents* cannot declare `hooks`. Ship the hooks in `hooks/hooks.json` instead ([Sub-agents](https://code.claude.com/docs/en/sub-agents)).
  - When two enabled plugins share a name, only one plugin's hooks register ([plugin components](https://code.claude.com/docs/en/plugins/components#hooks)).
- **Alternative: mods.** A plugin can register JS/TS handlers that run inside Claude Code's process. They can observe, rewrite or answer tool calls and draw UI, and they run in the CLI, Desktop, VS Code, `-p` and the SDK ([mods overview](https://code.claude.com/docs/en/plugins/mods/overview)). Organizations can restrict them.

## Implications for ownership tracking

These are facts and constraints only; later tickets make the decisions.

1. **Attributing tool edits is direct.**
   - `PreToolUse`/`PostToolUse` on `Edit`, `Write` and `NotebookEdit` give `session_id`, the absolute path (for Edit and Write), and on PostToolUse the exact line hunks (`structuredPatch`) plus the pre-edit content.
   - Constraint: `NotebookEdit` uses `notebook_path`, and `MultiEdit` does not exist.
   - Constraint: matching on `Edit.*` also matches `NotebookEdit` ([Matcher patterns](https://code.claude.com/docs/en/hooks#matcher-patterns)).
2. **Bash is the hard case.**
   - A hook sees only the command string. The `if` filter is best-effort, and the docs advise the permission system, not hooks, for hard allow or deny.
   - `bashEditDiff` is beta, off by default outside auto and bypass modes, and capped at 200 paths. It flags `shared` when concurrent Bash calls run in the same repo and skips `checkout`/`stash`.
   - Any design that attributes Bash-made changes must tolerate gaps (for example by diffing the working tree itself). It must not treat `bashEditDiff` as authoritative.
3. **Changes outside tool calls are invisible to tool hooks.** These include human edits in an editor, `!` shell mode **(unverified)** and `/rewind` restores **(unverified)**. Only `FileChanged`, which needs explicit paths and cannot block, or something outside Claude Code, can see them. This matches the glossary rule that the Human session owns whatever no agent session owns.
4. **The session key is not a stable "writer" key over time.**
   - `/clear` and forks mint new IDs mid-process.
   - Resume reuses an old ID, and two terminals can share one ID.
   - Subagents share the parent's `session_id` and differ only by `agent_id`, which Bash subprocesses do not receive.
   - Ownership must therefore decide whether the owner is `session_id`, `(session_id, agent_id)` or something else, and what happens to WIP across `/clear`, fork and resume.
5. **`CLAUDE_CODE_SESSION_ID` in Bash subprocesses** lets an agent-run `git` (or wrapper) know which session invoked it without parsing hooks. But a plugin cannot shadow `git` on `PATH`. Intercepting git therefore has to go through PreToolUse (deny with a message, or rewrite the command with `updatedInput`), through a user-level mechanism such as `CLAUDE_CODE_SHELL_PREFIX` ([env vars](https://code.claude.com/docs/en/env-vars)), or through git's own hooks. `CLAUDECODE=1` alone cannot separate agents from the human, because IDE terminals set it too.
6. **Blocking a Clobbered edit is possible.** PreToolUse can deny an Edit/Write with a reason Claude reads, including in `bypassPermissions` mode. Constraints:
   - It **fails open** on timeout or crash unless `onFailure: "block"` (v2.1.295+, newer than the locally installed 2.1.271).
   - A user-installed mod's `tool.check` can override a non-managed hook's deny.
   - Claude Code's own Edit tool allows editing a file that changed on disk since Claude read it, as long as `old_string` still matches exactly ([Edit tool behavior](https://code.claude.com/docs/en/tools-reference#edit-tool-behavior)). Claude Code alone does not prevent cross-session clobbers.
7. **Concurrency is guaranteed.** Hooks run in parallel within a session, PostToolUse fires concurrently for parallel tool calls, background subagents run alongside the main agent, and several sessions run as separate processes. Every hook invocation is a fresh process, so any shared ownership state must be safe under concurrent writers from many processes (see "Not yet specified" in the map).
8. **The latency budget is on the hot path.** PreToolUse and PostToolUse run synchronously around every matched tool call, and each spawns a process unless `async` is set (async cannot decide anything). `SessionEnd` gets 1.5 s, and plugin-declared timeouts do not extend it. So final cleanup at session end cannot depend on slow work, and a session can also die without `SessionEnd` firing (crash or kill) **(unverified that it never fires; not documented)**. `claude agents --json` (`sessionId` plus `pid`) is a supported way to detect which sessions are still alive.
9. **Context injection gives a no-new-CLI channel.** `SessionStart`/`additionalContext` and PreToolUse deny reasons can tell the agent about ownership without the agent learning a new CLI. Texts are capped at 10,000 characters and should be phrased as facts. Mid-session injections are replayed stale on resume.
10. **Distribution as a plugin fits "install once".** It needs a git-hosted marketplace. Project scope commits `enabledPlugins`, but each collaborator still runs an install once. Organizations can block plugin hooks (`allowManagedHooksOnly`), and `--bare` disables them. Several capabilities above are version-gated (`onFailure` 2.1.295, `bashEditDiff` 2.1.269, `mcp_server` 2.1.274, `scratchpad_dir` 2.1.257), so a minimum Claude Code version is a real spec input.

## Unverified points

- When and how `MultiEdit` was removed. It is simply absent from current docs and the changelog.
- Whether `NotebookEdit.notebook_path` is always absolute.
- Subagent hooks carry the parent's `session_id`: implied by examples and transcript layout, not stated in one sentence.
- Compaction keeps the same `session_id`: implied.
- Which `session_id` hooks of a background subagent report after the parent runs `/branch` or `/clear` mid-flight.
- Whether split-pane agent-team teammates get their own `session_id`.
- Whether hooks from different sessions run concurrently: inferred from separate processes, not documented.
- Whether write tools from one agent ever run in parallel.
- Whether `!` shell-mode commands fire PreToolUse/PostToolUse.
- Whether `/rewind` (Restore code) fires any hook.
- Whether `FileChanged` can watch a directory tree.
- Whether plugin hooks wait for workspace trust like settings-file hooks.
- Whether `SessionEnd` is skipped on crash or kill.

A short throwaway spike (`claude -p` with logging hooks on PreToolUse, PostToolUse, SubagentStart, SessionStart and SessionEnd, plus `/compact` and `--fork-session`) would settle the identity questions empirically.
