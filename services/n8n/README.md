# n8n Automation Platform

## Purpose
- Self-hosted workflow automation platform for orchestrating integrations and tasks.

## Ports
- `5678:5678` (HTTP web UI and API)

## Volumes
- `n8n_data` → persists `/home/node/.n8n` workflow state and credentials inside the container.

## Dependencies
- Requires Docker Engine with Compose v2.
- Expects reverse proxy/SSL termination to be handled externally if exposed publicly.

## Secrets
- `.env` (not committed) must define `N8N_ENCRYPTION_KEY` for credential encryption.
- Optional environment variables (host, port, protocol) default to local testing values.
- Production secrets should be stored in `ansible/group_vars/all/vault.yml` under `docker_service_env_files.n8n`.

## Deployment Notes
- Deploy via `ansible/playbooks/deploy-services.yml`.
- Healthcheck polls `http://localhost:5678/healthz`; ensure container image includes `curl` or install it.
- Adjust volume mappings if external storage is required.
