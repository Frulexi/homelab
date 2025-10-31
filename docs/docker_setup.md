# Docker Engine Setup Overview

The `ansible/playbooks/docker-setup.yml` playbook installs and configures Docker Engine, Docker Compose v2, and supporting tooling on Debian-based homelab hosts.

## What the playbook does

- Configures the official Docker APT repository, cleaning up any legacy entries that could conflict.
- Installs Docker Engine (CE), CLI tooling, Buildx, and the Compose v2 plugin.
- Manages the `docker` Unix group and optionally creates a dedicated service account.
- Ensures configurable directories (default `/opt/docker`) exist with proper ownership.
- Renders `/etc/docker/daemon.json` from templated variables, applying log rotation by default.
- Starts and enables the `docker` systemd service and restarts it automatically on configuration changes.

## Key variables (`group_vars/all.yml` and overrides)

| Variable | Purpose | Default |
| --- | --- | --- |
| `docker_users` | List of users to add to the `docker` group. | `[]` |
| `docker_create_service_account` | Whether to create a dedicated Docker service account. | `false` |
| `docker_service_account.name` | Name for the service account if created. | `docker` |
| `docker_daemon_config.log-opts.max-size` | Max log file size before rotation. | `"10m"` |
| `docker_daemon_config.log-opts.max-file` | Number of rotated log files kept. | `"5"` |
| `docker_daemon_registries` | Optional Docker registry mirrors to configure. | `[]` |
| `docker_repo_keyring_path` | Location where the Docker GPG key is stored. | `/etc/apt/keyrings/docker.asc` |
| `docker_packages` | Packages installed for Docker Engine and plugins. | `["docker-ce", "docker-ce-cli", "containerd.io", "docker-buildx-plugin", "docker-compose-plugin"]` |

Override these variables in `ansible/inventory/<env>/group_vars` or `host_vars` to tailor Docker users, logging, registry mirrors, or repository endpoints per environment.

## Requirements

- Supported OS: Debian-based distributions (Debian 10+/Ubuntu 20.04+ recommended).
- Outbound HTTPS connectivity to `download.docker.com` or a mirrored repository.
- Ansible collections installed per `ansible/collections/requirements.yml` (if used elsewhere).
- Target hosts must have `systemd` for service management.

## Running the playbook

```bash
ansible-playbook -i ansible/inventory/production ansible/playbooks/docker-setup.yml
```

To limit execution to a subset of hosts or perform a dry run:

```bash
# Run against staging hosts only
ansible-playbook -i ansible/inventory/staging ansible/playbooks/docker-setup.yml

# Perform a dry run with diff output
ansible-playbook -i ansible/inventory/production ansible/playbooks/docker-setup.yml --check --diff
```

## Post-run verification

1. Confirm Docker daemon status: `systemctl status docker`.
2. Validate docker group access: log in as each user in `docker_users` and run `docker ps`.
3. Inspect logging configuration: `docker info | grep -A3 'Logging Driver'`.
4. Review `/etc/docker/daemon.json` for expected registry mirrors and custom settings.

If configuration changes were applied, the playbook restarts Docker automatically. For manual intervention, run `sudo systemctl restart docker`.
