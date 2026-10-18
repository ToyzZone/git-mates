# WIP that changes hands: before tracking starts and after a session ends

Type: grilling
Status: open
Blocked by: 05
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

- **Before tracking starts:** who owns WIP that already exists when tracking begins? This is the user's exact scenario: session 1 was already running with WIP when session 2 opened for a quick fix. Should the tool scan for it automatically? Who gets it by default (the Human session?), and how can it be handed to the right session?
- **After a session ends:** what happens to its WIP if it ends without committing? Does it stay with the dead session, move to the Human session, or wait until a new session (for example a resumed one) claims it?

## Comments

- From [Unit of ownership and how every change gets attributed](05-unit-of-ownership.md), 2026-10-18: a change made while two sessions' Bash calls overlap goes to the Human session, and the agent is told which lines. That WIP needs the same way back to the right session as WIP that predates tracking. Ownership is keyed by `session_id` (subagents fold into the parent); WIP across `/clear`, fork and resume is still this ticket's to decide.
