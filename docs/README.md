# Documentation Index

- `SETUP.md` – initial environment preparation steps (to be authored).
- `DEPLOYMENT.md` – deployment workflow and procedures (to be authored).
- `MAINTENANCE.md` – regular maintenance activities (to be authored).
- `TROUBLESHOOTING.md` – operational runbook for common issues (to be authored).
- `ARCHITECTURE.md` – high-level system overview (to be authored).
- `DISASTER_RECOVERY.md` – backup and restoration plan (to be authored).
- `hardening.md` – details of the baseline system hardening playbook.
- `services.md` – catalog of homelab services (to be authored).

## Work Log

### Completed
- Base repository structure with Ansible, services, docs, and workflows folders scaffolded.
- `.gitignore`, `README.md`, and baseline documentation index added.
- Ansible inventories split for production and staging; `group_vars` applied at environment level.
- `security` role created (SSH/UFW/fail2ban/unattended upgrades/logging) with supporting templates.
- `playbooks/hardening.yml` playbook authored and validated via `ansible-playbook --syntax-check`.
- `docs/hardening.md` documents variables and execution details for the hardening playbook.

### Next Up
- Duplicate `group_vars` for staging (`ansible/inventory/staging/group_vars/homelab.yml`) if needed.
- Implement Phase 2.2: Docker installation role and `playbooks/docker-setup.yml`.
- Author service deployment role/playbook for Phase 2.3.
- Begin documenting setup/deployment/maintenance guides in `docs/`.
- Prepare GitHub Actions workflows for validation and deployment pipelines.
