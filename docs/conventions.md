# Conventions

How code is written in this repo. Keep new code consistent with what's here;
when in doubt, match the surrounding file.

## Language & toolchain

- **Rust**, stable (latest). Pin nothing fancier than the stable channel unless a
  feature forces it.
- Format with `cargo fmt` (default rustfmt). Lint with `cargo clippy
  --all-targets`. CI treats `clippy` warnings as errors (`-D warnings`); keep the
  tree clean.

## Module layout

Mirror the structure in [plan.md](plan.md):

- `cli.rs` owns clap types only; it parses and dispatches, no business logic.
- One file per subcommand under `commands/`. A command function takes parsed
  args + loaded config/state, does the work, returns `anyhow::Result<()>`.
- `providers/mod.rs` defines the `Provider` trait and the provider-neutral
  structs (`Offer`, `Instance`, `CreateSpec`, `OfferQuery`). Each provider is its
  own file implementing the trait. Provider code maps the provider's JSON to the
  neutral structs and never leaks provider-specific types upward.
- `config.rs`, `state.rs`, `ssh.rs`, `vscode.rs` are leaf utilities — no
  dependency on `commands/`.

Keep the dependency direction one-way: `main → cli → commands → {providers,
config, state, ssh, vscode}`. No cycles.

## Errors

- **`thiserror`** for typed, matchable errors in library-ish modules
  (`providers/*`, `ssh.rs`) — callers may want to branch on the variant
  (e.g. auth failure vs. not-found vs. timeout).
- **`anyhow`** at the application edge (`commands/*`, `main.rs`). Add context
  with `.context("…")` / `.with_context(|| …)` so failures read as a story, not a
  bare error.
- User-facing errors are actionable: say what failed *and* what to do
  (e.g. "no offer matched RTX_4090 under $0.50/hr — raise `--max-dph` or pick
  another `--gpu`").
- Never `unwrap()`/`expect()` on anything that can fail at runtime (I/O, network,
  parsing). `expect()` is acceptable only for true invariants, with a message
  explaining the invariant.

## Async

- `tokio` multi-thread runtime; `#[tokio::main]` in `main.rs`.
- HTTP via a single shared `reqwest::Client` (built once, cloned cheaply).
- Poll loops (boot wait, SSH probe) have an explicit timeout and a sane interval;
  never busy-loop.

## Naming & style

- Types `UpperCamelCase`, functions/vars `snake_case`, consts `SCREAMING_SNAKE`.
- Subcommands and flags are lowercase, kebab where multi-word (`--max-dph`).
- Prefer borrowing (`&str`, `&[T]`) in function signatures over owned args.
- Public items get a one-line `///` doc comment. Comment the *why*, not the
  *what*; match the surrounding density.

## CLI UX

- Every command works from sensible defaults; flags override.
- Side-effecting/destructive commands (`rm`) confirm unless `--yes`.
- Progress for anything >1s (spinner via `indicatif`). Quiet on success unless
  there's something worth saying; `-v` raises `tracing` verbosity.
- Cost is always visible where money is spent (offer price on `up`, `$/HR` in
  `ls`).

## Secrets & safety

- API keys come from `$VAST_API_KEY` or `~/.config/forge/config.toml` — never
  hardcode, never log, never commit. `config.toml` is user-local, not in the
  repo.
- `~/.ssh/config` is edited only inside marked `# >>> forge:<name> >>>` blocks;
  never rewrite the whole file.

## Tests

- Unit-test pure logic (config parsing, state (de)serialization, `~/.ssh/config`
  block insert/update/remove, offer selection) without network.
- Provider HTTP: test the JSON mapping against captured fixture responses; gate
  any live-API test behind an env flag and `#[ignore]` so `cargo test` stays
  offline and free.

## Commits

- Imperative, concise subject (≤72 chars): `add vast offer search`,
  `fix ssh-config block removal on rm`.
- Conventional-commit prefixes are welcome but not required (`feat:`, `fix:`,
  `docs:`, `refactor:`).
- One logical change per commit. Update [plan.md](plan.md) in the same commit
  when scope or behavior changes.
