# Design Document: agent

**Status:** Draft
**Created:** 2026-08-26

## Overview

`agent` is a CLI launcher for AI coding agents (Claude Code, Cursor, Codex, Gemini CLI, OpenCode, ...) with shared configuration. It owns provider/credential management and agent selection, and delegates instruction/skill/MCP/hook config generation to [agents-compat](https://github.com/atvari-eu/agents-compat), which it invokes before every launch to guarantee the target agent sees up-to-date generated config.

## Problem

Running more than one AI coding agent day to day means juggling separate logins, separate API key exports, separate flags for model/reasoning-effort selection, and separately maintained rules/skills/MCP config per tool — the last of which `agents-compat` already solves. What's missing is a single entry point that:

- picks a provider, model, and variant once, in a syntax that's the same regardless of which agent CLI ends up running
- knows which providers are portable across agents and which are locked to one (e.g. a Claude subscription only works with Claude Code) and fails fast with a clear message instead of forwarding a confusing error from the underlying CLI
- guarantees the underlying agent's generated config (`CLAUDE.md`, skills symlinks, MCP config, hooks) is synced before it starts, so you never launch against stale config

## Design Principles

1. **agents-compat stays the source of truth for standards** — `agent` does not duplicate AGENTS.md/skills/MCP/hooks generation logic; it shells out to (or links) `agents-compat` for all of that.
2. **Provider identity is orthogonal to agent identity** — a provider (credential + backend) and an agent (CLI binary + its config format) are separate concepts that combine, with an explicit compatibility check between them.
3. **Fail before exec, not after** — invalid provider/model/variant/agent combinations are rejected by `agent` with a specific message; nothing gets forwarded to the underlying CLI that would fail there instead.
4. **Convenience commands write canonical sources, not generated files** — `agent skill/hook/mcp add` write to the same canonical locations `agents-compat` scans (`.agents/skills/`, `.agents/mcp.json`, `.agents/hooks.json`), then trigger a sync. They never touch agent-specific generated output directly.
5. **Opt-in scope** — first run lets the user pick which `agents-compat` features they want active; `agent` should be useful even for someone who only wants the provider/launcher part.

## Supported Agents

Same set as `agents-compat` supports as generation targets — this list grows in lockstep with it:

| Agent | Launch binary | Externally-configurable provider |
|---|---|---|
| Claude Code | `claude` | Only for providers compatible with Claude Code's provider interface |
| Gemini CLI | `gemini` | Yes |
| Cursor | `cursor-agent` | Yes |
| OpenCode | `opencode` | Yes |
| Codex | `codex` | Yes |

## Architecture

### Crate Structure

```
agent/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── cli.rs                  # Clap parser
│   ├── error.rs                # Error types + exit codes
│   ├── config.rs               # ~/.config/agent/config.toml: defaults, enabled features
│   ├── providers/
│   │   ├── mod.rs              # Provider trait, ProviderScope (Global | AgentLocked)
│   │   ├── store.rs            # Credential storage (keyring, fallback to file)
│   │   ├── anthropic.rs        # API key + Claude subscription (OAuth device flow)
│   │   ├── openai.rs
│   │   └── ...
│   ├── agents/
│   │   ├── mod.rs              # Launch trait: binary name, provider/model/variant mapping
│   │   ├── claude.rs
│   │   ├── cursor.rs
│   │   ├── gemini.rs
│   │   ├── opencode.rs
│   │   └── codex.rs
│   ├── compat.rs               # agents-compat invocation (sync before launch)
│   ├── init.rs                 # First-run feature-selection wizard
│   └── launch.rs               # resolve -> validate -> sync -> exec
├── tests/
│   ├── integration/
│   └── fixtures/
```

### Dependencies

```toml
[dependencies]
clap = { version = "4", features = ["derive", "env"] }
clap_complete = "4"
anyhow = "1"
thiserror = "2"
serde = { version = "1", features = ["derive"] }
toml = "0.8"
agents-compat = { git = "https://github.com/atvari-eu/agents-compat" }
keyring = "3"
dialoguer = "0.11"
colored = "2"

[dev-dependencies]
assert_cmd = "2"
predicates = "3"
tempfile = "3"
```

`agent` depends on `agents-compat` as a library where its API allows (config discovery, sync), and falls back to shelling out to the `agents-compat` binary for anything not exposed as a library call yet.

### CLI Interface

```
agent [OPTIONS] [-- <AGENT_ARGS>...]

Options:
  --provider <PROVIDER>       Provider id (e.g. anthropic, claude-subscription, openai)
  --model <MODEL>
  --variant <VARIANT>         e.g. reasoning-effort / thinking variant
  --agent <AGENT>             Target agent to launch [claude|cursor|gemini|opencode|codex]
  --project-root <PATH>

Commands:
  init                         First-run wizard: pick enabled agents-compat features
  provider add <ID> [--api-key | --subscription]
  provider remove <ID>
  provider reauth <ID>
  provider list
  skill add|remove|list <NAME>
  hook add|remove|list <NAME>
  mcp add|remove|list <NAME> [-- <SERVER_COMMAND>...]
  doctor                       Show resolved config, compatibility checks, sync status
  completions <SHELL>
```

Anything after a literal ` -- ` is passed through unmodified to the underlying agent CLI, after `agent`'s own flags have been parsed.

### Core Traits

```rust
trait Provider {
    fn id(&self) -> &str;
    fn scope(&self) -> ProviderScope;      // Global | AgentLocked(AgentId)
    fn authenticate(&self) -> Result<Credential>;
    fn reauthenticate(&self, cred: &Credential) -> Result<Credential>;
}

trait LaunchableAgent {
    fn id(&self) -> AgentId;
    fn binary(&self) -> &str;
    fn accepts(&self, provider: &dyn Provider) -> bool;
    fn apply_provider(&self, cred: &Credential, model: &str, variant: Option<&str>) -> LaunchEnv;
}
```

`LaunchEnv` carries the env vars, config-file writes, or CLI flags needed to hand the resolved provider/model/variant to a specific agent binary — each `agents/*.rs` module implements the translation for its agent.

## Use Case 1: Provider Compatibility Validation

Providers declare a `ProviderScope`:

- `Global` — usable with any agent that accepts externally supplied credentials for that backend (e.g. an Anthropic API key works with `opencode` and, where supported, `claude`).
- `AgentLocked(AgentId)` — usable only with one specific agent (e.g. `claude-subscription` is a Claude Code OAuth session and cannot authenticate any other CLI).

Before launching, `agent` checks `provider.scope()` against the requested `--agent`. A locked provider requested against a different agent produces a specific, actionable error, e.g.:

```
error: provider `claude-subscription` can only be used with `--agent claude`
       (requested: --agent opencode)
```

This check happens before `agents-compat sync` runs and before any subprocess is spawned — no partially-started agent process, no confusing native error from the underlying CLI.

## Use Case 2: Launching an Agent

1. **Resolve** `--provider`/`--model`/`--variant`/`--agent` from flags, falling back to config defaults (`~/.config/agent/config.toml`).
2. **Validate** the provider/agent pairing (Use Case 1).
3. **Sync** by invoking `agents-compat` scoped to `--project-root` (or the auto-detected project root), respecting the feature set chosen at `agent init` (Use Case 4).
4. **Translate** the resolved provider credential, model, and variant into the target agent's `LaunchEnv` via `LaunchableAgent::apply_provider`.
5. **Exec** the target binary (replacing the current process), forwarding everything after ` -- ` verbatim.

## Use Case 3: Managing Skills, Hooks, and MCP Servers

`agent skill|hook|mcp add|remove|list` are convenience commands that write to the same canonical locations `agents-compat` already scans:

- `agent skill add <name>` scaffolds `.agents/skills/<name>/SKILL.md`
- `agent hook add <name>` writes an entry to the canonical hooks source
- `agent mcp add <name> -- <command...>` writes an entry to `.agents/mcp.json`

Each of these triggers an `agents-compat sync` afterward so generated per-agent files (`.claude/skills/`, `.cursor/mcp.json`, etc.) reflect the change immediately, without requiring a separate manual sync step.

## Use Case 4: First-Run Feature Selection

On first run (or via `agent init`), `agent` walks the user through which `agents-compat` features to keep active for the current project/user: rules bridging, skills symlinks, MCP config sync, hooks/permissions translation. The selection is stored in `agent`'s own config and applied as a filter around each `agents-compat sync` invocation before every launch, so a user who, say, only wants shared MCP config and not rules bridging can opt out of the rest.

## Open Questions

- `agents-compat`'s CLI currently only exposes `--agent` filtering (`scan|generate|sync|clean|status|watch`), not a per-feature filter. Selective feature sync (Use Case 4) needs either a new `agents-compat` flag/config file, or `agent` needs to call finer-grained library functions directly rather than the `sync` subcommand as a whole. Track as a coordination item with `agents-compat`.
- Credential storage backend (OS keyring vs. encrypted file vs. plain file with a warning) needs a decision before `providers/store.rs` is implemented — affects portability to headless/CI environments.
- OAuth device-flow details for subscription-based providers (e.g. Claude subscription) depend on what each agent's own auth flow exposes; may not be scriptable for every agent.
