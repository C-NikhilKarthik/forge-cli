# Forge — documentation index

Entry point for both humans and coding agents working in this repo. Read this
first, then jump to the doc you need.

## What is this

Forge is a Rust CLI for managing remote GPU machine lifecycles across cloud
providers. The north-star experience: `forge up <name>` → `forge code <name>`
puts you in VS Code on a freshly booted GPU in under two minutes. See
[context.md](context.md) for the why.

## Doc map

| Doc | Read it when you need… |
|---|---|
| [context.md](context.md) | the problem, the audience, the provider landscape, design philosophy |
| [plan.md](plan.md) | the concrete v0.1 scope, architecture, Vast.ai API reference, and the phased roadmap |
| [conventions.md](conventions.md) | how to write code here — module layout, error handling, naming, commits |
| [../CONTRIBUTING.md](../CONTRIBUTING.md) | how to set up, build, test, and open a PR |
| [../README.md](../README.md) | the public-facing pitch and quickstart |

## For agents — orient fast

- **Source of truth for scope:** [plan.md](plan.md). Do not expand scope beyond
  the current phase without saying so explicitly.
- **Before writing code:** skim [conventions.md](conventions.md). Match the
  existing module structure under `src/` (see plan.md → Architecture).
- **Provider API details** (endpoints, auth, request/response shapes) live in
  [plan.md](plan.md) → *Vast.ai API reference*. Confirm field names against one
  live response before relying on them (noted under *Risks*).
- **Secrets:** the Vast API key comes from `$VAST_API_KEY` or
  `~/.config/forge/config.toml`. Never hardcode or commit keys.
- **Costs:** commands here launch real, billed GPU instances. When testing
  end-to-end, prefer the cheapest offer and always `forge rm` (and verify in the
  Vast console) afterward.

## Repo layout (target)

```
forge-cli/
├── README.md
├── CONTRIBUTING.md
├── LICENSE
├── Cargo.toml
├── docs/                # you are here
│   ├── index.md  context.md  plan.md  conventions.md
└── src/
    ├── main.rs  cli.rs  error.rs  config.rs  state.rs  ssh.rs  vscode.rs
    ├── commands/        # up.rs ls.rs ssh.rs code.rs rm.rs
    └── providers/       # mod.rs (trait) + vast.rs
```
