# Contributing to Forge

Thanks for hacking on Forge! This is an early-stage project — the fastest way to
help is to build the [v0.1 milestone](docs/plan.md) and report rough edges.

## Before you start

- Read [docs/index.md](docs/index.md) → [docs/plan.md](docs/plan.md) (scope) →
  [docs/conventions.md](docs/conventions.md) (how we write code).
- Don't expand scope past the current phase in `plan.md` without updating it
  there first.

## Dev setup

1. **C++17 toolchain + make:**
   - macOS: `xcode-select --install` (gives `clang++`, `make`, and the libcurl
     headers in the SDK). Optionally `brew install curl` for a newer libcurl.
   - Linux: `sudo apt install build-essential libcurl4-openssl-dev`
     (or your distro's equivalent for `g++`, `make`, and libcurl dev headers).
2. **System tools on `PATH`:** `ssh`, `ssh-keygen`, `rsync`, and `code` (the VS
   Code CLI). Forge shells out to these.
3. **Vendored libraries** live under `third_party/` (committed single headers:
   nlohmann/json, toml++, CLI11, fmt) — nothing to install.
4. **Vast.ai credentials** (for end-to-end work): grab an API key from your
   [account keys page](https://cloud.vast.ai/account/) and set
   `export VAST_API_KEY=...` (or add it to `~/.config/forge/config.toml`).

## Build, run, check

```bash
make                              # compile → ./forge
./forge --help                    # run the CLI
./forge up test                   # exercise a command
make format                       # clang-format (run before committing)
make clean                        # remove build artifacts
```

The build must be **warning-clean** (`-Wall -Wextra -Wpedantic`). Run under
`valgrind` or AddressSanitizer occasionally to confirm the RAII wrappers (curl
handle, sockets, fds) leak nothing.

## A note on cost

Several commands launch **real, billed GPU instances**. When testing the full
loop:

- pick the cheapest offer you can,
- always `forge rm <name>` when done,
- and **verify in the [Vast console](https://cloud.vast.ai/instances/)** that the
  instance is actually gone.

Live-API tests are gated behind an env flag so the default test run stays
offline and free.

## Pull requests

1. Branch off `main`.
2. Keep PRs focused — one logical change. Update [docs/plan.md](docs/plan.md) in
   the same PR when behavior or scope changes.
3. Before pushing: `make format && make` clean (warning-free), and tests pass.
4. Write a clear description: what changed, why, and how you verified it
   (especially for anything that touches live instances or `~/.ssh/config`).

See [docs/conventions.md](docs/conventions.md) for code style, error-handling,
and commit-message conventions.

## License

By contributing, you agree your contributions are licensed under the project's
[MIT License](LICENSE). (If you'd prefer the Rust-ecosystem-standard
`MIT OR Apache-2.0` dual license, raise it in an issue before we accumulate
external contributions — it's easiest to change now.)
