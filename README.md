# agent

An AI agent launcher with shared configuration, powered by [agents-compat](https://github.com/atvari-eu/agents-compat).

## Status

**Work in progress** — no usable code yet. See [docs/design.md](docs/design.md) for the planned design.

## Motivation

Every AI coding agent (Claude Code, Cursor, Codex, Gemini CLI, OpenCode, ...) has its own config format, its own auth flow, and its own CLI conventions. `agents-compat` already solves shared instructions, skills, and MCP config by generating agent-specific files from open standards. `agent` builds on top of that: a single launcher command that keeps those generated configs current and then starts whichever agent CLI you asked for, with a consistent way to pick a provider, model, and variant across all of them.

## Benefits

- **Switch agents without migrating config** — instructions, skills, and MCP servers are defined once and synced to whichever agent you launch, so trying a new agent CLI doesn't mean copying rules and re-wiring MCP servers by hand
- **One set of flags across every agent** — pick provider, model, and variant the same way regardless of which underlying CLI ends up running, instead of relearning each tool's own flags/env vars
- **Fail fast on incompatible setups** — agent-locked providers (e.g. a Claude subscription) are rejected before anything launches, with a clear error instead of a confusing failure from the underlying CLI
- **Never launch against stale config** — `agents-compat` sync runs before every launch, so generated files always reflect your current instructions/skills/MCP config

## Planned Features

- Launch any agent supported by `agents-compat`, syncing its generated config beforehand
- Manage providers (add, remove, reauthenticate) independent of any single agent
- Select provider, model, variant, and agent via flags, validated against the current configuration
- Reject agent-locked providers (e.g. a Claude subscription) when used with an incompatible agent, with a clear error instead of a forwarded failure
- Pass provider-specific arguments straight through to the underlying agent CLI after ` -- `
- Launch agents against local/cloud Ollama models via `ollama launch`, opt-in via `--ollama` or a global config default
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
