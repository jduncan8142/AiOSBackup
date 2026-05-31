# CLAUDE.md — AiOSBackup

**Status: planning / not started.** No code exists yet; this repo holds the
license and the planning document set. Read the docs before writing any code.

## What this is

AiOSBackup is a **Rust backup utility** for **AiOS** — an AI-first, Linux-based
OS — that **also runs on Windows, macOS, and Linux clients**. Parent project:
https://github.com/jduncan8142/AiOS . Unlike the canvas widgets (AiOSMedia,
AiOSCalendar, …), AiOSBackup is a **cross-OS system component** — a headless
daemon plus a desktop GUI, in the same family as AiOSVault and AiOSPac, not a
canvas-hosted widget.

It makes **encrypted, deduplicated, point-in-time backups** and restores them:
- **Client-side encryption with the user's password before data leaves the machine.**
- **Full and incremental** backups, **ad-hoc and scheduled**.
- A desktop **GUI** for settings/configuration and for restore.
- All functions exposed to the AiOS agent as **MCP tooling** (parent **F-MCP**).
- **Remote backups to [rsync.net](https://www.rsync.net)** (and local targets),
  with an **SSH keypair generated** for unattended/automated runs.
- On AiOS, change detection via **AiOSFSS**; on other OSes, its own file watcher.

## Locked decisions (confirmed with Jason — see [PLAN.md](PLAN.md) "Key Decisions")

- **Engine: build on `rustic_core`** — the pure-Rust, restic-repo-compatible
  backup library — behind a thin `BackupEngine` trait. The repository uses
  **restic's vetted format and crypto** (so mature `restic`/`rustic` tooling can
  restore it if AiOSBackup is ever unavailable). This is the **one documented
  exception** to the AiOS "encrypt everything with `aios-crypto`" default;
  `aios-crypto`/the OS keyring still seals the *local* repo password and SSH key.
- **GUI: egui/eframe** — wgpu + winit, native Wayland (runs as a client on
  AiOSCanvas), AccessKit accessibility; same GPU stack as AiOSCanvas/AiOSTerminal.
- **Remote: rsync.net over SFTP** (Unix) and **rclone** (Windows, and the
  append-only/immutable tier against rsync.net's ZFS snapshots).
- **SSH keys: `ssh-key` (RustCrypto)** ed25519 keygen; private key sealed via
  AiOSVault/`aios-crypto` (OS keyring on non-AiOS), never plaintext.
- **Watch: a `ChangeSource` trait** — AiOSFSS event stream on AiOS, `notify` +
  `notify-debouncer-full` elsewhere.
- **Schedule: in-process `tokio-cron-scheduler`** in a long-lived daemon
  (catch-up on startup); the OS service only relaunches the daemon at boot.
- **Scope: all desktop OSes (AiOS + Linux + Windows + macOS) are first-class in v1.**

## Conventions inherited from AiOS

- License: **Apache-2.0** (AiOS-wide); every dependency must be permissive
  (Apache/MIT/BSD) — no GPL/AGPL linked libraries.
- Language/stack: **Rust** — AiOSBackup runs at a data-confidentiality boundary
  and handles untrusted restore inputs, so the "Rust for security-critical
  components" rule applies. **Headless core + thin frontends** (GUI / CLI / MCP /
  daemon), mirroring AiOSVault/AiOSPac/AiOSTerminal.
- Security: restore inputs and repository metadata are **untrusted data** under
  **F-PROMPT-SAFETY**; agent-initiated **restore is destructive and is mediated**
  (confirmed) under F-PROMPT-SAFETY / F-AGENT-POLICY — never a free agent action.
- Secrets: the repo password and SSH private key go through **AiOSVault** on
  AiOS (the OS keyring on other OSes), never stored in plaintext.
- Multi-user-ready from the start; AiOS is single-user through M2.

## The parent repo

The parent **AiOS** repo holds the authoritative cross-project docs
(`PLAN.md`, `DECISIONS.md`, `FEATURES.md`, `THREAT_MODEL.md`); read
`DECISIONS.md` before reopening any settled AiOS-wide choice. AiOSBackup
implements **F-BACKUP** and owns its own `B-*` sub-feature tags (written into
`FEATURES.md` at Phase 0).

## Next steps (when we start)

Confirm the PLAN open questions with Jason; run the Phase-0 engine spike
(`rustic_core` backup→restore round-trip to a local repo and to rsync.net over
SFTP); scaffold the Cargo workspace; write `FEATURES.md` (`B-*`) + `DESIGN.md`;
wire AiOSBackup in as an AiOS component (submodule + CI + `aios-packages` entry)
at its first release.
