## Why

OmniRoute's SQLite database (`/var/lib/omniroute/storage.sqlite`) has no off-host backup. The only existing backup is a manual, pre-upgrade tar that stops the service — there is no automated way to recover from VPS loss. The database holds API keys, JWT hashes, providers, and settings; losing it means reconfiguring the entire proxy.

## What Changes

- Add a daily scheduled backup worker that copies OmniRoute's **newest consistent snapshot** (`/var/lib/omniroute/db_backups/db_*.sqlite`, produced by the app's built-in WAL-safe backup) to a **GitHub private repository**.
- The snapshot is written to a **single fixed filename** at the repo root (`omniroute-backup.sqlite`), overwritten on each run — no `snapshots/` folder, no history of intermediate files in the working tree.
- Backup history is bounded with **keep-last-N + periodic history rewrite**: each run commits the current snapshot; a monthly job rewrites history to a single orphan commit and force-pushes, so repo size stays flat.
- Auth uses a **read/write SSH deploy key** added to the private repo; the private key is vaulted and deployed by Ansible. No client-side encryption of the backup file (private repo is trusted).
- Runs as a **systemd timer** (daily, `Persistent=true` to catch up after downtime), mirroring the existing `certbot.timer` precedent. No service downtime — the app's online backup API is used, never a stop/start.
- Failure visibility: logs to journald; no external alerting in v1.

## Capabilities

### New Capabilities
- `omniroute-backup`: Daily off-host backup of OmniRoute's SQLite database to a GitHub private repository — snapshot selection, single-file publication, bounded history, deploy-key auth, and timer-driven scheduling.

### Modified Capabilities
<!-- No existing specs; greenfield. -->

## Impact

- **Ansible role**: extend `roles/omniroute` (or add a sibling `backup` role) with a timer unit, a backup script, and vaulted deploy-key delivery. The existing manual `tasks/backup.yml` (stop-service tar) remains untouched — it is for pre-upgrade archives, not daily backup.
- **Vault**: new keys for the GitHub repo slug, deploy-key private key, and target branch.
- **Host packages**: none required — `git` is already installed by the `base` role; the snapshot source already exists.
- **GitHub**: a new private repository with the deploy key attached (manual one-time setup, documented in the change).
- **Runtime behavior**: no change to the OmniRoute service, supervisor config, or firewall. The built-in auto-backup stays enabled (local safety net, keep-last-5).
