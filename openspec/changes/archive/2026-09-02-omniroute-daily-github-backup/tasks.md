## 1. Repository & Vault Setup

- [x] 1.1 Add vault keys: `vault_backup_repo` (GitHub `owner/repo` slug), `vault_backup_deploy_key` (private half of the deploy keypair), `vault_backup_branch` (default `main`) to `ansible/group_vars/vps/vault.yml.example` with placeholder values and a README vault-table entry
- [x] 1.2 Add non-secret vars to `ansible/group_vars/vps/vars.yml`: `backup_repo_dir` (`/var/backups/omniroute-git`), `backup_script_path` (`/usr/local/libexec/omniroute-backup`), `backup_known_hosts_file` (`/etc/ssh/ssh_known_hosts` or per-user), `backup_snapshot_source` (`/var/lib/omniroute/db_backups`), `backup_target_file` (`omniroute-backup.sqlite`), `backup_schedule` (`*-*-* 03:00:00`), `backup_rewrite_day` (`1`)
- [x] 1.3 Update `ansible/site.yml` pre-task assert block to validate `vault_backup_repo` and `vault_backup_deploy_key` are set (non-placeholder)

## 2. Deploy Key Provisioning

- [x] 2.1 Add tasks to the `omniroute` role (or new `backup` role) that install the deploy key at root-only path (mode `0600`, owner root), skipping in check mode
- [x] 2.2 Add a task that pins the GitHub host key into the known-hosts file (via `known_hosts` module or templated file) so SSH never prompts on first connect
- [x] 2.3 Add a task that renders an SSH config drop-in (or `GIT_SSH_COMMAND` in the service unit) pointing git at the deploy key — avoid `core.sshCommand` ambiguity by making it explicit per-invocation

## 3. Backup Script

- [x] 3.1 Write `/usr/local/libexec/omniroute-backup` (bash, `set -eu`) template: select newest `db_*.sqlite` from `backup_snapshot_source`; exit non-zero with error if none found
- [x] 3.2 Script step: ensure repo clone exists at `backup_repo_dir` (clone if missing), `git pull --ff-only` otherwise
- [x] 3.3 Script step: copy newest snapshot to `<repo>/omniroute-backup.sqlite`; skip commit/push if file content is byte-identical (no-op run)
- [x] 3.4 Script step: `git add` + `git commit -m "backup: <date>"` + `git push` with the pinned SSH key/known-hosts
- [x] 3.5 Script step: on `backup_rewrite_day`, perform orphan-branch history rewrite: create orphan commit with current snapshot, force-push (bounds repo size)
- [x] 3.6 Render script via Ansible template into `backup_script_path`, root-owned mode `0750`; run `bash -n` syntax check in a verify task

## 4. Systemd Timer Units

- [x] 4.1 Add `omniroute-backup.service` (oneshot, `ExecStart={{ backup_script_path }}`) with `Environment=GIT_SSH_COMMAND` pointing at the deploy key and known-hosts; root user
- [x] 4.2 Add `omniroute-backup.timer` with `OnCalendar={{ backup_schedule }}` and `Persistent=true`; enable + start via `systemd_service`/`service` modules
- [x] 4.3 Verify timer is active and next-run time is scheduled (check mode aware — skip runtime verification in check mode per existing role convention)

## 5. Verification & Docs

- [x] 5.1 Add a verify task (post-apply, non-check-mode) that triggers one manual backup run (`systemctl start omniroute-backup.service`), asserts exit 0, and confirms the file landed in the repo checkout
- [x] 5.2 Document in `README.md` Operations section: the one-time GitHub repo + deploy-key registration step, what the worker does, retention/rewrite behavior, and the manual restore procedure (replace `/var/lib/omniroute/storage.sqlite`, restart supervisor)
- [x] 5.3 Run local static checks: `ansible-playbook --syntax-check`, `--list-tasks`, and `ansible-lint` if installed; confirm no new lint errors
