# AiOSBackup

A Rust backup utility for **AiOS** — and for Windows, macOS, and Linux clients alongside it.

AiOSBackup makes encrypted, deduplicated, point-in-time backups of a machine's
data and restores them on demand. Backups are **encrypted on the local machine
with the user's password before any byte leaves it**; they support **full and
incremental** runs, **ad-hoc and scheduled**; and they can be sent to a local
disk or to a remote target such as [rsync.net](https://www.rsync.net). On AiOS
it integrates with [AiOSFSS](https://github.com/jduncan8142/AiOSFSS) for change
detection and is supervised by the AiOS agent; on other operating systems it
watches the filesystem itself and runs as an OS service. Every function is also
exposed to the AiOS agent as **MCP tooling**, and a desktop **GUI** handles
settings and restore.

> **Status: planning / not started.** This repository currently holds the
> license, this README, and the planning document set below — there is no code
> yet. Read the plan before writing any.

## Planning documents

- **[PLAN.md](PLAN.md)** — vision, principles, architecture (headless core +
  GUI / CLI / MCP frontends), the engine + repository-format choice, encryption
  & key management, cross-platform strategy, security posture, key decisions,
  and open questions.
- **[ROADMAP.md](ROADMAP.md)** — phases mapped to the AiOS milestones M1 / M2 /
  M3, and the cross-OS delivery order.
- **[CLAUDE.md](CLAUDE.md)** — guidance for AI agents working in this repo.
- `FEATURES.md` (the `B-*` catalog) and `DESIGN.md` are written at Phase 0.

The parent **[AiOS](https://github.com/jduncan8142/AiOS)** repository holds the
authoritative cross-project docs (`PLAN.md`, `DECISIONS.md`, `FEATURES.md`,
`THREAT_MODEL.md`). AiOSBackup implements the parent feature **F-BACKUP**.

## License

Apache-2.0 (AiOS-wide). See [LICENSE](LICENSE).
