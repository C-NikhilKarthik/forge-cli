# Forge

> Go from nothing to *coding on a remote GPU* in under two minutes.

**Forge** is a CLI that manages the full lifecycle of remote GPU machines across
cloud providers — find a machine, start it, push your SSH key, wait for it to
boot, and drop straight into VS Code on it. Later it grows into declarative
templates and boot-time workflows (env setup, dataset + checkpoint download) so
a box is *developer-ready* the moment it comes up.

```bash
forge up train      # find + launch a GPU, provision SSH, wait until ready
forge code train    # open VS Code remote on it
forge ssh train     # shell in
forge ls            # what's running, and what it's costing
forge rm train      # tear it down
```

Built in Rust — a single static binary, no runtime to install.

## Status

🚧 **v0.1 in progress.** First milestone is the magic loop
(`up`/`ls`/`ssh`/`code`/`rm`) against [Vast.ai](https://vast.ai). See the
[roadmap](docs/plan.md) for what comes after (sync, templates, workflows, TTL
auto-shutdown, more providers).

## Install

> Not yet published. For now, build from source — see
> [CONTRIBUTING.md](CONTRIBUTING.md).

```bash
cargo install --path .
```

## Quickstart

1. Get a Vast.ai API key from your [account keys page](https://cloud.vast.ai/account/).
2. Set it: `export VAST_API_KEY=...` (or put it in `~/.config/forge/config.toml`).
3. `forge up train` — Forge searches offers, launches the cheapest reliable GPU
   matching your defaults, attaches a freshly generated SSH key, waits for boot,
   and writes a `~/.ssh/config` entry.
4. `forge code train` — VS Code opens connected to the box.

## Documentation

- **[docs/index.md](docs/index.md)** — start here (humans and agents)
- [docs/context.md](docs/context.md) — why Forge exists, who it's for
- [docs/plan.md](docs/plan.md) — v0.1 plan + roadmap
- [docs/conventions.md](docs/conventions.md) — code style and architecture rules
- [CONTRIBUTING.md](CONTRIBUTING.md) — dev setup and how to contribute

## License

MIT — see [LICENSE](LICENSE).
