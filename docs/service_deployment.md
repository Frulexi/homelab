# Docker Service Deployment Overview

The `ansible/playbooks/deploy-services.yml` playbook rolls out all Docker Compose workloads tracked in the repository, manages their configuration, and performs post-deployment cleanup.

## What the playbook does

- Discovers every service directory under `services/` that contains a `docker-compose.yml`.
- Copies compose files and optional `config/` subdirectories to the managed hosts under `/opt/homelab/services/<service>/`.
- Renders managed `.env` files from secrets stored in the Ansible vault, keeping secret data out of logs with `no_log: true`.
- Validates each compose project with `docker compose config` before starting containers.
- Restarts only the services whose compose files, configs, or secrets changed (unless `docker_service_force_recreate` overrides).
- Invokes the shared `docker-cleanup` role after deployments to prune unused Docker artifacts.

## Key variables (`group_vars/all.yml` + inventory overrides)

| Variable | Purpose | Default |
| --- | --- | --- |
| `docker_service_targets` | Limit deployment to specific services; empty = all discovered services. | `[]` |
| `docker_service_dest_root` | Base directory for deployed service artifacts. | `/opt/homelab/services` |
| `docker_service_env_files` | Mapping of service names to managed `.env` content (populate via vault). | `{}` |
| `docker_service_required_env` | List of services that must have entries in `docker_service_env_files`. | `[]` |
| `docker_service_force_recreate` | Force `docker compose up --force-recreate` on every run. | `false` |
| `docker_service_validate_config` | Run `docker compose config` sanity check before deployment. | `true` |
| `docker_service_run_cleanup` | Whether to trigger the `docker-cleanup` role afterward. | `true` |
| `docker_service_cleanup_level` | Cleanup intensity: `light`, `standard`, `deep`. | `standard` |

Override these per environment under `ansible/inventory/<env>/group_vars/` as needed. Service secrets belong in `ansible/inventory/<env>/group_vars/all/vault.yml` under `docker_service_env_files`.

## Secrets and `.env` handling

- Place placeholder values in each service’s `.env.example`; real secrets live in the vault. Example:

  ```yaml
  # ansible/inventory/production/group_vars/all/vault.yml
  docker_service_env_files:
    n8n: |
      N8N_ENCRYPTION_KEY=super-secret
  ```

- Ansible writes the secret contents to `/opt/homelab/services/<service>/.env` with mode `0640`. Set `docker_service_purge_missing_env: true` to remove managed `.env` files if a secret is deleted.
- Mark required secrets with `docker_service_required_env` to fail fast when a service is missing vault data.

## Cleanup options

The trailing play in `deploy-services.yml` calls the reusable `docker-cleanup` role. Key tunables:

- `docker_service_cleanup_level`: `light` prunes dangling images only, `standard` (default) adds stopped containers, networks, and build cache, `deep` also prunes unused volumes.
- `docker_service_cleanup_filters`: Supply filter strings (e.g., label selectors) to preserve specific resources:  
  ```yaml
  docker_service_cleanup_filters:
    images:
      - "label!=keep"
    volumes:
      - "label=managed"
  ```

Skip cleanup entirely by setting `docker_service_run_cleanup: false`.

## Running the playbook

```bash
# Deploy all services to production (prompts for vault secrets)
ansible-playbook -i ansible/inventory/production ansible/playbooks/deploy-services.yml --ask-vault-pass

# Deploy only specific services to staging
ansible-playbook -i ansible/inventory/staging ansible/playbooks/deploy-services.yml \
  -e "docker_service_targets=['n8n','homepage']" --ask-vault-pass
```

Use `--check --diff` for a dry run, or increase verbosity (`-v`) to see which services changed and the cleanup summary.

## Post-run verification

1. Inspect service state: `docker compose -p <service> ps` from `/opt/homelab/services/<service>/`.
2. Confirm `.env` permissions (`stat -c '%a %U:%G' .env`) and contents (vault values applied).
3. Review cleanup results with higher verbosity (`-vv`) for reclaimed-space summaries.
4. Check service-specific health checks or dashboards to ensure containers restarted cleanly.
