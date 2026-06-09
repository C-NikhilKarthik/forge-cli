# Forge — plan & roadmap

Living document. **v0.1** is the active milestone; later phases are sketched at
the end and share the same `Provider` trait. This is the source of truth for
scope — don't expand beyond the current phase without updating it here first.

## v0.1 scope (active)

The minimal magical loop against **Vast.ai**:

```
forge up train    →  forge code train      (coding on a GPU in ~2 min)
forge ls          →  forge ssh train       (shell in)
forge rm train    (tear it down)
```

Includes SSH-key provisioning and `~/.ssh/config` management. **Excludes** sync,
templates, workflows, TTL, and other providers (see roadmap).

### Prerequisite

`cargo`/`rustc` are installed via [rustup](https://rustup.rs). `rsync`, `code`,
and `ssh` must be on `PATH` (Forge shells out to them).

## Architecture

A **single binary crate** with modules (a multi-crate workspace is premature;
revisit if providers grow). Layout:

```
src/
├── main.rs            # tokio entry, clap dispatch, tracing init
├── cli.rs             # clap derive: Cli + Commands enum
├── error.rs           # thiserror error types (provider/io/config)
├── config.rs          # ~/.config/forge/config.toml (api keys, defaults)
├── state.rs           # ~/.local/share/forge/state.json (name → instance)
├── ssh.rs             # keygen + ~/.ssh/config managed blocks + readiness probe
├── vscode.rs          # `code --remote ssh-remote+<name>` launcher
├── commands/          # one file per subcommand (up/ls/ssh/code/rm)
│   ├── up.rs  ls.rs  ssh.rs  code.rs  rm.rs
└── providers/
    ├── mod.rs         # `Provider` trait (async-trait) + shared structs
    └── vast.rs        # VastProvider: reqwest client over the API below
```

### Crates

| Crate | Use |
|---|---|
| `clap` (derive) | CLI parsing/help |
| `tokio` (rt-multi-thread, macros) | async runtime |
| `reqwest` (json, rustls-tls) | HTTP to Vast API |
| `serde`, `serde_json` | (de)serialize API + state.json |
| `toml` | `config.toml` parsing |
| `anyhow` | app-level error context (main/commands) |
| `thiserror` | typed errors in `providers`/`ssh` |
| `async-trait` | object-safe async `Provider` trait for `dyn` dispatch |
| `indicatif` | spinner while instance boots / SSH comes up |
| `directories` | XDG-correct config/state/cache paths |
| `comfy-table` | `forge ls` table output |
| `tracing` + `tracing-subscriber` | `-v` debug logging |

> Avoid `serde_yaml` (unmaintained/archived). YAML isn't needed in v0.1; when
> templates land, use `serde_yaml_ng`.

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

**`config.toml`** (`directories::ProjectDirs("ai","forge","forge").config_dir()`):

```toml
[vast]
api_key = "..."           # else read $VAST_API_KEY
[defaults]
image   = "pytorch/pytorch:latest"
gpu     = "RTX_4090"
disk    = 40              # GB
max_dph = 0.5             # $/hr ceiling for offer search
```

**`state.json`** (data_dir) — local source of truth mapping friendly name →
provider instance. Lets `forge ssh train` work without a provider round-trip:

```json
{ "machines": { "train": {
    "provider": "vast", "instance_id": 1234567,
    "ssh_host": "ssh5.vast.ai", "ssh_port": 12345,
    "key_path": "~/.ssh/forge_train", "label": "forge-train",
    "created_at": "2026-06-09T...", "status": "running" } } }
```

## Provider trait (`providers/mod.rs`)

```rust
#[async_trait]
pub trait Provider {
    async fn search_offers(&self, req: &OfferQuery) -> Result<Vec<Offer>>;
    async fn create(&self, offer_id: u64, spec: &CreateSpec) -> Result<u64>; // instance id
    async fn list(&self) -> Result<Vec<Instance>>;
    async fn get(&self, id: u64) -> Result<Instance>;
    async fn stop(&self, id: u64) -> Result<()>;
    async fn destroy(&self, id: u64) -> Result<()>;
    async fn attach_ssh_key(&self, id: u64, pubkey: &str) -> Result<()>;
}
```

`Offer`, `Instance`, `CreateSpec`, `OfferQuery` are provider-neutral structs;
`vast.rs` maps Vast's JSON onto them. Adding RunPod/Lambda later = one new file
implementing the trait, dispatched via `Box<dyn Provider>` keyed on
`state.machines[name].provider`.

## Command behavior (v0.1)

**`forge up <name> [--gpu RTX_4090] [--image …] [--disk 40] [--max-dph 0.5]`**
1. Refuse if `<name>` already in `state.json` (suggest `forge rm` first).
2. `search_offers` with filters from flags/config defaults → pick cheapest
   reliable offer (print which: gpu, $/hr, location).
3. Ensure local keypair `~/.ssh/forge_<name>` (`ssh-keygen -t ed25519 -N "" -f
   …`, only if missing).
4. `create(offer_id, spec)` with `runtype:"ssh"`, `label:"forge-<name>"`.
5. **Boot wait** (indicatif spinner): poll `get(id)` until
   `actual_status == "running"` AND `ssh_host`/`ssh_port` are populated (timeout
   ~5 min; on timeout leave instance running so the user can retry/inspect).
6. `attach_ssh_key(id, <pubkey>)`.
7. **SSH readiness probe** (`ssh.rs`): TCP-connect to `ssh_host:ssh_port` and/or
   `ssh -o BatchMode=yes … true` with short retries until it answers.
8. Write managed `~/.ssh/config` block (below).
9. Persist to `state.json`. Print: `✓ train ready → forge code train`.

**`forge ls`** — `provider.list()`, reconcile with `state.json` (flag drift:
locally-known machines missing remotely, or unmanaged remote instances). Render
table: NAME, STATUS, GPU, $/HR, SSH, AGE.

**`forge ssh <name>`** — look up `state.json`, replace the process with `ssh
<name>` (using the config entry); fall back to explicit `ssh -i key -p port
root@host`.

**`forge code <name>`** — ensure `~/.ssh/config` entry exists, then `code
--remote ssh-remote+<name> /workspace` (default remote dir configurable).

**`forge rm <name> [--keep-key]`** — `provider.destroy(id)`, remove the
`~/.ssh/config` managed block, drop from `state.json`, optionally delete the
keypair. Confirm before destroy unless `--yes`.

## SSH key + `~/.ssh/config` strategy (`ssh.rs`)

- **Keys:** dedicated per-machine `~/.ssh/forge_<name>` (+`.pub`), ed25519,
  generated by shelling out to `ssh-keygen` (no native crypto dep). The public
  key is what we `attach_ssh_key` to the instance.
- **Config:** write an **idempotent managed block** delimited by markers so we
  update/remove cleanly without clobbering the user's file:

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

## Verification (end-to-end, against a real account)

1. **Build:** `cargo build` clean; `cargo run -- --help` lists commands.
2. **Auth/offers (no spend):** confirm `search_offers` returns real offers and we
   pick a sane one. Add a hidden `forge offers --gpu RTX_4090` debug subcommand,
   or `-v` log the chosen offer before create.
3. **Full loop (small spend):** cheapest offer → `forge up test` → "ready" in a
   few minutes → `forge ls` shows it → `forge ssh test` opens a shell → `forge
   code test` opens VS Code remote → `forge rm test` destroys it. **Confirm in
   the Vast console it's gone.**
4. **Idempotency/edges:** `forge up test` twice (second errors cleanly); `forge
   ssh missing` (clean "unknown machine"); Ctrl-C during boot wait leaves a
   recoverable state.

## Roadmap (post-v0.1 — same trait, layered on)

- **Sync:** `forge sync <name>` → `rsync` over the SSH config entry (`--exclude
  .git/__pycache__`); `forge watch` adds a debounced file-watch loop.
- **Templates:** `.forge/<name>.yaml` (provider/gpu/image/setup/ports/ttl) read
  by `up` (`serde_yaml_ng`).
- **Workflows:** map declarative `setup/datasets/checkpoints/run` into the Vast
  `onstart` script (≤4048 chars) or a generated remote bootstrap; `forge run
  <name>` + `forge logs <name>` (tmux/journalctl/docker logs).
- **TTL:** `--ttl 8h` in state; a background reconciler (or installed cron)
  auto-stops past-deadline machines.
- **More providers:** `runpod.rs` (`POST https://rest.runpod.io/v1/pods`),
  `lambda.rs`, `local.rs` (mock for tests).
- **Release:** GitHub Actions cross-compile → macOS (aarch64/x86_64), Linux
  (x86_64-gnu), Windows (x86_64-msvc).

## Risks / notes

- Confirm exact search-offers field names (`id` vs `ask_contract_id`) against one
  live response before wiring `create` — log the raw response once during step-2.
- Vast stop/start may not preserve the SSH endpoint; v0.1 treats `rm` as destroy
  and rewrites SSH config on every `up`, sidestepping the stop/start ambiguity.
