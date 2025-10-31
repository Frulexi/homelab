# System Hardening Overview

The `ansible/playbooks/hardening.yml` playbook applies a baseline security posture to all hosts in the `homelab` inventory group.

## What the playbook does

- Enforces SSH hardening (key-only auth, disables root login, limits ciphers/MACs, configurable port).
- Manages the local firewall via UFW, allowing only declared TCP/UDP ports.
- Installs and configures fail2ban to throttle SSH brute-force attempts.
- Enables unattended security updates with optional reboot and notification policies.
- Optionally forwards logs to a remote syslog collector.
- Stops and disables any services listed in `security_disable_services`.

## Key variables (`group_vars/all.yml`)

| Variable | Purpose | Default |
| --- | --- | --- |
| `security_ssh_port` | SSH listening port. | `1022` |
| `security_ssh_permit_root_login` | Controls root SSH login. | `"no"` |
| `security_ssh_password_authentication` | Enables/disables password auth. | `"no"` |
| `security_firewall_allowed_tcp_ports` | TCP ports kept open by UFW. | `[22]` |
| `security_fail2ban_maxretry` | Failed attempts before ban. | `5` |
| `security_fail2ban_findtime` | Interval (seconds) to count retries. | `600` |
| `security_fail2ban_bantime` | Ban duration in seconds. | `3600` |
| `security_unattended_upgrades_auto_reboot` | Auto reboots after updates. | `true` |
| `security_unattended_upgrades_reboot_time` | Scheduled reboot time. | `"02:30"` |
| `security_syslog_remote_host` | Remote syslog target (blank disables). | `""` |
| `security_disable_services` | Services to stop/disable. | `[]` |

Override these defaults per environment (e.g., staging vs. production) by creating host/group vars inside `ansible/inventory/<env>/group_vars` or `host_vars`. Global defaults live at `ansible/inventory/group_vars/all.yml`.

## Requirements

- Ansible collection `community.general` (install with `ansible-galaxy collection install -r ansible/collections/requirements.yml`).
- Target hosts should run a Debian-based distribution with `ufw`, `fail2ban`, and `unattended-upgrades` available.

## Running the playbook

```bash
ansible-galaxy collection install -r ansible/collections/requirements.yml
ansible-playbook -i ansible/inventory/production ansible/playbooks/hardening.yml
```
