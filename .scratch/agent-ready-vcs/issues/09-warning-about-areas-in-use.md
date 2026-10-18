# Warning sessions about areas another session is editing

Type: grilling
Status: open
Blocked by: 01, 05
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

How does a session learn, before it edits, that another session is editing the same file or lines right now? And how exactly does v0 block a Clobbered edit and tell the agent? (Decided while charting: block the edit and say which session owns those lines.)

Also decide:

- What counts as "editing right now" versus stale WIP that nobody is touching.
- Whether a warning appears when a session first reads or opens an area that another session is working on, or only when it tries to write.

## Comments

- From [Unit of ownership and how every change gets attributed](05-unit-of-ownership.md), 2026-10-18: a change made while two sessions' Bash calls overlap goes to the Human session, so an agent's own output (say from `cargo fmt`) can be Human-owned. Decide whether an agent's Edit on Human-owned lines is blocked like any Clobbered edit, or allowed.
