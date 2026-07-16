# headscaler
An Ansible playbook to deploy a hardened, dockerized [headscale](https://github.com/juanfont/headscale) + [headplane](https://github.com/tale/headplane) stack.

The deployed stack runs:

- Docker + Compose
- Headscale (VPN control plane)
- Headplane (web UI)
- Traefik (reverse proxy with automatic Let's Encrypt TLS)
- Tailscale (VPN node on the server itself)

Headplane is protected with HTTP Basic Auth. OIDC configuration is available in `templates/headplane-config.yaml.j2` if you need SSO.

---

# First-Time Setup

## 1. Server

Spin up a server with a public IP and two DNS A records pointing to it:
- `hs.example.com` → your server IP
- `plane.example.com` → your server IP

SSH in and create the sudoer user Ansible will use. **SSH root access will be disabled by the hardening role.**

```bash
useradd -m -s /bin/bash headscaler
visudo
```

Add this line:
```
headscaler ALL=(ALL) NOPASSWD: ALL
```

Copy your local machine's public key to `/home/headscaler/.ssh/authorized_keys`.

## 2. Local Machine

Clone the repo:
```bash
git clone https://github.com/nicoloboatto/headscaler.git
cd headscaler
```

Install Ansible dependencies:
```bash
ansible-galaxy install -r requirements.yml
```

Create your local config files from the examples — these are gitignored and never committed:
```bash
cp config.yml.example config.yml
cp inventory.ini.example inventory.ini
```

Edit `config.yml` with your domains, email, and version pins. Edit `inventory.ini` with your server's hostname:
```ini
[headscale_stack]
hs-01 ansible_host=hs.example.com
```

## 3. Deploy

Dry-run first:
```bash
ansible-playbook -i inventory.ini deploy_headscale_stack.yml --check
```

Deploy:
```bash
ansible-playbook -i inventory.ini deploy_headscale_stack.yml
```

At the end of the run, credentials are printed. **Save them — they are not stored anywhere.**
```
┌──────────────────────────────────────────────────────────────────
│  Headscale URL    : https://hs.example.com
│  Headplane URL    : https://plane.example.com
│  Plane login      : admin / <generated-password>
│  HP cookie secret : <generated-secret>
│  HS API Key       : <generated-api-key>
└──────────────────────────────────────────────────────────────────
```

## 4. Verify

Connect a Tailscale client:
```bash
tailscale login --login-server https://hs.example.com
```

Open the Headplane UI at `https://plane.example.com`. You'll be prompted for the Basic Auth credentials and the Headscale API key shown above.

---

# Updating the Stack

The update playbook handles headscale's requirement that SQLite migrations run sequentially — it steps through every intermediate minor version between what's running and the target, backing up the database before it starts.

Headplane and Traefik are updated directly (no sequential migration required).

Set the target versions in `config.yml`, then run:
```bash
ansible-playbook -i inventory.ini update_headscale_stack.yml
```

To update only the Tailscale node (e.g., to refresh an expired pre-auth key):
```bash
ansible-playbook -i inventory.ini update_headscale_stack.yml --tags tailscale
```

To update Ansible dependencies:
```bash
ansible-galaxy install -r requirements.yml --force
```

---

# Configuration

All variables live in `config.yml` (created from `config.yml.example`). Key settings:

| Variable | Purpose |
|---|---|
| `headscale_version` | Target headscale version (e.g. `v0.29.2`) |
| `headplane_version` | Target headplane version (e.g. `0.7.0`) |
| `traefik_version` | Target Traefik version (e.g. `v3.7.7`) |
| `tailscale_version` | Target Tailscale version (e.g. `stable`) |
| `hs_domain` | Public domain for headscale |
| `hp_domain` | Public domain for headplane |
| `acme_email` | Email for Let's Encrypt |
| `ts_user` | Headscale user the server node registers under |
| `ts_authkey_lifetime` | Pre-auth key lifetime (e.g. `1000d`) |

Only stable releases should be used for `headscale_version`. The update playbook automatically filters out pre-releases and draft releases when computing the upgrade path.

---

# DNS Records

Custom DNS records (for MagicDNS) can be managed from the Headplane UI or by editing `headscale/extra-records.json` directly on the server after deployment.
