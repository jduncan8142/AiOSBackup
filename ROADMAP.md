# AiOSBackup — Roadmap

The time-ordered delivery plan for the AiOS backup utility.

For vision, architecture, the engine/crypto choice, and the technology
decisions, see [PLAN.md](PLAN.md). The feature catalog referenced by tag
(`B-*`) is written in `FEATURES.md` at Phase 0; concrete tasks live in `TODO.md`.

## How this roadmap relates to AiOS

AiOSBackup is one component of the AiOS project, and its schedule is
**subordinate to the parent AiOS milestones** — M1 (booting prototype, Q3 2026),
M2 (daily driver, Q4 2027), M3 (alpha, Q4 2028). AiOSBackup does not set its own
milestones.

AiOSBackup is **not on the critical path for AiOS M1**: M1 is a booting
compositor with one interactive widget and no AI — a backup utility is not that.
So AiOSBackup is built in the time around the compositor and agent work. Because
it is **cross-OS from v1**, much of it (engine, CLI, scheduler, GUI) is fully
developable on a plain Linux/Windows/macOS dev machine *without* the rest of AiOS;
only the AiOS-specific integration (AiOSFSS change source, AiOSVault key custody,
agent supervision, any canvas presentation) is gated on sibling components.

Dates are deliberately approximate and inherit the parent's solo-developer
posture: **when a date is at risk, narrow the phase's scope rather than slip the
date.** The standing discipline: **restore correctness is the floor and must be
excellent before any feature breadth** (PLAN principle 2).

## Milestone overview

| Milestone | Date | Phases | What AiOSBackup delivers |
|-----------|------|--------|--------------------------|
| **M1 — Booting prototype** | Q3 2026 | P0 (partial) | Not on the M1 critical path. At most: project scaffold + CI, and the engine spike — a `rustic_core` encrypted backup→restore round-trip to a local repo and to rsync.net over SFTP, driven by an early CLI on the dev machine. |
| **M2 — Daily driver** | Q4 2027 | P1–P4 | Encrypted full+incremental backup & restore to local disk and rsync.net across **AiOS + Linux + Windows + macOS** (CLI); a long-lived daemon with in-process scheduling + file-watching and per-OS run-at-boot; the egui settings+restore GUI; and the agent **MCP** surface with AiOS integration (AiOSVault key custody, AiOSFSS change source, agent supervision, mediated restore). |
| **M3 — Alpha release** | Q4 2028 | P5 | Hardening: the append-only/immutable ransomware-resistant remote tier, performance, accessibility, full restore-assurance + external review of the untrusted-restore and crypto-boundary handling. |

## P0 — Foundations & Engine Spike · *heavy*

**Goal:** turn the bare repository into a buildable Rust workspace and de-risk the
single most important unknown — that the engine makes a backup that actually
restores, encrypted, to a real remote — before building anything on it.

- Scaffold the Cargo workspace (`aios-backup-core`, `aios-backup`,
  `aios-backup-gui`, `aios-backup-mcp`), `rust-toolchain.toml` (pinned to the
  sibling repos), `Makefile`, `NOTICE`, `.gitignore`, `aios-backup.example.toml`,
  and CI (GitHub Actions: `fmt`, `clippy --all-targets -- -D warnings`, `test`,
  `cargo audit`, `cargo deny`, the SHA-pinned-actions guard).
- **Engine spike** — drive `rustic_core` to initialize an encrypted repository,
  back up a sample tree, take an incremental, and **restore it with an integrity
  check**, against (a) a local repo and (b) **rsync.net over SFTP**. Confirm the
  `restic`-binary fallback reads the same repo. Confirm the **rclone** backend
  path for Windows.
- Resolve the Phase-0 items in [PLAN.md](PLAN.md): final crate layout, the
  `SecretStore` shape per OS, and the `Destination` config schema.
- Write `FEATURES.md` (the `B-*` catalog) and `DESIGN.md` (engine wrapping, the
  four traits, repository + secret layout).

**Exit:** `make build` / `test` / `lint` succeed; an encrypted backup taken with
`rustic_core` restores correctly from both a local repo and rsync.net; the design
is written down.

## P1 — Headless Engine + CLI · *heavy*

**Goal:** `aios-backup-core` + a CLI make and restore encrypted, incremental
backups to local and rsync.net on **all four OSes**.

- `B-ENGINE` — the `BackupEngine` trait over `rustic_core`: init, backup (full +
  incremental), list snapshots, restore, prune; the `restic`-binary fallback impl.
- `B-CRYPTO` — repository password handling; key derivation via the engine
  (restic-native); transient, zeroized password material.
- `B-REMOTE` — the `Destination` trait with **local-disk** and **rsync.net**
  providers (**SFTP** on Unix, **rclone** on Windows); the no-network local
  baseline first.
- `B-KEYS` — ed25519 keypair generation (`ssh-key`); emit the paste-ready public
  key + walk the rsync.net `authorized_keys` install workflow.
- `B-RESTORE` (core) — full + selective restore to original/alternate paths;
  restore-integrity verification; path-traversal-safe extraction.
- **Restore-integrity regression tests** established as a release gate.

**Exit:** the CLI takes encrypted full+incremental backups of a chosen set to a
local repo and to rsync.net, and restores them verified, on Linux/Windows/macOS
(and AiOS as a Linux-class target). No daemon, GUI, or agent yet.

## P2 — Daemon, Scheduler & Watching · *moderate*

**Goal:** unattended, scheduled, and change-triggered backups via a long-lived
service on every OS.

- `B-SCHED` — the in-process `tokio-cron-scheduler` in a long-lived daemon;
  scheduled full/incremental + ad-hoc jobs; persisted schedule + last-run state;
  **catch-up-on-startup** for missed runs.
- `B-WATCH` — the `ChangeSource` trait with the `notify` + `notify-debouncer-full`
  implementation (non-AiOS); change batches queue incrementals (continuous backup,
  opt-in).
- `B-XPLAT` — per-OS run-at-boot: systemd unit (Linux), Windows Service, launchd
  `LaunchDaemon` (macOS) — each only relaunches the daemon; `SecretStore` keyring
  impls per OS (Credential Manager / Keychain / Secret Service).

**Exit:** scheduled and watch-triggered encrypted backups run unattended after a
reboot on Linux, Windows, and macOS, with secrets in the OS keyring.

## P3 — Settings & Restore GUI · *moderate*

**Goal:** a desktop GUI for configuration and restore, on every desktop OS and as
a Wayland client on AiOS.

- `B-GUI` — the egui/eframe frontend: backup-set + destination + schedule config
  forms; the SSH-key setup wizard (generate → show public key → rsync.net
  install steps); a **snapshot browser + restore file-tree** with selection; live
  progress bars and event log from the core's stream; password entry/rotation
  with the forgotten-password warning.
- Native file/folder pickers (`rfd`); AccessKit accessibility pass (opt-in, an
  AiOS goal); runs as a standalone Wayland client on AiOSCanvas.

**Exit:** a user configures backups, sets up the rsync.net key, runs an ad-hoc
backup, and performs a selective restore entirely from the GUI on all desktop
OSes.

## P4 — Agent / MCP & AiOS Integration · *moderate* · **→ M2**

**Goal:** the AiOS milestone M2 — AiOSBackup is agent-operable and natively
integrated on AiOS.

- `B-AGENT` — the **MCP server** (parent F-MCP): read/observe + backup tools
  freely available; **restore, prune, and destructive config mediated/confirmed**
  (never autonomous), with provenance; the agent never receives the password.
  Adopt `aios-mcp` (`aios-libs` Phase B, Sprint 8) as the chokepoint when ready.
- **AiOSVault** `SecretStore` impl — repo password + SSH key sealed in and
  released by the vault on AiOS (mediated where its model allows).
- **AiOSFSS** `ChangeSource` impl — subscribe to its change-event stream on AiOS
  instead of running a second watcher.
- **Agent supervision** — AiOSBackup runs as an agent-supervised service on AiOS,
  the way AiOSFSS does.

**Exit (= AiOSBackup's M2 contribution):** with the agent runtime, AiOSVault, and
AiOSFSS, the agent can drive backups and request (mediated) restores over MCP; on
AiOS, keys live in the vault and change detection comes from AiOSFSS.

## P5 — Hardening, Immutable Tier & Restore Assurance · *moderate* · **→ M3**

**Goal:** alpha-quality — the parent milestone M3.

- **Append-only / immutable tier** — the rclone `serve restic --append-only`
  workflow against rsync.net's ZFS snapshots as a per-destination,
  ransomware-resistant option.
- `B-SAFETY` (complete) — full review of the untrusted-restore surface
  (path-traversal, tampered-repository handling), egress constraints, and the
  documented restic-vs-`aios-crypto` crypto boundary; external review.
- `B-PERF` — backup/restore throughput and memory behaviour on real data sizes;
  dedup/prune tuning.
- Accessibility passes; validation on real Windows/macOS hardware (the dev VMs
  can't fully cover multi-OS — see PLAN Risks); track `rustic_core` to 1.0 and
  re-evaluate dropping the `restic`-binary fallback.

## Cross-component dependencies

- **AiOSBackup is not on the M1 critical path** — it is built around the
  compositor and agent work; P0–P3 need no other AiOS component.
- **P4 depends on the parent agent runtime** (MCP consumption + supervision),
  **AiOSVault** (its daemon/control plane — the V2 work, Sprint 8) for key
  custody on AiOS, and **AiOSFSS** (its change-event stream) for the AiOS
  `ChangeSource`.
- **`aios-mcp`** (`aios-libs` Phase B, Sprint 8) is adopted as the F-MCP
  chokepoint when available; until then AiOSBackup exposes the surface directly.
- **No dependency on AiOSCanvas for v1** — the GUI runs as a standalone Wayland
  client; an in-canvas widget presentation is a possible later `C-*` dependency
  (PLAN Open Question 3).
- **Complementary to AiOSFSS, not coupled as transport** — FSS is LAN
  device-to-device sync; AiOSBackup is point-in-time encrypted snapshots to
  local/remote. AiOSBackup consumes FSS *events*, not its sync transport.

## Change Log

| Date | Change | Rationale |
|------|--------|-----------|
| 2026-05-31 | Initial roadmap created | Repository kickoff; phases aligned to the parent AiOS milestones M1/M2/M3, with the headless engine + restore correctness first and the GUI/agent/AiOS integration layered behind it |
| 2026-05-31 | Set **all desktop OSes first-class in v1**; pulled the rclone backend + per-OS services into P1–P2 (not deferred) | Jason's OS-scope decision (2026-05-31) — cross-OS is the point of the tool |
