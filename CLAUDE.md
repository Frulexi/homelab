# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Homelab Infrastructure as Code (IaC) project. Ansible manages a Docker Compose-based home server with services deployed to `/opt/homelab/services/<service>/` on managed hosts. Secrets are managed via Ansible Vault.

## Common Commands

All commands run from the repo root. **Always activate the venv first:**

```bash
source .venv/bin/activate
```

```bash
# Install required Ansible collections
ansible-galaxy collection install -r ansible/collections/requirements.yml

# Deploy all services (production)
ansible-playbook -i ansible/inventory/production ansible/playbooks/deploy-services.yml --vault-password-file ~/.vault_pass.txt

# Deploy specific services only
ansible-playbook -i ansible/inventory/production ansible/playbooks/deploy-services.yml \
  -e '{"docker_service_targets": ["traefik", "monitoring"]}' --vault-password-file ~/.vault_pass.txt

# Force redeploy even when no files changed
ansible-playbook -i ansible/inventory/production ansible/playbooks/deploy-services.yml \
  -e '{"docker_service_targets": ["traefik"], "docker_service_force_recreate": true}' --vault-password-file ~/.vault_pass.txt

# System hardening
ansible-playbook -i ansible/inventory/production ansible/playbooks/hardening.yml

# Docker installation
ansible-playbook -i ansible/inventory/production ansible/playbooks/docker-setup.yml

# Dry run / check mode
ansible-playbook -i ansible/inventory/production ansible/playbooks/deploy-services.yml \
  --check --diff --vault-password-file ~/.vault_pass.txt

# Syntax validation
ansible-playbook ansible/playbooks/deploy-services.yml --syntax-check

# Edit encrypted vault secrets
ansible-vault edit ansible/inventory/production/group_vars/all/vault.yml
```

Replace `production` with `staging` for the staging environment. Use `--ask-vault-pass` only in interactive terminals; prefer `--vault-password-file ~/.vault_pass.txt` for non-interactive runs.

## Architecture

### Ansible Structure

- **Playbooks** (`ansible/playbooks/`): Three orchestration playbooks — `hardening.yml` (security), `docker-setup.yml` (Docker Engine), `deploy-services.yml` (service deployment + cleanup).
- **Roles** (`ansible/roles/`):
  - `security/` — SSH hardening, UFW firewall, fail2ban, unattended upgrades, syslog
  - `docker/` — Docker Engine + Compose v2 installation and daemon config
  - `docker_service/` — Dynamic service discovery and deployment (see below)
  - `docker-cleanup/` — Configurable cleanup (light/standard/deep) via `cleanup_level` variable
- **Inventory** (`ansible/inventory/`): Separate `production/` and `staging/` directories, each with `hosts.ini` and `group_vars/`.
- **Shared group_vars** (`ansible/group_vars/all.yml`): Default values for security and Docker variables, inherited by all inventories. Per-environment overrides live in `ansible/inventory/<env>/group_vars/homelab.yml`.
- **Vault password**: Expected at `~/.vault_pass.txt` (configured in `ansible.cfg`).

### Service Discovery & Deployment

The `docker_service` role automatically discovers services by scanning `services/` for `docker-compose.yml` files. For each discovered service it:

1. Copies `docker-compose.yml` to the remote host under `/opt/homelab/services/<name>/`
2. Copies `config/` directory if present (non-sensitive configs)
3. Deploys `.env` files from vault-encrypted `docker_service_env_files` variable (with `no_log: true`)
4. Runs `docker compose up -d` — **only when files changed** (or unconditionally when `docker_service_force_recreate: true` or `docker_service_always_run: true`)

Use `docker_service_targets` to limit deployment to specific services. When empty, all discovered services deploy.

Key `docker_service` role variables (defaults in `ansible/roles/docker_service/defaults/main.yml`):

| Variable | Default | Purpose |
|---|---|---|
| `docker_service_force_recreate` | `false` | Force `up -d` even when no files changed |
| `docker_service_always_run` | `false` | Always run compose up (without `--force-recreate`) |
| `docker_service_validate_config` | `true` | Run `docker compose config` before deploying |
| `docker_service_purge_stale_config` | `false` | Remove remote `config/` when absent locally |
| `docker_service_purge_missing_env` | `false` | Remove remote `.env` when secret removed from vault |
| `docker_service_required_env` | `[]` | Services that must have vault secrets or deployment fails |

### Services Directories

There are three service source directories at the repo root:

- **`services/`** — Production services. Used by the production inventory (default).
- **`dev_services/`** — Staging/dev services. Used by the staging inventory via a `docker_service_source_root` override in `ansible/inventory/staging/group_vars/homelab.yml`.
- **`od_services/`** — On-demand services (managed via Sablier for auto-start on HTTP request). Not deployed by `deploy-services.yml` unless `docker_service_source_root` is explicitly pointed at it.

The `docker_service` role discovers services from whichever directory `docker_service_source_root` points to. Production inherits the role default (`services/`); staging overrides it to `dev_services/`. Place dev-only services or modified versions of production services in `dev_services/`.

Each subdirectory under a source directory represents a deployable service containing at minimum a `docker-compose.yml`.

### Adding a New Service

1. Create `services/<service-name>/docker-compose.yml` (plus `config/` if needed).
2. Add `<service-name>_network` to `docker_service_networks` in `ansible/inventory/production/group_vars/homelab.yml` (so the network is created before deployment).
3. If the service needs secrets, add an entry under `docker_service_env_files.<service-name>` in the vault and add it to `docker_service_required_env` if it must not deploy without secrets.
4. If the service joins multiple Docker networks, add `traefik.docker.network: "traefik_network"` to its Traefik labels — otherwise Traefik may pick the wrong network IP and fail to route traffic.

### Traefik Integration

Services expose themselves to Traefik via Docker container labels. Standard label set:

```yaml
labels:
  traefik.enable: "true"
  traefik.http.routers.<name>.entrypoints: "websecure"
  traefik.http.routers.<name>.tls.certresolver: "cloudflare"
  traefik.http.routers.<name>.rule: "Host(`<name>.prod.landingsnet.com`)"
  traefik.http.services.<name>.loadbalancer.server.port: "<port>"
  traefik.docker.network: "traefik_network"   # Required when container is on multiple networks
```

All production services use `*.prod.landingsnet.com`. The `traefik.docker.network` label is mandatory when a container belongs to more than one Docker network — without it Traefik may resolve an IP from the wrong network and be unable to reach the backend.

### Homepage Dashboard Integration

Homepage auto-discovers containers via Docker labels. Add these to any service that should appear on the dashboard:

```yaml
labels:
  homepage.name: "Display Name"
  homepage.description: "Short description"
  homepage.icon: "icon-name.png"   # see https://github.com/walkxcode/dashboard-icons
  homepage.group: "Group Name"      # must match a group defined in services/homepage/config/settings.yaml
  homepage.href: "https://<name>.prod.landingsnet.com"
```

New groups must be added to `services/homepage/config/settings.yaml` under the `layout:` key before they will render.

### Networking

Services communicate over shared Docker networks defined in `docker_service_networks` in `ansible/inventory/production/group_vars/homelab.yml`. The `docker_service` role ensures these networks exist before deployment. All networks are declared `external: true` in compose files — Ansible creates them, not Compose. Traefik acts as the reverse proxy, routing via container labels. Sablier handles on-demand container lifecycle (auto-start on HTTP request) and is configured in the `traefik` service.

### Secret Management

- Service secrets stored in `ansible/inventory/<env>/group_vars/all/vault.yml` (encrypted with `ansible-vault`)
- Structured under `docker_service_env_files.<service-name>` as multi-line strings
- Services requiring secrets are listed in `docker_service_required_env`; deployment fails if their entries are missing
- `.env` files and vault files are gitignored

## Key Configuration

- **SSH**: Non-standard port 1022, key-only auth, no root login. Production host `prod-vm1` (192.168.1.191) uses key `~/.ssh/prod-vm1.pem`.
- **ansible.cfg**: Vault password file at `~/.vault_pass.txt`, YAML stdout callback, roles path set to `ansible/roles`
- **Renovate** (`renovate.json`): Automated Docker image updates on weekends, grouped by service category. `apple-trans` and `twingate/connector` are excluded.
- **Required Ansible collections**: `community.general`, `community.docker`
