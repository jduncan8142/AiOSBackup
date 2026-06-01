# AiOSBackup — Project Plan

A Rust backup utility for AiOS and for Windows/macOS/Linux clients: encrypted,
deduplicated, point-in-time backups — full and incremental, ad-hoc and
scheduled — to local and remote (rsync.net) targets, with a GUI for settings and
restore, and every function exposed to the AiOS agent as MCP tooling.

**Status:** Planning — pre-Phase 0. The repository holds the license, the README,
and this document set; there is no code yet. The major technology forks were
resolved with Jason on 2026-05-31 (see Key Decisions).

## Vision

AiOSBackup is how data on an AiOS machine — and on the Windows, macOS, and Linux
machines around it — survives a disk failure, a ransomware event, a fat-fingered
deletion, or a stolen laptop. AiOS is an AI-first, local-first OS that encrypts
everything and trusts no cloud by default; its backup story has to honor that.
So AiOSBackup's defining property is simple and absolute: **a backup is encrypted
on the local machine, with a key derived from the user's password, before a
single byte leaves the device.** The remote end — even a trusted one like
rsync.net — only ever sees ciphertext.

It is **not just an AiOS tool.** A backup utility that only protects the one
experimental OS would protect almost nothing real. AiOSBackup is **cross-OS from
v1**: the same engine, repository format, encryption, scheduler, and agent
surface run on AiOS, Linux, Windows, and macOS. On AiOS it leans on the
platform — AiOSFSS for change detection, AiOSVault for key custody, the agent as
supervisor and operator. On other operating systems it stands alone: it watches
the filesystem itself, keeps keys in the OS keyring, and installs as a system
service.

It **does not reinvent the backup engine.** Deduplicating, encrypting,
incremental, restorable backup is decades-deep and unforgiving to get wrong — and
getting it wrong means silent data loss discovered only when a restore is
attempted in a crisis. AiOSBackup builds on **`rustic_core`**, the pure-Rust
implementation of the battle-tested **restic** repository format. The novelty
budget is spent on AiOS integration, the cross-OS story, the agent/MCP surface,
the scheduler, and the GUI — not on a bespoke crypto-and-chunking format whose
first real test is someone's lost data.

**Near-term:** a headless engine that makes and restores encrypted incremental
backups to a local disk and to rsync.net, driven by a CLI, on all four target
OSes. **Long-term:** the full product — a scheduler, continuous backup driven by
file-watching, a polished settings + restore GUI, agent-driven backup/restore
over MCP, an append-only ransomware-resistant remote tier, and deep AiOS
integration (AiOSFSS, AiOSVault, agent supervision).

## Guiding Principles

1. **Encrypted before it leaves the machine.** Client-side encryption with a
   user-set password is non-negotiable and is the product's defining property.
   The remote target sees only ciphertext; losing the remote never means losing
   confidentiality. The user owns the key; a forgotten password means
   unrecoverable data, and the UX must make that consequence unmistakable.
2. **Restore is sacred.** A backup tool is only as good as its restores. Restore
   correctness outranks every other feature; every release proves backup→restore
   round-trips with integrity checks, and the repository stays readable by
   upstream `restic`/`rustic` as a last-resort recovery path.
3. **Don't reinvent the engine.** Chunking, dedup, encryption, the snapshot
   model, and pruning are solved and dangerous to reimplement. Build on
   `rustic_core` (restic format); spend novelty on integration, not on format.
4. **Cross-OS is a first-class requirement, not a port.** AiOS, Linux, Windows,
   and macOS are all v1 targets. Platform differences (file-watching, service
   management, key storage, the SFTP-vs-rclone transport) live behind traits so
   the engine and core logic are OS-agnostic.
5. **Keys live in the vault / keyring, never in plaintext.** The repository
   password and the automated-backup SSH private key are sealed — via AiOSVault
   on AiOS, the OS keyring (Windows Credential Manager / macOS Keychain / Secret
   Service) elsewhere — and loaded into memory only when needed, then zeroized.
6. **Headless core, thin frontends.** All backup logic is a library with no UI,
   mirroring the AiOS-wide pattern (AiOSVault, AiOSPac, AiOSTerminal). The GUI,
   the CLI, the MCP server, and the scheduling daemon are all thin clients of one
   core. No business logic in a frontend.
7. **The agent is a client, and destructive actions are mediated.** The agent can
   trigger backups, list snapshots, and request restores through an MCP surface —
   but **restore overwrites data**, so agent-initiated restore is a *mediated,
   confirmed* action under F-PROMPT-SAFETY / F-AGENT-POLICY, never an autonomous
   one. The agent never holds the repository password.
8. **Restore inputs and repository metadata are untrusted data.** Snapshot
   metadata, paths, and file contents pulled from a repository (which may have
   been tampered with at the remote) are treated as hostile input on restore —
   path-traversal-safe, never interpreted as instructions (F-PROMPT-SAFETY).
9. **Local-first, cloud-optional.** A local-disk or LAN target works with no
   account and no network. rsync.net (or any remote) is additive — never a
   precondition for backing up.
10. **Multi-user-ready from day one.** Backup sets, schedules, repositories, and
    keys carry a user scope even though AiOS is single-user through M2.

## How it fits into AiOS

AiOS is a parent repo plus one component repo per major piece. The Rust system
components are AiOSCanvas (compositor), AiOSFSS (file sync), AiOSVault (secrets),
AiOSPac (packaging), and AiOSTerminal (the terminal); shared logic lives in the
`aios-libs` workspace (`aios-crypto`, `aios-ipc`, `aios-mcp`, `aios-recurrence`).
**AiOSBackup is a new peer system component** — a headless daemon + GUI, like
AiOSVault — **not** a canvas widget. Its distinguishing trait among the
components is that it is **cross-OS**: it is the one AiOS component explicitly
built to run, unchanged in its core, on Windows and macOS as well.

Its relationships are few and explicit:

- **`aios-crypto` (`aios-libs`).** Used to **seal the local repository password
  and the SSH private key** at rest on AiOS (the OS keyring fills this role on
  other OSes). It is **not** used for the backup repository's own crypto — see
  "The engine & repository format" for that deliberate boundary.
- **AiOSVault (key custody on AiOS).** On AiOS, the sealed repo password and SSH
  key are stored in and released by AiOSVault, mediated where its model allows,
  so the long-lived password need not sit in AiOSBackup's memory. On other OSes,
  the OS keyring plays this role.
- **AiOSFSS (change detection on AiOS).** On AiOS, AiOSBackup **consumes**
  AiOSFSS's change-event stream (it already runs a debounced `notify` watcher and
  exposes a JSON-RPC/event control plane) to know what changed, rather than
  running a second watcher. AiOSFSS does LAN device-to-device sync; AiOSBackup
  does point-in-time encrypted snapshots to local/remote targets — **complementary,
  not overlapping.** AiOSBackup does **not** use AiOSFSS as a transport.
- **The AiOS agent (operator, via MCP).** The agent can trigger backups, query
  snapshots and status, and request restores over an MCP surface (parent F-MCP),
  exactly the agent-as-client posture the other components take. Restore is
  mediated/confirmed. The agent supervises the AiOSBackup daemon on AiOS the way
  it supervises AiOSFSS.
- **AiOSCanvas (GUI host on AiOS).** On AiOS the egui GUI runs as a Wayland
  client; whether it also becomes an in-canvas widget surface is an open question
  (it is not required for v1).

AiOSBackup implements the parent feature **F-BACKUP** (to be minted in the parent
`FEATURES.md`), which carves the "backup strategy" clause out of the existing
**F-RECOVERY** (F-RECOVERY keeps the undo window and account/device recovery). It
owns its own `B-*` sub-feature tags (B-ENGINE, B-CRYPTO, B-REMOTE, B-KEYS,
B-SCHED, B-WATCH, B-RESTORE, B-GUI, B-AGENT, B-XPLAT, B-SAFETY), written into
`FEATURES.md` at Phase 0.

## Architecture

AiOSBackup is a headless core library with swappable frontends — the same pattern
AiOS uses elsewhere (AiOSVault's daemon/CLI, AiOSPac's library core).

```
        AiOS agent runtime (Python, parent repo)  ·  human user
              ▲ events / status   │ commands ▼
┌──────────────────────────────────────────────────────────────┐
│  Frontends (thin; swappable)                                   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐  │
│  │ GUI (egui) │ │ CLI        │ │ MCP server │ │ Daemon /   │  │
│  │ settings + │ │            │ │ (F-MCP)    │ │ scheduler  │  │
│  │ restore    │ │            │ │            │ │ service    │  │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘  │
├────────────────────────────────────────────────────────────────┤
│  aios-backup-core (headless, no UI)                            │
│   backup set config · snapshot/index model · restore planner ·│
│   scheduler · progress + event stream · key/secret access     │
├───────────────┬────────────────┬───────────────┬──────────────┤
│ BackupEngine  │ Destination    │ ChangeSource  │ SecretStore   │
│ trait         │ trait          │ trait         │ trait         │
│  → rustic_    │  → local disk  │  → AiOSFSS    │  → AiOSVault  │
│    core       │  → rsync.net   │    (AiOS)     │    (AiOS)     │
│  (restic fmt) │    SFTP / rclone│  → notify    │  → OS keyring │
│  [restic CLI  │  → (generic)   │    (other OS) │    (other OS) │
│   fallback]   │                │               │               │
├───────────────┴────────────────┴───────────────┴──────────────┤
│  OS substrate   filesystem · OS service mgr · SSH/network      │
└────────────────────────────────────────────────────────────────┘
```

- **`aios-backup-core`** — a headless library: it owns the backup-set
  configuration (what to back up, where, on what schedule), the snapshot/index
  query model, the restore planner (full + selective), the scheduler, key/secret
  access, and a structured progress + event stream. No UI, no windowing — fully
  testable headless. It is the single brain that the GUI, CLI, MCP server, and
  daemon all drive.
- **`BackupEngine` trait → `rustic_core`** — the actual snapshot/chunk/encrypt/
  restore engine. AiOSBackup wraps `rustic_core` (restic repository format)
  behind a thin trait so the core is not coupled to one library's surface. A
  **fallback** implementation that shells out to the upstream `restic` binary
  (identical on-disk format) is kept available while `rustic_core` matures toward
  1.0. The engine does full + incremental snapshots, content-defined dedup,
  client-side encryption, restore, and prune.
- **`Destination` trait** — where a repository lives: a **local-disk** provider
  (and LAN paths), an **rsync.net** provider over **SFTP** (Unix) or **rclone**
  (Windows, and the append-only/immutable tier), and a generic provider for other
  rclone/SFTP/S3-style targets later. New destinations are added without touching
  the core.
- **`ChangeSource` trait** — how the tool learns what changed for continuous /
  triggered backups: on AiOS it subscribes to the **AiOSFSS** event stream; on
  other OSes it uses **`notify`** + **`notify-debouncer-full`**. Both feed the
  same "files changed → queue an incremental" path, so the engine is OS-agnostic.
- **`SecretStore` trait** — custody of the repository password and the SSH
  private key: **AiOSVault** on AiOS, the **OS keyring** elsewhere. Secrets are
  loaded transiently and zeroized.
- **Frontends** — the **GUI** (egui) for settings/config and restore is the
  primary human surface; the **CLI** is the scriptable/dev surface; the **MCP
  server** exposes the core to the agent (F-MCP); the **daemon/scheduler** is the
  long-lived service that runs scheduled and watch-triggered backups. All four
  are thin clients of `aios-backup-core`.

## The engine & repository format

**Decision (locked): build on `rustic_core`; the repository uses the restic
format and restic-native crypto.** `rustic_core` (Apache-2.0 OR MIT) is a
pure-Rust backup *library* — not just the `rustic` CLI — implementing the restic
repository format: content-defined chunking for **deduplication**, **incremental**
snapshots by content-addressing (an incremental simply doesn't re-upload chunks
the repository already has), **client-side encryption** of every chunk and the
index, and **full + selective restore**. rsync.net's free daily **ZFS snapshots**
add server-side point-in-time versioning on top.

**The crypto exception (deliberate, confirmed with Jason).** Adopting the restic
format *means inheriting restic's cryptography* — scrypt-derived keys, AES-256
in CTR mode with Poly1305-AES authentication — **not** AiOS's `aios-crypto`
envelope (Argon2id + XChaCha20-Poly1305). This is the **one documented exception**
to the AiOS "encrypt everything with `aios-crypto`" default, and it is a
considered trade:

- **Why accept it:** the payoff is **data-loss resilience through interoperability**.
  The repository is readable and restorable by the mature, widely-deployed
  upstream `restic` and `rustic` tools — so if AiOSBackup is ever broken,
  unavailable, or abandoned, the user's backups are not hostage to it. For a
  data-loss-sensitive tool that is worth more than crypto-stack uniformity. The
  restic format and its crypto are themselves well-analyzed and battle-tested.
- **Where `aios-crypto` still applies:** it seals the **local** repository
  password and the SSH private key at rest (via AiOSVault on AiOS), so AiOS-side
  secret custody stays on the AiOS crypto stack even though the repository
  payload is restic-native.
- **The boundary is explicit and documented** so it is never mistaken for an
  oversight: repository-at-rest/on-wire crypto = restic-native; local secret
  custody = `aios-crypto`/keyring.

**Engine maturity & fallback.** `rustic_core` still self-describes as beta. The
mitigations: wrap it behind the `BackupEngine` trait; pin versions; treat
**restore-integrity regression tests** as a release gate; rely on restic-format
interop as the ultimate safety net; and keep a `restic`-binary `BackupEngine`
implementation available as a drop-in (same format) if a target OS or version
proves unstable. Track `rustic_core`'s progress toward 1.0.

## Encryption & key management

- **Repository encryption:** the user sets a password; `rustic_core` derives the
  repository master key from it (scrypt) and encrypts all chunks + index
  client-side. The remote sees only ciphertext.
- **Password custody:** the password is sealed — AiOSVault on AiOS, OS keyring
  elsewhere (`SecretStore` trait) — so unattended/scheduled backups can run
  without prompting, while the password is never written in plaintext. A
  user-facing path to enter/rotate it lives in the GUI and CLI.
- **The forgotten-password reality:** client-side encryption means a lost
  password = unrecoverable backups. The GUI/CLI must surface this plainly at
  setup (and support an explicit, user-chosen recovery-key export if we offer
  one — an open question).
- **SSH keypair for automated backups:** AiOSBackup generates an **ed25519**
  keypair with `ssh-key` (RustCrypto, pure-Rust), writes the OpenSSH-format
  private key into the `SecretStore` (sealed), and presents the public key as the
  exact one-line `ssh-ed25519 …` string for the user to install into rsync.net's
  `authorized_keys` (the GUI walks the `scp …/.ssh/authorized_keys` /
  append-via-`dd` workflow rsync.net documents). Passwordless keys are required
  for unattended runs; sealing them in the vault/keyring keeps that safe.

## Cross-platform strategy

All four targets — **AiOS, Linux, Windows, macOS** — are first-class in v1
(Jason's call, 2026-05-31). The OS-specific surfaces are isolated behind the
traits above:

| Concern | AiOS | Linux | Windows | macOS |
|---|---|---|---|---|
| Change detection (`ChangeSource`) | AiOSFSS event stream | `notify` (inotify) | `notify` (ReadDirectoryChangesW) | `notify` (FSEvents) |
| Key custody (`SecretStore`) | AiOSVault | Secret Service / keyring | Credential Manager | Keychain |
| rsync.net transport (`Destination`) | SFTP | SFTP | **rclone** (OpenDAL-SFTP is Unix-only) | SFTP |
| Run-at-boot | agent-supervised service | systemd unit | Windows Service | launchd `LaunchDaemon` |

The **OpenDAL SFTP backend is Unix-only**, so the **Windows** client reaches
rsync.net via the **rclone** backend; rclone also backs the append-only/immutable
tier on every OS. The OS-native "service" in each case does nothing but relaunch
the long-lived AiOSBackup daemon at boot — all scheduling stays in-process (see
below), so behavior is uniform across OSes.

## Scheduling & file watching

- **Scheduler (`B-SCHED`):** one in-process **`tokio-cron-scheduler`** inside the
  long-lived daemon, owning its own schedule + last-run state in a small local
  store, with a **catch-up-on-startup** check so a backup missed during downtime
  fires on next launch. Scheduled full/incremental runs, ad-hoc runs, and
  watcher-triggered incrementals are all jobs on one runtime with one
  concurrency/locking model — uniform across every OS. This beats maintaining
  separate systemd-timer / launchd / Task-Scheduler unit generators.
- **Watching (`B-WATCH`):** the `ChangeSource` trait (AiOSFSS on AiOS, `notify` +
  `notify-debouncer-full` elsewhere) collapses the burst of events a single save
  emits into one change batch and queues an incremental. Continuous backup is
  thus opt-in on top of the schedule.

## Restore

- **Full and selective restore (`B-RESTORE`):** restore an entire snapshot or a
  chosen subtree/files to the original location or an alternate path; the GUI
  presents the snapshot's file tree (`CollapsingHeader`/tree view) for selection
  and a progress bar from the core's event stream.
- **Integrity:** restores verify chunk integrity (the format authenticates every
  chunk); a restore-verification mode re-reads and checks restored data.
- **Safety:** restore is **destructive** (it can overwrite). The GUI confirms
  overwrites; **agent-initiated restore is mediated/confirmed** (never
  autonomous); restored paths are validated against traversal and treated as
  untrusted (a tampered remote repository must not be able to write outside the
  chosen restore root).

## Agent integration (F-MCP)

Per the AiOS-wide **F-MCP** requirement (parent `DECISIONS.md`, 2026-05-23),
`aios-backup-core` is fronted by an **MCP server** consumed by the agent (and,
via the agent, by other components). The surface, sketched:

- **Read/observe (freely available):** list backup sets, list snapshots, query
  last-run status and schedule, query repository health, subscribe to progress/
  completion/error events.
- **Backup (low-risk, may be agent-initiated under policy):** trigger an ad-hoc
  full or incremental backup of a configured set.
- **Restore + destructive config (mediated only):** restore (overwrites data),
  delete/prune snapshots, change destinations or the password — these are
  **mediated, confirmed** actions per F-PROMPT-SAFETY / F-AGENT-POLICY, with
  provenance, never autonomous. The agent never receives the repository password.

When `aios-mcp` (the shared F-MCP framework, `aios-libs` Phase B, Sprint 8) is
available, AiOSBackup adopts it as the one chokepoint for allowlists, provenance,
and audit; until then it exposes an equivalent surface directly.

## Security considerations (F-PROMPT-SAFETY)

- **Confidentiality is the product.** Client-side encryption before egress means
  a compromised or hostile remote (or a stolen backup) yields only ciphertext.
  Repository crypto is restic-native and well-analyzed; local secret custody is
  `aios-crypto`/keyring-sealed.
- **Restore inputs are untrusted.** Snapshot metadata, stored paths, and file
  contents from a repository (potentially tampered at the remote) are treated as
  hostile on restore: path-traversal-safe extraction confined to the restore
  root, size/count sanity limits, never interpreted as instructions.
- **Destructive operations are gated.** Restore, prune, destination/password
  changes require explicit confirmation; agent-initiated forms are mediated.
- **Keys never leak through the agent or the GUI.** The repo password and SSH key
  live sealed; loaded transiently, zeroized; the agent surface never returns
  them.
- **Network egress is constrained.** Backups talk only to the configured
  destination endpoint(s); novel destinations follow the parent's egress posture.
- **Supply chain.** All deps permissive-licensed; `cargo audit` / `cargo deny` in
  CI like every Rust component; SHA-pinned GitHub Actions per the Sprint-3 policy;
  the eventual `aios-packages` release tag is SSH-signed per the Sprint-5 signing
  bootstrap.
- **Stack choice serves safety.** Rust core keeps the config/scheduling/restore
  logic memory-safe; the engine is a vetted Rust library (`rustic_core`), not a
  bespoke crypto implementation.

## Key Decisions

### Inherited from the AiOS project (confirmed — see parent `DECISIONS.md`)

- **Apache-2.0** across all code; permissive deps only.
- **Rust** for system/security-critical components.
- **Headless core, thin frontends.**
- **F-PROMPT-SAFETY**: untrusted inputs are data; destructive agent actions mediated.
- **Secrets through AiOSVault** (F-VAULT) on AiOS, mediated where possible.
- **F-MCP**: every component exposes an MCP tool surface for the agent.
- **Multi-user-ready** from the start; AiOS single-user through M2.

### AiOSBackup-specific — locked with Jason (2026-05-31)

- **Engine: `rustic_core` behind a `BackupEngine` trait**, restic repository
  format, with a `restic`-binary fallback. *Rationale:* don't reinvent a
  security-critical engine; gain restic/rustic interop as a restore safety net;
  isolate the beta-library risk behind a trait + restore-integrity tests.
- **Repository crypto: restic-native (scrypt + AES-256-CTR/Poly1305-AES)**, the
  **one documented exception** to the `aios-crypto`-everywhere default; chosen for
  interoperable, recoverable backups. `aios-crypto`/keyring still seals the local
  password + SSH key.
- **GUI: egui/eframe** — wgpu+winit, native Wayland (AiOSCanvas client), AccessKit
  accessibility, same GPU stack as Canvas/Terminal, dual MIT/Apache. *Runner-up:*
  iced, if a retained/reactive architecture is ever needed (its accessibility is
  still incomplete today).
- **Remote transport: rsync.net over SFTP (Unix) + rclone (Windows & append-only
  tier).** Local disk is the no-network baseline. Destinations are pluggable
  behind a `Destination` trait (local + rsync.net are the v1 implementations).
- **SSH keys: `ssh-key` (RustCrypto) ed25519**; private key sealed in the
  `SecretStore` (AiOSVault / OS keyring), never plaintext.
- **Change detection: a `ChangeSource` trait** — AiOSFSS on AiOS, `notify` +
  `notify-debouncer-full` elsewhere (don't double-watch on AiOS).
- **Scheduling: in-process `tokio-cron-scheduler`** in a long-lived daemon,
  catch-up-on-startup; OS-native service only relaunches the daemon.
- **Scope: all desktop OSes (AiOS + Linux + Windows + macOS) first-class in v1.**
- **Layout: a Cargo workspace** (`aios-backup-core` lib + `aios-backup` CLI/daemon
  + `aios-backup-gui` + `aios-backup-mcp`), edition 2024, toolchain pinned to the
  sibling repos. Final shape confirmed in Phase 0.

## Repository layout (planned)

```
AiOSBackup/
├─ Cargo.toml              # cargo workspace
├─ Cargo.lock              # committed
├─ rust-toolchain.toml     # channel pinned to the sibling repos
├─ Makefile                # build / release / test / lint / fmt / clean
├─ LICENSE                 # Apache-2.0 (present)
├─ NOTICE
├─ README.md               # present
├─ CLAUDE.md               # present — agent guidance, links this doc set
├─ PLAN.md ROADMAP.md      # this document set
├─ FEATURES.md             # the B-* catalog (Phase 0)
├─ DESIGN.md               # engine wrapping, traits, repo/secret layout (Phase 0)
├─ aios-backup.example.toml# backup sets, destinations, schedule, options
├─ .github/workflows/      # CI: fmt, clippy, test, cargo-audit, cargo-deny, actions-pinned
├─ crates/
│  ├─ aios-backup-core/    # headless: config, engine/destination/watch/secret traits,
│  │                       #   rustic_core engine impl, scheduler, restore planner, events
│  ├─ aios-backup/         # binary — CLI + long-lived daemon/scheduler service
│  ├─ aios-backup-gui/     # egui settings + restore frontend
│  └─ aios-backup-mcp/     # MCP server fronting the core (adopts aios-mcp when available)
└─ tests/                  # integration tests incl. backup→restore round-trips + fixtures
```

`FEATURES.md` (`B-*`) and `DESIGN.md` are written at Phase 0; this PR delivers
`PLAN.md` and `ROADMAP.md`.

## Proposed dependencies

- **Engine:** `rustic_core` + `rustic_backend` (restic format; local/SFTP/rclone
  backends). `restic` binary available as a fallback engine.
- **House style (shared with sibling repos):** `anyhow`, `thiserror`, `clap`
  (derive), `serde`/`serde_json`, `toml`, `tracing`/`tracing-subscriber`.
- **Async / daemon:** `tokio`; **scheduler:** `tokio-cron-scheduler`.
- **Watching:** `notify` + `notify-debouncer-full` (non-AiOS); AiOSFSS client for
  the AiOS `ChangeSource`.
- **SSH keys:** `ssh-key` (RustCrypto, `ed25519` feature).
- **Secrets:** the AiOSVault client (AiOS) and an OS-keyring crate (e.g.
  `keyring`) elsewhere; `zeroize`/`secrecy` for transient key material; the
  internal `aios-crypto` for sealing where used.
- **GUI:** `eframe`/`egui` (+ `egui` AccessKit feature), `rfd` for native
  file/folder pickers.
- **Dev:** `tempfile`, sample data fixtures; a throwaway local SFTP/rclone target
  for transport tests.

All deps are Apache-2.0/MIT/BSD — no GPL/AGPL linked libraries. (The optional
`restic`/`rclone` binaries are separate processes, not linked libraries.)

## Phases & milestones

Phase and milestone planning lives in [ROADMAP.md](ROADMAP.md). In short: Phase 0
scaffolds the workspace and runs the engine spike (backup→restore round-trip to a
local repo and to rsync.net over SFTP); the first usable deliverable is a
**headless engine + CLI making and restoring encrypted incremental backups to
local and rsync.net on all four OSes**; then the **scheduler + daemon + service**,
the **GUI**, the **agent/MCP surface + AiOSFSS/AiOSVault integration**, and the
**append-only/immutable tier + hardening**. Phases track the AiOS milestones **M1
(Q3 2026), M2 (Q4 2027), M3 (Q4 2028)** — AiOSBackup does not set its own.

## Risks

- **Restore correctness is existential.** A backup that won't restore is worse
  than none. Mitigation: restore-integrity round-trip tests as a release gate;
  restic-format interop as a last-resort recovery path; restore-verification mode.
- **`rustic_core` is beta.** API churn and undiscovered bugs in an early library
  holding real data. Mitigation: trait isolation, version pinning, regression
  tests, `restic`-binary fallback, track 1.0.
- **Cross-OS surface multiplies the work.** Four OSes × (watch, keyring, service,
  transport) is real breadth for a solo dev. Mitigation: the traits keep the core
  shared; per-OS code is thin and isolated; narrow scope before slipping dates.
- **Forgotten password = unrecoverable data.** A support and UX hazard. Mitigation:
  unmistakable setup warnings; optional explicit recovery-key export (open
  question); never silently store the password unsealed.
- **rsync.net / transport specifics drift.** Provider-side details (paths, rclone
  config, append-only setup) can change. Mitigation: isolate behind `Destination`;
  integration-test against a local SFTP/rclone stand-in.
- **VMs can't fully validate.** The dev VMs lack real multi-OS coverage; Windows/
  macOS paths need validation on real machines before milestone sign-off.

## Open Questions

Decisions that need Jason's (or a sibling repo's) input. The big technology forks
(engine, repo crypto, GUI, OS scope) are **resolved** — see Key Decisions.

1. **Default backup set on AiOS.** What does AiOSBackup back up by default on AiOS
   — the user's data dirs, the AiOSVault file, canvas state, Org/Comms data? This
   needs the canonical AiOS data layout (coordinate with AiOSFSS's
   `AIOS_DATA_DIR` and the other components) and is deferrable until that firms up.
2. **Secret custody on non-AiOS clients.** Confirm the OS-keyring approach
   (Windows Credential Manager / macOS Keychain / Secret Service via the `keyring`
   crate) for the repo password + SSH key, with an `aios-crypto`-sealed local file
   as the fallback when no keyring is available (headless servers). Recommended:
   keyring-first, sealed-file fallback.
3. **GUI on AiOS: standalone Wayland client vs. canvas widget.** v1 runs the egui
   GUI as a standalone Wayland client on AiOSCanvas. Should settings/restore
   *also* become an in-canvas widget surface later (a new `C-*` capability
   dependency)? Recommended: standalone client for v1; revisit a canvas-widget
   presentation post-M2.
4. **Append-only / immutable remote tier.** Offer the rclone
   `serve restic --append-only` workflow against rsync.net's ZFS snapshots as a
   ransomware-resistant option from the start (recommended: yes, as a per-
   destination toggle), or defer to hardening?
5. **Recovery-key export.** Given client-side encryption, do we offer an explicit,
   user-initiated **recovery key** (a separately-stored key that can also unlock
   the repo) to soften the forgotten-password cliff, or keep strictly
   password-only? Recommended: offer an opt-in recovery-key export, clearly
   user-owned.
6. **Component wiring timing.** Submodule + parent CI + `aios-packages` entry land
   at AiOSBackup's **first release** (matching the released-components pattern),
   not at plan time. Confirm.

## Change Log

| Date | Change | Rationale |
|------|--------|-----------|
| 2026-05-31 | Initial AiOSBackup planning document set created (PLAN, ROADMAP, CLAUDE, README) | Repository kickoff; derived from the parent AiOS docs + Jason's feature brief, after three research streams (engine, transport/keys/watch/schedule, GUI) |
| 2026-05-31 | Locked: `rustic_core` engine (restic format) + **restic-native repo crypto** as the one documented `aios-crypto` exception; **egui** GUI; rsync.net via **SFTP + rclone**; `ssh-key` keygen sealed in Vault/keyring; `ChangeSource`/`Destination`/`SecretStore`/`BackupEngine` traits; in-process `tokio-cron-scheduler`; **all desktop OSes first-class in v1** | Jason's decisions on the three pivotal forks (2026-05-31); engine/GUI/transport recommendations adopted from the research |
