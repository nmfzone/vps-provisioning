# Sumopod VPS Ansible

Local-only Ansible automation for the planned Ubuntu 24.04 Sumopod Lighthouse host. This repository does not contain credentials and has not connected to a VPS.

## Before execution

1. Copy `ansible/group_vars/vps/vault.yml.example` to `ansible/group_vars/vps/vault.yml` and replace every placeholder using `ansible-vault encrypt ansible/group_vars/vps/vault.yml`. Do not commit the plaintext file.
2. Review the host address in `ansible/inventory/production.yml`; keep public keys, email, and credentials only in the encrypted vault.

The `nmfdev` administrator password is stored only as a SHA-512 crypt hash in the encrypted vault (`vault_admin_password_hash`). Generate it interactively without putting the plaintext or hash in shell history, then edit the vault interactively:

```bash
python3 -c 'import getpass, crypt; print(crypt.crypt(getpass.getpass("Password: "), crypt.mksalt(crypt.METHOD_SHA512)))'
ansible-vault edit ansible/group_vars/vps/vault.yml
```

Replace `vault_admin_password_hash` with the generated `$6$...` value. The playbook rejects the example placeholder and any hash that does not use SHA-512 crypt. This password is for sudo/local console use; SSH password authentication remains disabled.
3. Bootstrap with the existing `ubuntu` account:

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
- OmniRoute runs as unprivileged `omniroute` under Supervisor, using native `/usr/local/bin/bunx omniroute@3.8.49 serve --no-open --no-recovery --port 20128`.
- Node.js 24 is installed before Bun from NodeSource's signed `node_24.x` APT repository. This provides `/usr/bin/node` for OmniRoute's published `#!/usr/bin/env node` executable; Bunx remains the package resolver and launcher.
- Nginx serves the existing `nmfdev.web.id` `index.html` unchanged and reverse-proxies `ai-proxy.nmfdev.web.id`.
- Let’s Encrypt is intentionally documented as a post-DNS execution step; certificates and private keys remain on the host.

## Operations

Run local static checks before any host execution from the `ansible/` directory: `ansible-playbook -i inventory/production.yml site.yml --syntax-check`, `ansible-playbook -i inventory/production.yml site.yml --list-tasks`, `ansible-inventory -i inventory/production.yml --list`, and `ansible-lint site.yml` when installed. A real `--check` contacts the target but is dependency-aware for first-run previews; post-apply service verification is intentionally skipped in check mode.

The Node.js role configures NodeSource directly with an APT keyring and deb822 `Signed-By` repository definition; it does not execute a `curl | bash` installer. It validates that `/usr/bin/node` reports major version 24. On a pristine host, check mode previews repository/package changes but skips Node validation and OmniRoute Bunx resolution because those executables are only created by the real run; on an already provisioned host, installed runtime validation remains active.

Before an OmniRoute upgrade, stop it through Supervisor, archive `/var/lib/omniroute` with xattrs/acls into `/var/backups` (root-only, with retention), record Bun and package versions, update the exact package pin, run the Bunx preflight, and verify `/healthz`, logs, and HTTPS. Roll back by restoring the prior package pin and data archive.

For emergency recovery, use the Tencent Lighthouse browser console, restore `/root/pre-ansible-backup`, temporarily restore a known-good SSH drop-in and UFW rule, validate with `sshd -t`/`nginx -t`, then reload. Certificate issues require checking DNS, HTTP challenge reachability, `/etc/letsencrypt/live/`, and `certbot renew --dry-run`.

Oh My Zsh is enabled for `nmfdev` by default. The base role installs `zsh`, checks out the configured official Oh My Zsh repository revision into `/home/nmfdev/.oh-my-zsh`, sets the login shell to `/usr/bin/zsh`, and creates `.zshrc` only when it does not already exist. Existing `.zshrc` files are never overwritten, and the `omniroute` service user is not targeted. To disable installation and shell changes, set `enable_oh_my_zsh: false` in `ansible/group_vars/vps/vars.yml`; to upgrade, review and replace `oh_my_zsh_repo_revision` with a vetted commit from the official repository.

## Safety

No Docker, Terraform, OmniRoute systemd unit, private key, password, API key, JWT, or plaintext secret is included. The production inventory is operational metadata and currently contains the target host address.
