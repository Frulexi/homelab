# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Homelab Infrastructure as Code (IaC) project. Ansible manages a Docker Compose-based home server with services deployed to `/opt/homelab/services/<service>/` on managed hosts. Secrets are managed via Ansible Vault.

## Common Commands

All commands run from the repo root.

```bash
# Install required Ansible collections
ansible-galaxy collection install -r ansible/collections/requirements.yml

# Deploy all services (production)
ansible-playbook -i ansible/inventory/production ansible/playbooks/deploy-services.yml --ask-vault-pass

# Deploy specific services only
ansible-playbook -i ansible/inventory/production ansible/playbooks/deploy-services.yml \
  -e '{"docker_service_targets": ["traefik", "monitoring"]}' --ask-vault-pass

# System hardening
ansible-playbook -i ansible/inventory/production ansible/playbooks/hardening.yml

# Docker installation
ansible-playbook -i ansible/inventory/production ansible/playbooks/docker-setup.yml

# Dry run / check mode
ansible-playbook -i ansible/inventory/production ansible/playbooks/deploy-services.yml \
  --check --diff --ask-vault-pass

# Syntax validation
ansible-playbook ansible/playbooks/deploy-services.yml --syntax-check

# Edit encrypted vault secrets
ansible-vault edit ansible/inventory/production/group_vars/all/vault.yml
```

Replace `production` with `staging` for the staging environment.

## Architecture

### Ansible Structure

- **Playbooks** (`ansible/playbooks/`): Three orchestration playbooks — `hardening.yml` (security), `docker-setup.yml` (Docker Engine), `deploy-services.yml` (service deployment + cleanup).
- **Roles** (`ansible/roles/`):
  - `security/` — SSH hardening, UFW firewall, fail2ban, unattended upgrades, syslog
  - `docker/` — Docker Engine + Compose v2 installation and daemon config
  - `docker_service/` — Dynamic service discovery and deployment (see below)
  - `docker-cleanup/` — Configurable cleanup (light/standard/deep) via `cleanup_level` variable
- **Inventory** (`ansible/inventory/`): Separate `production/` and `staging/` directories, each with `hosts.ini` and `group_vars/`.
- **Global defaults** (`ansible/group_vars/all.yml`): Security and Docker default variables.
- **Vault password**: Expected at `~/.vault_pass.txt` (configured in `ansible.cfg`).

### Service Discovery & Deployment

The `docker_service` role automatically discovers services by scanning `services/` for `docker-compose.yml` files. For each discovered service it:

1. Copies `docker-compose.yml` to the remote host under `/opt/homelab/services/<name>/`
2. Copies `config/` directory if present (non-sensitive configs)
3. Deploys `.env` files from vault-encrypted `docker_service_env_files` variable (with `no_log: true`)
4. Runs `docker compose up -d`

Use `docker_service_targets` to limit deployment to specific services. When empty, all discovered services deploy.

### Services Directories

There are two service source directories at the repo root:

- **`services/`** — Production services. Used by the production inventory (default).
- **`dev_services/`** — Staging/dev services. Used by the staging inventory via a `docker_service_source_root` override in `ansible/inventory/staging/group_vars/homelab.yml`.

The `docker_service` role discovers services from whichever directory `docker_service_source_root` points to. Production inherits the role default (`services/`); staging overrides it to `dev_services/`. Place dev-only services or modified versions of production services in `dev_services/`.

Each subdirectory under either source directory represents a deployable service containing at minimum a `docker-compose.yml`. Current production services: traefik, monitoring (Grafana/Prometheus/Loki/Alloy), homeassistant, authentik, n8n, portainer, unifi, apple-trans, actualbudget, minecraft-bedrock, twingate, github-runner.

### Networking

Services communicate over shared Docker networks defined in `docker_service_networks` (in group_vars). The `docker_service` role ensures these networks exist before deployment. Traefik acts as the reverse proxy, routing via container labels. Sablier handles on-demand container lifecycle (auto-start on HTTP request).

### Secret Management

- Service secrets stored in `ansible/inventory/<env>/group_vars/all/vault.yml` (encrypted with `ansible-vault`)
- Structured under `docker_service_env_files.<service-name>` as multi-line strings
- Services requiring secrets are listed in `docker_service_required_env`; deployment fails if their entries are missing
- `.env` files and vault files are gitignored

## Key Configuration

- **SSH**: Non-standard port 1022, key-only auth, no root login
- **ansible.cfg**: Vault password file at `~/.vault_pass.txt`, YAML stdout callback, roles path set to `ansible/roles`
- **Renovate** (`renovate.json`): Automated Docker image updates on weekends, grouped by service category
- **Required Ansible collections**: `community.general`, `community.docker`
