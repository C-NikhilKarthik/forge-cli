# Conventions

How code is written in this repo. Keep new code consistent with what's here;
when in doubt, match the surrounding file.

## Language & toolchain

- **C++17.** No compiler-specific extensions; must build under both `clang++`
  (macOS) and `g++` (Linux).
- Build flags: `-std=c++17 -Wall -Wextra -Wpedantic -O2 -g`. The build must be
  **warning-clean** — treat warnings as bugs.
- Format with `clang-format` (`make format`). Optionally lint with `clang-tidy`.

## Module layout

Mirror the structure in [plan.md](plan.md):

- Headers in `include/forge/`, implementations in `src/`. One `.cpp` per
  subcommand under `src/commands/`.
- `cli.cpp` owns CLI11 setup only — it parses and dispatches, no business logic.
- A command function takes parsed args + loaded config/state, does the work, and
  throws on failure. It returns `void` (or an exit code); errors propagate as
  exceptions to `main`.
- `provider.hpp` defines the abstract `Provider` base class and the
  provider-neutral structs (`Offer`, `Instance`, `CreateSpec`, `OfferQuery`).
  Each provider is its own `.cpp`/subclass; provider code maps the provider's
  JSON to the neutral structs and never leaks provider-specific types upward.
- Keep the dependency direction one-way: `main → cli → commands → {providers,
  config, state, ssh, vscode, http}`. No cycles.

## Resource management (RAII first)

This is the heart of doing it in C++ — get it right.

- **Every** owned OS resource lives in a class whose destructor releases it:
  `CURL*` (`curl_easy_cleanup`), socket fds (`close`), file handles, etc. Never
  release in the middle of a function by hand and rely on reaching the end.
- Use `std::unique_ptr` (with custom deleters where needed) and `std::string`/
  `std::vector` — no raw `new`/`delete`, no manual buffer juggling.
- `curl_global_init`/`curl_global_cleanup` exactly once, guarded by an RAII
  object created in `main`.
- Rule of zero: prefer classes that need no hand-written destructor/copy/move
  because their members are already RAII. When you do write one, follow the rule
  of five.

## Errors

- Throw, don't return codes, for application flow. Exception hierarchy in
  `error.hpp`: `ForgeError : std::runtime_error`, with `ApiError`, `NotFound`,
  `Timeout`, `ConfigError` for cases callers branch on.
- `main.cpp` has the single top-level `try/catch`: print a clean, actionable
  message to `stderr` and return non-zero. Nowhere else swallows exceptions
  silently.
- At syscall/library boundaries, check the return value, capture `errno`, and
  throw with context: `throw ForgeError(fmt::format("spawn ssh-keygen: {}",
  strerror(errno)))`. Same for non-2xx HTTP (`ApiError` with status + body
  snippet).
- User-facing messages say what failed *and* what to do (e.g. "no offer matched
  RTX_4090 under $0.50/hr — raise --max-dph or pick another --gpu").

## Style & naming

- Types `PascalCase`; functions/variables `snake_case`; constants/enumerators
  `kPascalCase` or `SCREAMING_SNAKE` (pick one, stay consistent); member fields
  trailing underscore (`ssh_port_`).
- Subcommands and flags lowercase, kebab where multi-word (`--max-dph`).
- Pass by `const&` for non-owned inputs; return by value (move/RVO). Prefer
  `std::string_view` for read-only string params where lifetime is clear.
- `#pragma once` in headers. Include what you use; keep header includes minimal
  (forward-declare where possible).
- One-line `///` (or `//`) doc on public declarations. Comment the *why*, not the
  *what*; match surrounding density.

## CLI UX

- Every command works from sensible defaults; flags override.
- Destructive commands (`rm`) confirm unless `--yes`.
- Show progress for anything >1s (a simple spinner). Quiet on success unless
  there's something worth saying; `-v` raises verbosity.
- Cost is always visible where money is spent (offer price on `up`, `$/HR` in
  `ls`).

## Secrets & safety

- API keys come from `$VAST_API_KEY` or `~/.config/forge/config.toml` — never
  hardcode, never log, never commit.
- `~/.ssh/config` is edited only inside marked `# >>> forge:<name> >>>` blocks,
  written atomically (temp file + `rename`); never rewrite the whole file
  in place.

## Dependencies

- Vendor header-only libs under `third_party/` (committed). Don't add a package
  manager for v0.1. The only link dependency is `libcurl`.
- New third-party code needs a real justification — prefer the standard library
  and POSIX.

## Tests

- Unit-test pure logic (config parsing, state (de)serialization, `~/.ssh/config`
  block insert/update/remove, offer selection) without network.
- Provider HTTP: test JSON mapping against captured fixture responses. Gate any
  live-API test behind an env flag so the default test run stays offline and
  free.
- Run `valgrind`/ASan periodically to confirm the RAII wrappers leak nothing.

## Commits

- Imperative, concise subject (≤72 chars): `add vast offer search`,
  `fix ssh-config block removal on rm`.
- Conventional-commit prefixes welcome but not required (`feat:`, `fix:`,
  `docs:`, `refactor:`).
- One logical change per commit. Update [plan.md](plan.md) in the same commit
  when scope or behavior changes.
