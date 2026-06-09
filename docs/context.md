# Context — why Forge exists

## The problem

Renting a GPU box and actually getting work done on it is death by a thousand
manual steps. Today the flow is some mix of:

- clicking through a provider console to pick and launch an instance,
- copy-pasting SSH commands and IP/port that change every launch,
- hand-editing `~/.ssh/config`,
- `rsync`-ing code up,
- running a README's worth of setup (`pip install`, dataset download, checkpoint
  download, env vars),
- and — the expensive part — *forgetting it's running* and paying for an idle
  4090 overnight.

Every team rebuilds this as a pile of bash scripts and tribal knowledge.

## The idea

One tool that owns the whole lifecycle and makes the common path a single
command. The bar for v0.1:

```
forge up train   →   forge code train
```

should drop you into VS Code, editing on a live GPU, in under two minutes — SSH
keys, config, and boot-wait all handled for you.

After that, Forge moves *setup* to the right place — into the machine's boot —
so a box arrives developer-ready: environment built, datasets and checkpoints
already downloaded. That declarative, reproducible boot is where Forge earns its
keep versus a folder of shell scripts.

## Who it's for

ML engineers and researchers who rent GPUs on demand (Vast.ai, RunPod, Lambda)
and want their local-dev ergonomics on remote hardware without babysitting it.

## Why Rust

- First-class CLI ecosystem (`clap`, `indicatif`) and the same neighborhood as
  the tools this wants to feel like (`cargo`, `uv`, `mise`, `atuin`).
- Strong types make the provider abstraction clean; `serde` makes API + config
  handling cheap.
- `tokio` for async HTTP/polling; shell out to `ssh`/`rsync`/`code` rather than
  reimplementing them.
- Ships as a single binary — trivial cross-platform distribution.

## Provider landscape

| Provider | API shape | Trade-off |
|---|---|---|
| **Vast.ai** *(v0.1 target)* | REST, marketplace: search offers → create from an offer `id` | Cheapest GPUs; "ask/contract" is just marketplace jargon for an offer |
| RunPod | clean REST (`POST /v1/pods`) | Simplest API; good second provider |
| Lambda | simple REST (launch/terminate + SSH keys) | Fixed instance types, often capacity-constrained |

Forge hides these behind one `Provider` trait (see [plan.md](plan.md)), so the
CLI surface is identical regardless of where the machine runs.

## Design principles

1. **The happy path is one command.** Defaults that just work; flags for the
   rest.
2. **Local state is authoritative for naming.** A friendly name (`train`) maps
   to a provider instance id in local state, so day-to-day commands don't need a
   round-trip.
3. **Never clobber the user's files.** `~/.ssh/config` edits live inside
   marked, idempotent blocks.
4. **Shell out to battle-tested tools** (`ssh`, `rsync`, `code`) instead of
   reimplementing them.
5. **Money is a first-class concern.** Surface cost; make teardown and (later)
   TTL auto-shutdown easy and obvious.
