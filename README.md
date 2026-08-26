# agent

An AI agent launcher with shared configuration, powered by [agents-compat](https://github.com/atvari-eu/agents-compat).

## Status

**Work in progress** — no usable code yet. See [docs/design.md](docs/design.md) for the planned design.

## Motivation

Every AI coding agent (Claude Code, Cursor, Codex, Gemini CLI, OpenCode, ...) has its own config format, its own auth flow, and its own CLI conventions. `agents-compat` already solves shared instructions, skills, and MCP config by generating agent-specific files from open standards. `agent` builds on top of that: a single launcher command that keeps those generated configs current and then starts whichever agent CLI you asked for, with a consistent way to pick a provider, model, and variant across all of them.

## Planned Features

- Launch any agent supported by `agents-compat`, syncing its generated config beforehand
- Manage providers (add, remove, reauthenticate) independent of any single agent
- Select provider, model, variant, and agent via flags, validated against the current configuration
- Reject agent-locked providers (e.g. a Claude subscription) when used with an incompatible agent, with a clear error instead of a forwarded failure
- Pass provider-specific arguments straight through to the underlying agent CLI after ` -- `
- Manage skills, hooks, and MCP servers, synced via `agents-compat`
- First-run wizard to enable/disable individual `agents-compat` features

## Supported Agents

Every agent supported by `agents-compat` is a planned launch target:

| Agent | Status |
|---|---|
| Claude Code | Planned |
| Gemini CLI | Planned |
| Cursor | Planned |
| OpenCode | Planned |
| Codex | Planned |

## License

[MIT](LICENSE) © [atvari GmbH](https://atvari.eu)
