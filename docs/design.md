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

| Agent | Launch binary | Externally-configurable provider | `ollama launch` support |
|---|---|---|---|
| Claude Code | `claude` | Only for providers compatible with Claude Code's provider interface | Yes |
| Gemini CLI | `gemini` | Yes | No |
| Cursor | `cursor-agent` | Yes | No |
| OpenCode | `opencode` | Yes | Yes |
| Codex | `codex` | Yes | Yes |

`ollama launch` support is upstream Ollama's, not ours to extend — see [Use Case 5](#use-case-5-launching-via-ollama-launch).

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
│   ├── config.rs               # ~/.config/agent/config.toml: defaults (incl. ollama), enabled features
│   ├── providers/
│   │   ├── mod.rs              # Provider trait, ProviderScope (Global | AgentLocked)
│   │   ├── store.rs            # Credential storage (keyring, fallback to file)
│   │   ├── anthropic.rs        # API key + Claude subscription (OAuth device flow)
│   │   ├── openai.rs
│   │   └── ...
│   ├── agents/
│   │   ├── mod.rs              # Launch trait: binary name, provider/model/variant mapping, ollama-launch support
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
agent [AGENT] [OPTIONS] [-- <AGENT_ARGS>...]

Arguments:
  [AGENT]                      Target agent to launch [claude|cursor|gemini|opencode|codex] —
                                positional shorthand for --agent. If omitted and no default
                                agent is configured, `agent` prompts interactively (Use Case 6).

Options:
  --provider <PROVIDER>       Provider id (e.g. anthropic, claude-subscription, openai)
  --model <MODEL>
  --variant <VARIANT>         e.g. reasoning-effort / thinking variant
  --agent <AGENT>             Target agent to launch [claude|cursor|gemini|opencode|codex]
  --project-root <PATH>
  --ollama                    Launch via `ollama launch <agent>` instead of the agent binary directly
  --no-ollama                 Override a config default of ollama = true for this invocation

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

`agent claude` is equivalent to `agent --agent claude`; the positional form is the common case, `--agent` remains for scripts/completions that prefer explicit flags or need to place the agent after other options. Both resolve to the same `--agent` value (Use Case 2, step 1), so specifying both with conflicting values (`agent claude --agent cursor`) is a validation error, not a silent pick of one, per Design Principle 3. Agent names are reserved words in the positional slot: they cannot collide with subcommands (`init`, `provider`, `skill`, `hook`, `mcp`, `doctor`, `completions`) today, and any new subcommand must avoid colliding with a supported agent id.

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
    fn supports_ollama_launch(&self) -> bool;   // whether `ollama launch <id>` exists upstream
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

1. **Resolve** `--provider`/`--model`/`--variant`/`--agent` (or its positional shorthand, `agent <agent>`)/`--ollama` from flags, falling back to config defaults (`~/.config/agent/config.toml`). If no agent can be resolved this way, fall back further to interactive selection (Use Case 6) before continuing.
2. **Validate** the provider/agent pairing (Use Case 1), or the `--ollama`/agent pairing (Use Case 5) when `--ollama` is active.
3. **Sync** by invoking `agents-compat` scoped to `--project-root` (or the auto-detected project root), respecting the feature set chosen at `agent init` (Use Case 4). This runs regardless of launch mode — instructions/skills/MCP config apply no matter who serves the model.
4. **Translate**:
   - Default mode: the resolved provider credential, model, and variant into the target agent's `LaunchEnv` via `LaunchableAgent::apply_provider`.
   - `--ollama` mode: skip provider resolution and build an `ollama launch <agent> [--model <model>]` invocation instead (Use Case 5).
5. **Exec** the resulting command (replacing the current process), forwarding everything after ` -- ` verbatim.

## Use Case 3: Managing Skills, Hooks, and MCP Servers

`agent skill|hook|mcp add|remove|list` are convenience commands that write to the same canonical locations `agents-compat` already scans:

- `agent skill add <name>` scaffolds `.agents/skills/<name>/SKILL.md`
- `agent hook add <name>` writes an entry to the canonical hooks source
- `agent mcp add <name> -- <command...>` writes an entry to `.agents/mcp.json`

Each of these triggers an `agents-compat sync` afterward so generated per-agent files (`.claude/skills/`, `.cursor/mcp.json`, etc.) reflect the change immediately, without requiring a separate manual sync step.

## Use Case 4: First-Run Feature Selection

On first run (or via `agent init`), `agent` walks the user through which `agents-compat` features to keep active for the current project/user: rules bridging, skills symlinks, MCP config sync, hooks/permissions translation. The selection is stored in `agent`'s own config and applied as a filter around each `agents-compat sync` invocation before every launch, so a user who, say, only wants shared MCP config and not rules bridging can opt out of the rest.

## Use Case 5: Launching via `ollama launch`

[`ollama launch <agent>`](https://docs.ollama.com/cli) is Ollama's own interactive setup flow (Ollama v0.15+): it configures and starts a supported coding CLI against a local or cloud Ollama model, with no env vars or config files needed on the agent's side. As of this writing it supports `claude`, `opencode`, `codex` (plus `vscode` and `droid`, which aren't `agent` launch targets) — notably not `cursor-agent` or `gemini`.

`--ollama` is a launch-mode switch, not a provider: when active, `agent` does not resolve `--provider`/credentials at all, because Ollama supplies and serves the model itself.

1. **Enable** via `--ollama` on the command line, or `ollama = true` under `[defaults]` in `~/.config/agent/config.toml` — the flag/config value is resolved the same way as `--provider`/`--model`/etc. (Use Case 2, step 1). `--no-ollama` overrides a config default of `true` for a single invocation.
2. **Validate**, before doing anything else (no subprocess spawned yet, consistent with Design Principle 3):
   - `LaunchableAgent::supports_ollama_launch()` for the requested `--agent`. Requesting `--ollama --agent cursor` fails fast with:
     ```
     error: `--ollama` is not supported for `--agent cursor`
            (ollama launch supports: claude, opencode, codex)
     ```
     This mirrors the Use Case 1 provider-compatibility check: no partial launch, no error forwarded from `ollama` or the underlying agent CLI.
   - `--provider` and `--variant` are rejected as incompatible with `--ollama` (`--variant` has no equivalent in Ollama's launch flow), with a clear error rather than being silently ignored.
3. **Sync** still runs as normal (Use Case 2, step 3) — `--ollama` only changes how the model is served, not whether `agents-compat`-generated config is fresh.
4. **Exec** `ollama launch <agent-id>`, appending `--model <model>` if `--model` was given (letting Ollama's own TUI prompt for a model otherwise). Anything after ` -- ` is still forwarded, matching the non-`--ollama` path.

## Use Case 6: Interactive Agent Selection and Defaults

`agent` invoked with no agent resolvable from the positional argument, `--agent`, or `[defaults].agent` in `~/.config/agent/config.toml` prompts interactively rather than erroring:

1. **Prompt** — list the supported agents (see [Supported Agents](#supported-agents)) and ask the user to pick one, the same `dialoguer`-based single-select style used by `agent init` (Use Case 4). This step is skipped entirely once an agent is resolved from the positional arg, `--agent`, or the config default — it only fires when none of those apply.
2. **Offer to persist** — after picking, ask "set `<agent>` as the default agent? [y/N]". If accepted, write `agent = "<agent>"` under `[defaults]` in `~/.config/agent/config.toml`, so future no-argument invocations skip straight to step 3 without prompting.
3. **Continue the launch** — proceeds into Use Case 2 (Validate → Sync → Translate → Exec) using the picked agent, exactly as if it had been passed via `--agent`.
4. **Non-interactive contexts** — when stdin isn't a TTY (CI, scripts) and no agent can be resolved, `agent` fails fast with an actionable error (e.g. `error: no agent specified and none configured as default; pass --agent or run \`agent\` interactively`) instead of hanging on a prompt it can't render, per Design Principle 3.

For `--provider`/`--model`/`--variant`, a missing value is **not** prompted for and does not block the launch: `agent` passes nothing extra for that option and lets the target agent's own `LaunchableAgent::apply_provider` fall through to whatever that agent CLI already defaults to on its own (e.g. `claude` run with no provider override behaves exactly as `claude` invoked directly would). `agent` does, however, let a user pin cross-agent defaults for these explicitly via the same `[defaults]` table used for the launch agent and `--ollama`:

```toml
[defaults]
agent = "claude"
provider = "anthropic"
model = "claude-sonnet-5"
ollama = false
```

Any of `agent`/`provider`/`model`/`variant`/`ollama` may be set under `[defaults]`; a key left unset falls through to the underlying agent's own default rather than `agent` inventing one. There's no dedicated `agent config set` subcommand for these in this iteration — they're set either via the Use Case 6 step 2 prompt (for `agent`) or by editing `config.toml` directly (for the rest); a `provider`/`model`/`variant`-equivalent interactive default-setting flow is an open question below.

## Open Questions

- `agents-compat`'s CLI currently only exposes `--agent` filtering (`scan|generate|sync|clean|status|watch`), not a per-feature filter. Selective feature sync (Use Case 4) needs either a new `agents-compat` flag/config file, or `agent` needs to call finer-grained library functions directly rather than the `sync` subcommand as a whole. Track as a coordination item with `agents-compat`.
- Credential storage backend (OS keyring vs. encrypted file vs. plain file with a warning) needs a decision before `providers/store.rs` is implemented — affects portability to headless/CI environments.
- OAuth device-flow details for subscription-based providers (e.g. Claude subscription) depend on what each agent's own auth flow exposes; may not be scriptable for every agent.
- Whether `agent` should check for `ollama` on `PATH` and its version (`ollama launch` requires v0.15+) before exec, versus letting the exec fail and forwarding `ollama`'s own error — leaning toward a `doctor` check (non-blocking) plus a fast exec-not-found error, consistent with Design Principle 3.
- `ollama launch <agent> --config` (configure without launching) isn't exposed by `agent` yet — deferred until there's a concrete need to configure-without-launching through `agent` rather than calling `ollama launch --config` directly.
- Whether `--provider`/`--model`/`--variant` defaults (Use Case 6) need their own interactive prompt/persist flow (mirroring the agent-selection prompt), or a `agent config set <key> <value>` command, versus staying edit-`config.toml`-by-hand only, as designed today.
