## Context

OmniRoute 3.8.49 runs under Supervisor as the unprivileged `omniroute` user with `DATA_DIR=/var/lib/omniroute`. Its single SQLite database lives at `/var/lib/omniroute/storage.sqlite` in WAL mode. The app already maintains WAL-safe consistent snapshots at `/var/lib/omniroute/db_backups/db_<timestamp>_<reason>.sqlite` via its built-in backup (native SQLite online-backup API), pruned to keep the last 5. See `proposal.md` for motivation; requirements in `specs/omniroute-backup/spec.md`.

Existing infrastructure: git is already installed by the `base` role; the only scheduler precedent on the box is `certbot.timer` (systemd). All secrets live in `ansible/group_vars/vps/vault.yml`; the repo's safety section forbids plaintext secrets.

## Goals / Non-Goals

**Goals**
- One daily, non-disruptive off-host backup of the newest OmniRoute snapshot to a GitHub private repo.
- Repo stays small: single fixed-named file in the working tree, history collapsed periodically.
- Fully Ansible-managed except one-time GitHub repo + deploy-key registration.

**Non-Goals**
- Backing up `/var/lib/omniroute` in full (uploads, call logs, npm prefix) — the existing manual pre-upgrade `backup.yml` tar remains the tool for that.
- Multi-file history, point-in-time restore from arbitrary dates (rewrite discards old blobs by design).
- Client-side encryption of the backup file (decision: private repo is trusted).
- External alerting on failure (v1 logs to journald only).
- Restore tooling — recovery is a documented manual step (replace `storage.sqlite`, restart supervisor).

## Decisions

### D1. Snapshot source: use OmniRoute's built-in `db_backups/`, don't run our own snapshot
The app's `db.backup()` uses the native SQLite online-backup API (the same mechanism as `sqlite3 .backup`) and produces a consistent single-file snapshot with no WAL sidecars — safe to commit as-is. Alternatives rejected: `sqlite3 .backup`/`VACUUM INTO` from our own script (requires installing `sqlite3` CLI, reinvents what the app already does); plain `cp` (unsafe under WAL — torn copy / lost uncheckpointed frames); stop-service tar (the old `backup.yml` approach — unacceptable daily downtime).

Consequence: the built-in auto-backup stays **enabled** (it is the snapshot producer and a local safety net). We do not set `DISABLE_SQLITE_AUTO_BACKUP`. The worker must run at least daily so the newest snapshot is always captured before the keep-last-5 pruning rolls over.

### D2. Scheduling: systemd timer + service unit
`OnCalendar=daily`, `Persistent=true` (catches up after downtime), `RandomizedDelaySec` small jitter optional. Matches the `certbot.timer` precedent; journald gives us structured logs per the "Failure logging" requirement. Timer units can't run inline commands, so a small `oneshot` service wraps the backup script. Alternatives rejected: cron (no catch-up, no journald, root crontab less auditable).

### D3. Publication model: single fixed-named file, commit every run
Each run copies the newest `db_*.sqlite` to `<repo>/omniroute-backup.sqlite`, `git add` + `git commit` + `git push`. The working tree always holds exactly one snapshot file. `git commit --allow-empty` is avoided — a no-op run (identical file) is skipped when nothing changed.

### D4. History bounding: monthly orphan-commit rewrite + force-push
`git checkout --orphan` (or `git branch -M` of an orphan commit), commit the current snapshot, `git push --force`. Collapses history to a single commit containing only the current file; repo size stays flat regardless of how long the host runs. Runs as part of the same daily worker but only when the day-of-month matches (e.g. the 1st). This satisfies "Bounded backup history".

### D5. Auth: SSH deploy key (read/write), vaulted private key
Static private key in `vault.yml`, deployed by Ansible as root-only file (`/root/.ssh/omniroute-backup` or equivalent), host key pinned in `known_hosts` to prevent MITM. GitHub deploy key with `Allow write access` checked. Alternatives rejected: fine-grained PAT (long-lived token, manual rotation); storing the key in this repo (forbidden).

### D6. Script placement & privileges
Backup script installed at `/usr/local/libexec/omniroute-backup` (root-owned, mirroring the existing `omniroute-start` pattern). Runs as root (needs read access to `/var/lib/omniroute/db_backups/` and the git repo + SSH key). Snapshot files are readable by root regardless of `omniroute` ownership.

### D7. Repo checkout location
`/var/backups/omniroute-git` — root-only, alongside the existing `/var/backups/omniroute` archive dir. `git pull --ff-only` before publishing to keep the working tree in sync with remote state.

## Risks / Trade-offs

- **Snapshot gap if worker lags > retention window** → app prunes to keep-last-5. Mitigation: daily schedule + `Persistent=true`; worst case the newest snapshot is still present. Acceptable for a "current state" backup.
- **Force-push loses remote-only history** → by design (bounded history). Mitigation: rewrite runs monthly from the same host; remote history is always a strict subset of what the host holds that day. Restore is current-state-only — documented in the proposal.
- **Deploy key compromise** → key is rw on one private backup repo. Mitigation: key only in vault + root-only on host; rotation is a vault edit + re-run.
- **Repo becomes source of sensitive data** → DB contains API keys/JWT hashes; stored plaintext in a private repo by explicit user decision (no encryption). Mitigation: private repo + deploy key; note in README.
- **git push failure (network/auth)** → run fails, logged to journald; previous backup untouched; `Persistent` next run retries.

## Migration Plan

1. One-time manual: create GitHub private repo, generate SSH keypair, add public half as repo deploy key (rw).
2. Add vault keys: repo slug, private key, branch.
3. Extend/annotate the `omniroute` role (or sibling role) with: deploy-key file, `known_hosts` pin, backup script, systemd service + timer units.
4. Run playbook; verify first backup manually (`systemctl start omniroute-backup.service`), then check the remote repo.
5. Rollback: `systemctl disable --now omniroute-backup.timer`, remove script + key files. No service impact — OmniRoute itself is untouched.

## Open Questions

None — all forks (target, shape, filename, retention model, auth, encryption, alerting) resolved during exploration.
