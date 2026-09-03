# Sumopod VPS Ansible

Local-only Ansible automation for the planned Ubuntu 24.04 Sumopod Lighthouse host. This repository does not contain credentials and has not connected to a VPS.

## Vault Setup

The playbook reads one encrypted file, `ansible/group_vars/vps/vault.yml`. Start from the example and keep real values local:

```bash
cp ansible/group_vars/vps/vault.yml.example ansible/group_vars/vps/vault.yml
chmod 600 ansible/group_vars/vps/vault.yml
ansible-vault edit ansible/group_vars/vps/vault.yml
ansible-vault view ansible/group_vars/vps/vault.yml
ansible-vault encrypt ansible/group_vars/vps/vault.yml  # only if currently plaintext
```

Populate every current key independently:

| Vault key | Purpose                                                     | Safe generation or entry                                                                                         |
| --- |-------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| `vault_ansible_host` | Provider-assigned public IPv4, IPv6, or hostname.           | Enter it in the encrypted editor.                                                                                |
| `vault_admin_authorized_key` | One complete OpenSSH public key line.                       | Run `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "email@example.com"`, then copy only `~/.ssh/id_ed25519.pub`. |
| `vault_admin_password_hash` | Password for the new generated admin                        | Run `openssl passwd -6`                                                                                          |
| `vault_letsencrypt_email` | Certificate renewal contact address in normal email format. | Enter it directly in the editor.                                                                                 |
| `vault_omniroute_jwt_secret` | Independent high-entropy JWT signing secret.                | Run `openssl rand -base64 48`                                                                                    |
| `vault_omniroute_api_key_secret` | Independent high-entropy API-key secret.                    | Run `openssl rand -base64 48`                                                                                    |
| `vault_omniroute_initial_password` | Initial OmniRoute application password.                     | Run `openssl rand -base64 36`                                                                                    |
| `vault_omniroute_ws_bridge_secret` | Independent WebSocket bridge authentication secret.         | Run `openssl rand -base64 48`.                                                                                   |
| `vault_omniroute_storage_encryption_key` | Independent 64-character hexadecimal storage key.           | Run `openssl rand -hex 32`.                                                                                      |
| `vault_backup_repo` | GitHub `owner/repo` slug of the private backup repository.      | Enter the slug, e.g. `you/omniroute-backup`.                                                 |
| `vault_backup_deploy_key` | Private half of the read/write SSH deploy key for that repo. | Run `ssh-keygen -t ed25519 -C "omniroute-backup" -f ~/.ssh/omniroute-backup`, paste the private key file, register the `.pub` half as a repo deploy key with write access. |
| `vault_backup_branch` | Branch the daily backup worker pushes to.                     | Enter the default branch name, usually `main`.                                                |

Run each command separately and paste its output immediately into `ansible-vault edit`; don't reuse values or save terminal transcripts.

Then, bootstrap with the existing `ubuntu` account:

```bash
cd ansible
ansible-playbook -i inventory/production.yml site.yml --ask-vault-pass --syntax-check
ansible-playbook -i inventory/production.yml site.yml --ask-vault-pass --check --diff
ansible-playbook -i inventory/production.yml site.yml --ask-vault-pass
```

Use the provider image's configured SSH key for bootstrap. Add `--ask-pass` only if that image was explicitly configured for password SSH, and add `--ask-become-pass` only when sudo requires it. Check mode contacts the host but skips runtime verification and tasks whose prerequisites would only be created by the real run.

Keep the original Ubuntu session open. Verify `ssh -o PreferredAuthentications=publickey nmfdev@VPS_PUBLIC_IP 'id; sudo -v'` in a separate terminal. Only then change `ansible_user: ubuntu` to `ansible_user: nmfdev` in the inventory and omit `--ask-pass` for later runs.

## Architecture

- Public firewall exposure is SSH, HTTP, and HTTPS only; OmniRoute listens on `127.0.0.1:20128`.
- OmniRoute runs as unprivileged `omniroute` under Supervisor. Ansible installs exact `omniroute@3.8.49` with npm into the dedicated, service-owned `/var/lib/omniroute/npm` prefix, and the secret-loading wrapper executes `/var/lib/omniroute/npm/bin/omniroute` directly without resolving packages during service restarts.
- Node.js 24 is installed from NodeSource's signed `node_24.x` APT repository and satisfies OmniRoute 3.8.49's declared Node engine (`>=22.22.2 <23 || >=24.0.0 <27`).
- Nginx serves the existing `nmfdev.web.id` `index.html` unchanged and reverse-proxies `ai-proxy.nmfdev.web.id`.
- Let’s Encrypt is intentionally documented as a post-DNS execution step; certificates and private keys remain on the host.

## Operations

Run local static checks before any host execution from the `ansible/` directory: `ansible-playbook -i inventory/production.yml site.yml --syntax-check`, `ansible-playbook -i inventory/production.yml site.yml --list-tasks`, `ansible-inventory -i inventory/production.yml --list`, and `ansible-lint site.yml` when installed. A real `--check` contacts the target but is dependency-aware for first-run previews; post-apply service verification is intentionally skipped in check mode.

The Node.js role configures NodeSource directly with an APT keyring and deb822 `Signed-By` repository definition; it does not execute a `curl | bash` installer. It validates that `/usr/bin/node` reports major version 24. On a pristine host, check mode previews repository/package and OmniRoute installation changes but skips validation of executables that would only be created by the real run; on an already provisioned host, installed runtime validation remains active.

OmniRoute's npm prefix and npm cache are owned by the unprivileged `omniroute` account. Ansible probes the installed package metadata and only invokes npm when the exact pinned version is absent, so Supervisor restarts never contact the registry. `OMNIROUTE_MEMORY_MB` is the sole heap setting: OmniRoute 3.8.49 converts it to the spawned server's Node `--max-old-space-size` value. Do not add a competing heap value to `NODE_OPTIONS`, because an explicit Node heap flag takes precedence over `OMNIROUTE_MEMORY_MB`.

Before an OmniRoute upgrade, stop it through Supervisor, archive `/var/lib/omniroute` with xattrs/acls into `/var/backups` (root-only, with retention), record Node and package versions, update the exact package pin, apply the playbook, and verify `/healthz`, logs, and HTTPS. Roll back by restoring the prior package pin and data archive.

For emergency recovery, use the Tencent Lighthouse browser console, restore `/root/pre-ansible-backup`, temporarily restore a known-good SSH drop-in and UFW rule, validate with `sshd -t`/`nginx -t`, then reload. Certificate issues require checking DNS, HTTP challenge reachability, `/etc/letsencrypt/live/`, and `certbot renew --dry-run`.

Oh My Zsh is enabled for `nmfdev` by default. The base role installs `zsh`, checks out the configured official Oh My Zsh repository revision into `/home/nmfdev/.oh-my-zsh`, sets the login shell to `/usr/bin/zsh`, and creates `.zshrc` only when it does not already exist. Existing `.zshrc` files are never overwritten, and the `omniroute` service user is not targeted. To disable installation and shell changes, set `enable_oh_my_zsh: false` in `ansible/group_vars/vps/vars.yml`; to upgrade, review and replace `oh_my_zsh_repo_revision` with a vetted commit from the official repository.

A daily backup worker (`omniroute-backup.timer`) publishes the newest OmniRoute SQLite snapshot to the configured GitHub private repository. It copies the app's own consistent snapshot from `/var/lib/omniroute/db_backups/` to a single fixed file (`omniroute-backup.sqlite`) in the repository root, commits, and pushes each day at 03:00 (with `Persistent=true` catch-up). On the 1st of each month it rewrites the repository history to a single commit and force-pushes, so the remote stays small; history is therefore current-state-only, not a point-in-time archive. The built-in local auto-backup (keep-last-5) remains enabled as the on-host safety net.

### One-time GitHub setup (manual)

1. Create a private repository (e.g. `you/omniroute-backup`).
2. Generate a keypair: `ssh-keygen -t ed25519 -C "omniroute-backup" -f ~/.ssh/omniroute-backup`.
3. Add `~/.ssh/omniroute-backup.pub` as a deploy key on the repository with write access enabled.
4. Fill `vault_backup_repo` (the `owner/repo` slug), `vault_backup_deploy_key` (the private key), and `vault_backup_branch` in the vault.

### Restore from the GitHub backup

On a replacement host, clone the repository (or fetch the file from the GitHub web UI), stop OmniRoute through Supervisor, replace `/var/lib/omniroute/storage.sqlite` with the backed-up file, and start OmniRoute again. The snapshot is a standalone SQLite file and needs no WAL sidecars.

## Safety

No Docker, Terraform, OmniRoute systemd unit, private key, password, API key, JWT, or plaintext secret is included. The production inventory is operational metadata and currently contains the target host address.
