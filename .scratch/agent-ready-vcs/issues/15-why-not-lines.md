# The "why not GitButler, jj or Scion?" lines

Type: grilling
Status: open
Blocked by: 12, 13, 14
Map: [Agent-ready version control: v0 spec](../map.md)

## Question

The destination asks for a one-line answer to "why not GitButler or jj?". We now share GitButler's branch model ([ADR 0002](../../../docs/adr/0002-a-branch-per-session.md)), so that line must rest on other ground. What exactly does the spec say?

- **GitButler:** candidate from [the GitButler workspace research](11-gitbutler-workspace.md): *"GitButler leaves each agent to decide which changes are its own and asks agents to use `but` instead of git. We record ownership of every change, including Bash and hand edits, keep plain git for agents, and ship under a real open-source license."* Check it against how the workspace, adjacent changes and throwing a session away get decided.
- **Scion:** add it to the destination? Candidate from [the Scion research](10-scion.md): *"Anthropic's Scion is an unreleased, in-house replacement for git's server side at monorepo scale. It does nothing for sessions sharing one folder."*
- **jj:** confirm the line still holds.
