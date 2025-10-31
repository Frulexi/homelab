# Project Plan: Homelab Infrastructure as Code Migration

## Project Overview

**Goal:** Migrate existing Docker Compose home server to full IaC with CI/CD pipeline

**Tech Stack:** Ansible + Docker Compose + GitHub Actions + Self-Hosted Runner

**Timeline:** Phased approach with incremental delivery


---

## Phase 1: Foundation & Setup (Week 1)

### 1.1 Repository Structure Setup

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  Create base directory structure
- [ ]  Set up .gitignore for sensitive files
- [ ]  Create README.md with project overview
- [ ]  Initialize Ansible directory structure
- [ ]  Set up inventory files (production/staging)

**Acceptance Criteria:**

`homelab/
├── .github/
│   └── workflows/
├── ansible/
│   ├── inventory/
│   │   ├── production
│   │   └── staging
│   ├── playbooks/
│   ├── roles/
│   └── ansible.cfg
├── services/
├── docs/
├── .gitignore
└── README.md`

**Reviewer Focus:**

- Directory structure follows Ansible best practices
- .gitignore includes all sensitive files (vault passwords, .env files, etc.)
- README has clear setup instructions

---

### 1.2 GitHub Actions Self-Hosted Runner Setup

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  Runner installation script/documentation
- [ ]  Systemd service configuration for runner
- [ ]  Runner labels and tags configured
- [ ]  Test workflow to verify runner connectivity

**Acceptance Criteria:**

- Runner shows as "online" in GitHub
- Test workflow executes successfully
- Runner configured to start on boot
- Documentation for runner maintenance

**Reviewer Focus:**

- Security: Runner runs as dedicated user
- Service properly configured for auto-restart
- Clear documentation for troubleshooting

---

## Phase 2: Ansible Foundation (Week 1-2)

### 2.1 System Hardening Playbook

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  `playbooks/hardening.yml` - SSH hardening, firewall, fail2ban
- [ ]  Role: `roles/security/` with modular tasks
- [ ]  Variables for configurable security settings
- [ ]  Documentation of security measures applied

**Tasks to implement:**

yaml

- `SSH key-only authentication
- Disable root login
- Configure UFW firewall
- Install and configure fail2ban
- Set up automatic security updates
- Configure system logging
- Disable unnecessary services`

**Acceptance Criteria:**

- Playbook runs idempotently (can run multiple times safely)
- All security tasks have proper error handling
- Variables in `group_vars/all.yml` for customization
- Playbook tested on clean system

**Reviewer Focus:**

- Security best practices followed
- No hardcoded values (use variables)
- Proper use of `become` privileges
- Tasks are idempotent

---

### 2.2 Docker Installation Playbook

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  `playbooks/docker-setup.yml`
- [ ]  Role: `roles/docker/`
- [ ]  Install Docker Engine and Docker Compose v2
- [ ]  Create docker user/group
- [ ]  Configure Docker daemon settings

**Tasks to implement:**

yaml

- `Add Docker apt repository
- Install Docker Engine
- Install Docker Compose v2
- Create docker group
- Add service user to docker group
- Create /opt directory structure
- Configure Docker logging
- Set up Docker daemon.json`

**Acceptance Criteria:**

- Docker and Compose installed and working
- Non-root user can run docker commands
- Docker service starts on boot
- Logging configured (json-file with rotation)

**Reviewer Focus:**

- Latest stable Docker version
- Proper user permissions
- Docker daemon configuration follows best practices

---

### 2.3 Service Deployment Playbook

**Deliverables:**

- [ ]  `playbooks/deploy-services.yml`
- [ ]  Role: `roles/docker-service/`
- [ ]  Task to copy compose files
- [ ]  Task to copy config directories (non-sensitive)
- [ ]  Task to handle .env files and secrets
- [ ]  Task to deploy with docker compose
- [ ]  Task for Docker cleanup

**Key features:**

yaml

- `Dynamic service discovery from services/ directory
- Copy docker-compose.yml to /opt/{service}/
- Copy config directories if they exist
- Handle .env files:
  - Services should include `.env.example` files in repo (with placeholder values)
  - Actual `.env` files with secrets managed via:
    - Ansible vault for encrypted .env templates
- Run docker compose up -d
- Handle service restarts on changes
- Cleanup old images/containers
```
**Acceptance Criteria:**
- Playbook discovers services automatically
- Only changed services restart
- Config files copied correctly (non-sensitive)
- .env handling strategy documented and implemented
- Secrets never exposed in logs or repository
- Clear error messages if required secrets/env files missing
- Proper error handling for failed deployments
- Cleanup runs after successful deployment
---

services/
├── service-name/
│   ├── docker-compose.yml
│   ├── .env.example (template with placeholders)
│   ├── config/ (non-sensitive configs only)
│   └── README.md (service-specific notes including secret requirements)

**For each service:**

- [ ]  Compose file validated (`docker compose config`)
- [ ]  Volume paths use relative or configurable paths
- [ ]  Networks defined if inter-service communication needed
- [ ]  Environment variables documented in README.md
- [ ]  `.env.example` created with all required variables (placeholder values)
- [ ]  Actual `.env` excluded from git (verify .gitignore)
- [ ]  Secrets handling documented (what needs vault/manual setup)


**Acceptance Criteria:**

- All services have valid compose files
- No hardcoded host-specific paths
- All services documented
- Test deployment on staging

**Reviewer Focus:**

- Compose files follow consistent structure
- No exposed secrets in compose files
- Proper use of networks and volumes
- Resource limits defined where appropriate

---

### 3.2 Create Service Inventory

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  `docs/services.md` - Complete service catalog
- [ ]  Dependency map (which services depend on others)
- [ ]  Port allocation table
- [ ]  Network topology diagram

**Service documentation template:**

markdown

`## Service Name
- ****Purpose:**** Brief description
- ****Ports:**** 8080:80
- ****Volumes:**** /data, /config
- ****Dependencies:**** None / depends on service-x
- ****Notes:**** Special considerations`

**Reviewer Focus:**

- All services documented
- Port conflicts identified
- Dependencies clear
- Special requirements noted

---

## Phase 4: CI/CD Pipeline (Week 3)

### 4.1 Validation Workflow

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  `.github/workflows/validate.yml`
- [ ]  Runs on all PRs and pushes
- [ ]  Validates all compose files
- [ ]  Validates Ansible syntax
- [ ]  Lints YAML files

**Workflow steps:**

yaml

`1. Checkout code
2. Validate all docker-compose.yml files
3. Run ansible-playbook --syntax-check
4. Run ansible-lint (if applicable)
5. Check for secrets in code
6. YAML lint all files`

**Acceptance Criteria:**

- Workflow runs on every push/PR
- Fails on invalid compose files
- Fails on Ansible syntax errors
- Fast execution (< 2 minutes)

**Reviewer Focus:**

- All validation steps included
- Clear error messages
- Doesn't deploy anything (validation only)

---

### 4.2 Deployment Workflow

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  `.github/workflows/deploy.yml`
- [ ]  Deploys to production on main branch
- [ ]  Manual approval gate (optional)
- [ ]  Deployment status notifications

**Workflow structure:**

yaml

`Jobs:
1. validate (reuse validation workflow)
2. deploy-staging (if staging branch)
3. deploy-production (if main branch)
   - Runs Ansible playbook
   - Reports success/failure
   - Cleanup step`

**Acceptance Criteria:**

- Only runs on main branch push
- Calls Ansible playbook correctly
- Handles failures gracefully
- Logs are clear and useful

**Reviewer Focus:**

- Proper use of self-hosted runner
- Environment secrets handled correctly
- Error handling comprehensive
- Deployment is atomic (all or nothing)

---

## Phase 5: Automation & Maintenance (Week 4)

### 5.1 Automated Cleanup

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  `playbooks/cleanup.yml` - Docker resource cleanup
- [ ]  Systemd timer for weekly cleanup
- [ ]  Integration into deploy workflow
- [ ]  Cleanup logs and reporting

**Cleanup tasks:**

yaml

- `Remove stopped containers
- Remove unused images (with filters)
- Remove unused networks
- Remove unused volumes (with caution)
- Prune build cache
- Log cleanup actions`

**Acceptance Criteria:**

- Safe cleanup (preserves active resources)
- Runs weekly automatically
- Can be triggered manually
- Logs what was removed

**Reviewer Focus:**

- Volume pruning is safe (uses labels/filters)
- No accidental removal of important data
- Proper logging for audit trail

---

### 5.2 Monitoring & Alerting Setup

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  Add monitoring service (Prometheus + Grafana or similar)
- [ ]  Deploy as part of services
- [ ]  Basic dashboards for container health
- [ ]  Alert on deployment failures

**Acceptance Criteria:**

- Monitoring stack deployed via same pipeline
- Dashboards accessible
- Basic alerts configured
- Documentation for adding new alerts

**Reviewer Focus:**

- Monitoring doesn't impact performance
- Dashboards are useful
- Alerts are actionable

---

## Phase 6: Documentation & Handoff (Week 4)

### 6.1 Complete Documentation

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  `docs/SETUP.md` - Initial setup guide
- [ ]  `docs/DEPLOYMENT.md` - How to deploy changes
- [ ]  `docs/MAINTENANCE.md` - Ongoing maintenance
- [ ]  `docs/TROUBLESHOOTING.md` - Common issues
- [ ]  `docs/ARCHITECTURE.md` - System overview

**Documentation requirements:**

- Step-by-step instructions
- Screenshots/diagrams where helpful
- Troubleshooting for common errors
- Contact/support information

**Reviewer Focus:**

- Documentation is complete
- Instructions are clear
- Examples are accurate
- Links work

---

### 6.2 Disaster Recovery Plan

**Assigned to:** Developer Agent

**Reviewed by:** Reviewer Agent

**Deliverables:**

- [ ]  `docs/DISASTER_RECOVERY.md`
- [ ]  Backup strategy documented
- [ ]  Restore procedure tested
- [ ]  Recovery time objectives defined

**DR elements:**

markdown

`1. What to backup (volumes, configs)
2. How to backup (scripts/tools)
3. Backup schedule
4. Restore procedure
5. Testing schedule
6. RTO/RPO targets
```

**Reviewer Focus:**
- Backup strategy is complete
- Restore procedure actually works
- Critical data identified

---

## Deliverables Summary

### Code Deliverables
```
├── ansible/
│   ├── playbooks/
│   │   ├── hardening.yml
│   │   ├── docker-setup.yml
│   │   ├── deploy-services.yml
│   │   └── cleanup.yml
│   ├── roles/
│   │   ├── security/
│   │   ├── docker/
│   │   └── docker-service/
│   └── inventory/
│       ├── production
│       └── staging
├── services/
│   ├── service1/
│   ├── service2/
│   └── ... (10-15 services)
├── .github/
│   └── workflows/
│       ├── validate.yml
│       └── deploy.yml
└── docs/
    ├── SETUP.md
    ├── DEPLOYMENT.md
    ├── MAINTENANCE.md
    ├── TROUBLESHOOTING.md
    ├── ARCHITECTURE.md
    ├── DISASTER_RECOVERY.md
    └── services.md`

### Testing Requirements

- [ ]  All playbooks run successfully on clean system
- [ ]  All services deploy and start correctly
- [ ]  CI/CD pipeline executes end-to-end
- [ ]  Rollback procedure tested
- [ ]  Cleanup automation tested

---

## Developer Agent Instructions

For each task:

1. **Read the requirements carefully**
2. **Create the deliverable** (playbook, workflow, documentation)
3. **Test locally** if possible
4. **Add inline comments** explaining complex logic
5. **Follow best practices**:
    - Ansible: Use roles, variables, handlers
    - GitHub Actions: Use reusable workflows
    - Docker Compose: Use labels, health checks
6. **Submit for review** with:
    - What was implemented
    - How to test it
    - Any assumptions made
    - Known limitations

---

## Reviewer Agent Instructions

For each review:

1. **Check against acceptance criteria**
2. **Verify best practices**:
    - Security considerations
    - Idempotency
    - Error handling
    - Documentation
3. **Test execution** (if possible)
4. **Provide feedback**:
    - What's good
    - What needs improvement
    - Security concerns
    - Suggestions for optimization
5. **Approve or request changes**

---

## Success Criteria

### Technical

- ✅ All services running via Ansible deployment
- ✅ CI/CD pipeline fully functional
- ✅ Zero manual steps for deployment
- ✅ Automated cleanup working
- ✅ All playbooks idempotent

### Documentation

- ✅ Complete setup guide
- ✅ All services documented
- ✅ Troubleshooting guide complete
- ✅ Disaster recovery tested

### Quality

- ✅ No secrets in repository
- ✅ All code reviewed
- ✅ Security best practices followed
- ✅ System passes hardening checks