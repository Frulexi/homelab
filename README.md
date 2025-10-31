# Homelab Infrastructure as Code

This repository captures the migration of an existing Docker Compose based homelab into a fully automated Infrastructure as Code (IaC) stack powered by Ansible and GitHub Actions.

## Repository Layout

```
homelab/
├── .github/
│   └── workflows/        # CI/CD workflows (validation, deployment, etc.)
├── ansible/
│   ├── inventory/        # Inventory definitions (production, staging)
│   ├── playbooks/        # Playbooks orchestrating host configuration
│   ├── roles/            # Reusable Ansible roles
│   └── ansible.cfg       # Repository-scoped Ansible configuration
├── services/             # Docker Compose service definitions
├── docs/                 # Project documentation
├── .gitignore
└── README.md
```

## Getting Started

1. Install Ansible (`pip install ansible`) and Docker where applicable.
2. Copy the example inventory files under `ansible/inventory/` and adjust host variables for your environment.
3. Populate `services/` with per-service Docker Compose bundles.
4. Run playbooks from the repository root, e.g.:
   ```bash
   ansible-playbook -i ansible/inventory/production ansible/playbooks/hardening.yml
   ```

Further documentation lives in the `docs/` directory and will be expanded throughout the project phases.
