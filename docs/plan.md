# Forge — plan & roadmap

Living document. **v0.1** is the active milestone; later phases are sketched at
the end and share the same `Provider` abstraction. This is the source of truth
for scope — don't expand beyond the current phase without updating it here first.

> **Implementation language: C++17**, built with a hand-written Makefile. This
> is a deliberate learning choice (low-level systems work in a language we know),
> with a planned eventual port to Rust. The architecture is kept port-friendly:
> the `Provider` abstract base class maps directly onto a Rust trait later.

## v0.1 scope (active)

The minimal magical loop against **Vast.ai**:

```
forge up train    →  forge code train      (coding on a GPU in ~2 min)
forge ls          →  forge ssh train       (shell in)
forge rm train    (tear it down)
```

Includes SSH-key provisioning and `~/.ssh/config` management. **Excludes** sync,
templates, workflows, TTL, and other providers (see roadmap).

### Prerequisites

- A C++17 compiler (`clang++` on macOS, `g++` on Linux) and `make`.
- **libcurl** dev headers/lib (`-lcurl`) on the system.
- `ssh`, `ssh-keygen`, `rsync`, and `code` (the VS Code CLI) on `PATH` — Forge
  shells out to them.

## Systems concepts you'll touch (the point of doing this in C++)

| Concept | Where it shows up |
|---|---|
| Process spawning & lifecycle | `fork`/`execvp`/`waitpid` and `posix_spawn` to run `ssh-keygen`, `rsync`, `code`; bare `execvp` (no fork) for `forge ssh` so ssh takes over the terminal |
| BSD sockets | SSH readiness probe: `getaddrinfo` → `socket` → non-blocking `connect` + `poll` with a timeout |
| File descriptors & RAII | wrapping `CURL*`, socket fds, and file handles in classes that clean up in their destructor |
| `errno` / return-code handling | converting failed syscalls into exceptions with context |
| Filesystem & XDG paths | `std::filesystem` + `getenv` to resolve `~/.config`, `~/.local/share`, `~/.ssh` |

## Architecture

A **single binary** built from modular translation units. Layout:

```
forge-cli/
├── Makefile
├── third_party/              # vendored single-header libraries (committed)
│   ├── json.hpp              # nlohmann/json
│   ├── toml.hpp              # toml++ (toml11/marzer toml++)
│   ├── CLI11.hpp             # CLI11 argument parser
│   └── fmt/                  # fmtlib, header-only mode
├── include/forge/            # public headers (.hpp)
│   ├── cli.hpp  config.hpp  state.hpp  ssh.hpp  vscode.hpp  http.hpp
│   ├── error.hpp             # ForgeError (std::runtime_error) hierarchy
│   ├── provider.hpp          # abstract Provider + neutral structs
│   └── providers/vast.hpp
└── src/
    ├── main.cpp              # parse args, dispatch, top-level try/catch
    ├── cli.cpp               # CLI11 setup: subcommands + flags
    ├── config.cpp state.cpp ssh.cpp vscode.cpp http.cpp
    ├── commands/             # one .cpp per subcommand
    │   ├── up.cpp ls.cpp ssh.cpp code.cpp rm.cpp
    └── providers/
        └── vast.cpp          # VastProvider : Provider
```

Dependency direction is one-way: `main → cli → commands → {providers, config,
state, ssh, vscode, http}`. Leaf utilities don't depend on `commands/`.

### Libraries

| Concern | Choice | Notes |
|---|---|---|
| Arg parsing | **CLI11** (vendored header) | clean subcommands + flags |
| HTTP | **libcurl** (system, `-lcurl`) | wrap `CURL*` in an RAII `HttpClient` |
| JSON | **nlohmann/json** (vendored header) | parse API responses, (de)serialize state.json |
| TOML config | **toml++** (vendored header) | parse `config.toml` |
| Formatting/printing | **fmt** (vendored, `FMT_HEADER_ONLY`) | nicer than iostreams; `std::format` is C++20 |
| Process / sockets / fs | POSIX + libstdc++ | `<spawn.h>`, `<unistd.h>`, `<sys/socket.h>`, `<netdb.h>`, `<poll.h>`, `std::filesystem` |

Vendoring the header-only libs (committed under `third_party/`) keeps the build
a single `make` with no package manager. The only external link dependency is
`libcurl`.

### Errors

- A small exception hierarchy in `error.hpp`: `ForgeError : std::runtime_error`,
  with `ApiError`, `NotFound`, `Timeout`, `ConfigError` subclasses for cases
  callers branch on (parallels Rust's `thiserror`).
- Commands throw on failure; `main.cpp` has a single top-level `try/catch` that
  prints a clean, actionable message and returns a non-zero exit code (parallels
  `anyhow` at the app edge).
- At syscall boundaries, check the return value, capture `errno`, and throw a
  `ForgeError` with context (`fmt::format("ssh-keygen failed: {}", strerror(errno))`).

## Vast.ai API reference

- **Base URL:** `https://console.vast.ai`
- **Auth header:** `Authorization: Bearer <VAST_API_KEY>`

| Action | Method + path | Notes |
|---|---|---|
| Search offers | `POST /api/v0/bundles/` | filter JSON; returns `{offers:[{id, ask_contract_id, gpu_name, num_gpus, gpu_ram, dph_total, reliability, geolocation, ...}]}`. `id` is the ask id used to create. |
| Create instance | `PUT /api/v0/asks/{id}/` | body: `{image, disk, runtype:"ssh", label, onstart?, env?}`. Returns `{success, new_contract}` (the new instance id). |
| List instances | `GET /api/v0/instances/` | `{instances:[{id, actual_status, cur_state, ssh_host, ssh_port, public_ipaddr, ports, gpu_name, num_gpus, dph_total, label, image_uuid}]}` |
| Show instance | `GET /api/v0/instances/{id}/` | one instance; poll for readiness |
| Stop instance | `PUT /api/v0/instances/{id}/` | body `{"state":"stopped"}` |
| Destroy instance | `DELETE /api/v0/instances/{id}/` | full teardown |
| Attach SSH key | `POST /api/v0/instances/{id}/ssh/` | body `{"ssh_key":"ssh-ed25519 AAAA…"}` |

**Search filter example** (RTX 4090, ≤$0.50/hr, on-demand, cheapest first):

```json
{ "q": { "gpu_name": {"in":["RTX_4090"]}, "num_gpus": {"gte":1},
         "dph_total": {"lte":0.5}, "reliability": {"gte":0.98},
         "type":"ondemand", "order":[["dph_total","asc"]], "limit":20 } }
```

> "ask contract" = a marketplace offer. To create: search → take an offer's
> `id` → `PUT /api/v0/asks/{id}/`. No bidding required for `type:"ondemand"`.

## Data model

**`config.toml`** at `$XDG_CONFIG_HOME/forge/config.toml` (else
`~/.config/forge/config.toml`):

```toml
[vast]
api_key = "..."           # else read $VAST_API_KEY
[defaults]
image   = "pytorch/pytorch:latest"
gpu     = "RTX_4090"
disk    = 40              # GB
max_dph = 0.5             # $/hr ceiling for offer search
```

**`state.json`** at `$XDG_DATA_HOME/forge/state.json` (else
`~/.local/share/forge/state.json`) — local source of truth mapping a friendly
name → provider instance. Lets `forge ssh train` work without a provider
round-trip:

```json
{ "machines": { "train": {
    "provider": "vast", "instance_id": 1234567,
    "ssh_host": "ssh5.vast.ai", "ssh_port": 12345,
    "key_path": "~/.ssh/forge_train", "label": "forge-train",
    "created_at": "2026-06-09T...", "status": "running" } } }
```

## Provider abstraction (`include/forge/provider.hpp`)

Provider-neutral structs + an abstract base class; `VastProvider` maps Vast's
JSON onto the structs. This is the seam a Rust `trait` would later replace.

```cpp
struct Offer    { uint64_t id; std::string gpu_name; int num_gpus;
                  double dph_total, reliability; std::string geolocation; };
struct Instance { uint64_t id; std::string status, ssh_host, public_ip,
                  gpu_name, label; int ssh_port; double dph_total; };
struct CreateSpec { std::string image, label, onstart; int disk; };
struct OfferQuery { std::string gpu; int num_gpus; double max_dph, min_reliability; };

class Provider {
public:
    virtual ~Provider() = default;
    virtual std::vector<Offer>    search_offers(const OfferQuery&)            = 0;
    virtual uint64_t              create(uint64_t offer_id, const CreateSpec&)= 0;
    virtual std::vector<Instance> list()                                      = 0;
    virtual Instance              get(uint64_t id)                            = 0;
    virtual void                  stop(uint64_t id)                           = 0;
    virtual void                  destroy(uint64_t id)                        = 0;
    virtual void                  attach_ssh_key(uint64_t id,
                                                 const std::string& pubkey)   = 0;
};
```

Construction returns `std::unique_ptr<Provider>`, chosen from
`state.machines[name].provider`. Adding RunPod/Lambda later = one new
`Provider` subclass.

`HttpClient` (in `http.hpp`) wraps a `CURL*` (RAII) and exposes
`get/post/put/del(path, json_body)` returning `{long status, json body}`, with
the bearer header set once.

## Command behavior (v0.1)

**`forge up <name> [--gpu RTX_4090] [--image …] [--disk 40] [--max-dph 0.5]`**
1. Refuse if `<name>` already in `state.json` (suggest `forge rm` first).
2. `search_offers` with filters from flags/config defaults → pick cheapest
   reliable offer (print which: gpu, $/hr, location).
3. Ensure local keypair `~/.ssh/forge_<name>` — spawn `ssh-keygen -t ed25519 -N
   "" -f …` (via `posix_spawn`/`fork`+`exec`), only if missing.
4. `create(offer_id, spec)` with `runtype:"ssh"`, `label:"forge-<name>"`.
5. **Boot wait** (spinner): poll `get(id)` on an interval until `actual_status
   == "running"` AND `ssh_host`/`ssh_port` are populated (timeout ~5 min; on
   timeout leave the instance running so the user can retry/inspect).
6. `attach_ssh_key(id, <pubkey>)`.
7. **SSH readiness probe** (`ssh.cpp`): TCP-connect to `ssh_host:ssh_port` via
   sockets (non-blocking `connect` + `poll`), retry until it answers.
8. Write managed `~/.ssh/config` block (below).
9. Persist to `state.json`. Print: `✓ train ready → forge code train`.

**`forge ls`** — `provider.list()`, reconcile with `state.json` (flag drift:
locally-known machines missing remotely, or unmanaged remote instances). Print a
hand-formatted table: NAME, STATUS, GPU, $/HR, SSH, AGE.

**`forge ssh <name>`** — look up `state.json`, then `execvp("ssh", {"ssh",
name})` so ssh replaces the Forge process and owns the terminal; fall back to
explicit `ssh -i key -p port root@host`.

**`forge code <name>`** — ensure `~/.ssh/config` entry exists, then spawn `code
--remote ssh-remote+<name> /workspace` (default remote dir configurable).

**`forge rm <name> [--keep-key]`** — `provider.destroy(id)`, remove the
`~/.ssh/config` managed block, drop from `state.json`, optionally delete the
keypair. Confirm before destroy unless `--yes`.

## SSH key + `~/.ssh/config` strategy (`ssh.cpp`)

- **Keys:** dedicated per-machine `~/.ssh/forge_<name>` (+`.pub`), ed25519,
  generated by spawning `ssh-keygen` (no crypto lib). The public key is what we
  `attach_ssh_key` to the instance.
- **Config:** write an **idempotent managed block** delimited by markers so we
  update/remove cleanly without clobbering the user's file. Implementation: read
  the file, splice out any existing `# >>> forge:<name> >>>` … `# <<< … <<<`
  region, append the fresh block, write atomically (write temp + `rename`):

  ```
  # >>> forge:train >>>
  Host train
      HostName ssh5.vast.ai
      Port 12345
      User root
      IdentityFile ~/.ssh/forge_train
      StrictHostKeyChecking accept-new
      UserKnownHostsFile ~/.ssh/known_hosts
  # <<< forge:train <<<
  ```

  Vast often rotates `ssh_host`/`ssh_port` between stop/start, so the block is
  rewritten on every `up`. `accept-new` avoids host-key prompts on fresh boxes.

## Build (Makefile)

- `clang++`/`g++` with `-std=c++17 -Wall -Wextra -Wpedantic -O2 -g`, includes
  `-Iinclude -Ithird_party`, links `-lcurl`.
- Compile each `src/**/*.cpp` → `build/*.o`, link into `./forge`.
- Targets: `all` (default), `clean`, `run` (`./forge`), `format` (clang-format),
  optionally `tidy` (clang-tidy).

## Verification (end-to-end, against a real account)

1. **Build:** `make` produces `./forge`; `./forge --help` lists commands.
2. **Auth/offers (no spend):** confirm `search_offers` returns real offers and we
   pick a sane one. Add a hidden `forge offers --gpu RTX_4090` debug subcommand,
   or `-v` log the chosen offer before create.
3. **Full loop (small spend):** cheapest offer → `forge up test` → "ready" in a
   few minutes → `forge ls` shows it → `forge ssh test` opens a shell → `forge
   code test` opens VS Code remote → `forge rm test` destroys it. **Confirm in
   the Vast console it's gone.**
4. **Edges:** `forge up test` twice (second errors cleanly); `forge ssh missing`
   (clean "unknown machine"); Ctrl-C during boot wait leaves a recoverable state.
5. **Hygiene:** build is warning-clean; run under `valgrind`/ASan once to confirm
   no leaks in the curl/socket/fd RAII wrappers.

## Roadmap (post-v0.1 — same abstraction, layered on)

- **Sync:** `forge sync <name>` → spawn `rsync` over the SSH config entry
  (`--exclude .git/__pycache__`); `forge watch` adds a debounced file-watch loop
  (`kqueue` on macOS / `inotify` on Linux — more good systems learning).
- **Templates:** `.forge/<name>.yaml` (provider/gpu/image/setup/ports/ttl) read
  by `up`.
- **Workflows:** map declarative `setup/datasets/checkpoints/run` into the Vast
  `onstart` script (≤4048 chars) or a generated remote bootstrap; `forge run
  <name>` + `forge logs <name>` (tmux/journalctl/docker logs).
- **TTL:** `--ttl 8h` in state; a background reconciler (or installed cron)
  auto-stops past-deadline machines.
- **More providers:** RunPod (`POST https://rest.runpod.io/v1/pods`), Lambda, a
  local mock for tests.
- **Eventual Rust port:** once the surface stabilizes, port module-by-module;
  the `Provider` base class becomes a trait, RAII wrappers become `Drop`,
  exceptions become `Result`.

## Risks / notes

- Confirm exact search-offers field names (`id` vs `ask_contract_id`) against one
  live response before wiring `create` — log the raw response once during step-2.
- Vast stop/start may not preserve the SSH endpoint; v0.1 treats `rm` as destroy
  and rewrites SSH config on every `up`, sidestepping the stop/start ambiguity.
- macOS ships libcurl but headers come from the Xcode Command Line Tools (or
  Homebrew `curl`); CONTRIBUTING covers setup.
