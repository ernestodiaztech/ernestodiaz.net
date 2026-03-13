---
draft: true
title: '35 - Git Server'
description: "Part 35 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 1
---

{{< badge "Ansible" >}}
{{< badge content="VS Code" color="blue" >}}
{{< badge content="Linux" color="red" >}}

## Part 35: Git Server — Self-Hosted Gitea

> *Every playbook, role, and collection in this guide has lived on GitHub. That works fine until the day the lab needs to run air-gapped, a webhook round-trip to GitHub adds latency to the CI/CD loop, or — most importantly — the automation that manages the infrastructure needs a Git server that it itself provisions and controls. Gitea closes that loop. It's lightweight, Docker-native, and has a full REST API that Ansible can drive. After this part, AWX pulls playbooks from a Git server that lives on the automation VM at 172.16.0.30, and GitHub becomes an optional upstream mirror rather than a dependency.*

---

## 35.1 — Architecture

Gitea runs as a Docker Compose stack on the automation VM alongside AWX. PostgreSQL is shared — AWX and Gitea each get their own database on the same PostgreSQL instance, which is more resource-efficient than running two separate database servers.

```
automation VM (172.16.0.30)
│
├── Docker: gitea
│   ├── gitea/gitea:latest        — Gitea app (port 3000 web, 2222 SSH)
│   └── Data volume: /opt/gitea/data
│
├── Docker: postgres               — shared PostgreSQL instance
│   ├── Database: gitea            — Gitea's database
│   ├── Database: awx              — AWX's database (Part 38+)
│   └── Data volume: /opt/postgres/data
│
└── Docker: nginx                  — reverse proxy (optional for TLS)
    └── Port 80/443 → gitea:3000

Access:
  Web UI:  http://172.16.0.30:3000
  SSH:     git@172.16.0.30:2222  (clone/push via SSH)
  API:     http://172.16.0.30:3000/api/v1

Mirror relationship:
  GitHub (upstream) ←──── Gitea pull mirror (every 8 hours)
  Gitea (primary for AWX) ←── AWX project sync
  Developer (local) ──push──► GitHub ──mirror──► Gitea ──webhook──► AWX
```

### Why mirror rather than migrate

```
Full migration:
  Pro:  Gitea becomes the single source of truth
  Con:  Lose GitHub Actions CI (until AWX fully replaces it)
        Teammates or future employers can't see public work on GitHub
        One more thing to back up as the only copy

Mirror (what this guide uses):
  Pro:  GitHub remains the canonical public remote — push there as normal
        Gitea mirrors every 8 hours (or on-demand) — AWX always has a
        local copy without a GitHub dependency at runtime
        If Gitea goes down, push to GitHub and the lab is unblocked
        The mirror relationship is explicit — no confusion about
        which remote is authoritative
  Con:  Slight lag between GitHub push and Gitea mirror update
        (solved with a webhook from GitHub → Gitea force-sync)
```

---

## 35.2 — Prerequisites on the Automation VM

```bash
# SSH to automation VM
ssh ansible@172.16.0.30

# Install Docker and Docker Compose
sudo apt update
sudo apt install -y docker.io docker-compose-plugin curl git

# Add ansible user to docker group (no sudo needed for docker commands)
sudo usermod -aG docker ansible
newgrp docker

# Verify
docker --version
docker compose version
```

---

## 35.3 — Directory and Compose File

```bash
# Create directory structure on automation VM
sudo mkdir -p /opt/gitea/{data,config}
sudo mkdir -p /opt/postgres/data
sudo chown -R ansible:ansible /opt/gitea /opt/postgres

# Create the Docker Compose file
mkdir -p ~/gitea
cat > ~/gitea/docker-compose.yml << 'EOF'
---
version: "3.8"

services:

  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: "${POSTGRES_MASTER_PASSWORD}"
      # Gitea database created via init script below
    volumes:
      - /opt/postgres/data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      USER_UID: "1000"
      USER_GID: "1000"
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: postgres:5432
      GITEA__database__NAME: gitea
      GITEA__database__USER: gitea
      GITEA__database__PASSWD: "${GITEA_DB_PASSWORD}"
      GITEA__server__DOMAIN: "172.16.0.30"
      GITEA__server__ROOT_URL: "http://172.16.0.30:3000/"
      GITEA__server__SSH_DOMAIN: "172.16.0.30"
      GITEA__server__SSH_PORT: "2222"
      GITEA__server__START_SSH_SERVER: "true"
      GITEA__security__INSTALL_LOCK: "true"
      GITEA__security__SECRET_KEY: "${GITEA_SECRET_KEY}"
      GITEA__security__INTERNAL_TOKEN: "${GITEA_INTERNAL_TOKEN}"
      GITEA__service__DISABLE_REGISTRATION: "true"
      GITEA__service__REQUIRE_SIGNIN_VIEW: "true"
      GITEA__log__LEVEL: "Info"
      GITEA__webhook__ALLOWED_HOST_LIST: "172.16.0.0/24,192.168.100.0/24"
      GITEA__cron__ENABLED: "true"
      GITEA__cron__RUN_AT_START: "true"
    volumes:
      - /opt/gitea/data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"    # Web UI
      - "2222:22"      # SSH (maps container 22 to host 2222)
    networks:
      - backend
      - frontend

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge
EOF
```

### PostgreSQL init script — creates gitea database and user

```bash
cat > ~/gitea/init-db.sql << 'EOF'
-- Create Gitea database user and database
-- AWX database will be added here in Part 38
CREATE USER gitea WITH PASSWORD '${GITEA_DB_PASSWORD}';
CREATE DATABASE gitea OWNER gitea ENCODING 'UTF8';
GRANT ALL PRIVILEGES ON DATABASE gitea TO gitea;

-- Create awx user now (placeholder — used in Part 38)
CREATE USER awx WITH PASSWORD '${AWX_DB_PASSWORD}';
CREATE DATABASE awx OWNER awx ENCODING 'UTF8';
GRANT ALL PRIVILEGES ON DATABASE awx TO awx;
EOF
```

### ### ⚠️ Warning
> The `init-db.sql` script only runs when PostgreSQL initialises for the first time (empty `/opt/postgres/data` directory). If PostgreSQL has already started once without this script, you'll need to create the databases manually with `psql` or destroy the data volume and start fresh.

### Environment file — secrets

```bash
# Generate strong secrets
POSTGRES_MASTER_PASSWORD=$(openssl rand -base64 32)
GITEA_DB_PASSWORD=$(openssl rand -base64 32)
GITEA_SECRET_KEY=$(openssl rand -base64 64 | tr -d '\n')
GITEA_INTERNAL_TOKEN=$(openssl rand -base64 64 | tr -d '\n')
AWX_DB_PASSWORD=$(openssl rand -base64 32)

cat > ~/gitea/.env << EOF
POSTGRES_MASTER_PASSWORD=${POSTGRES_MASTER_PASSWORD}
GITEA_DB_PASSWORD=${GITEA_DB_PASSWORD}
GITEA_SECRET_KEY=${GITEA_SECRET_KEY}
GITEA_INTERNAL_TOKEN=${GITEA_INTERNAL_TOKEN}
AWX_DB_PASSWORD=${AWX_DB_PASSWORD}
EOF

chmod 600 ~/gitea/.env

# Save these to Ansible vault immediately — they're needed in later parts
# ansible-vault edit inventory/group_vars/all/vault.yml
# Add:
# vault_postgres_master_password: "<value>"
# vault_gitea_db_password: "<value>"
# vault_gitea_secret_key: "<value>"
# vault_gitea_internal_token: "<value>"
# vault_awx_db_password: "<value>"
echo "Secrets generated and saved to ~/gitea/.env"
echo "Add these to Ansible vault before proceeding"
```

---

## 35.4 — Ansible Playbook: Deploy Gitea

```bash
# On the control node / automation VM
cat > ~/projects/ansible-network/playbooks/infrastructure/gitea/deploy.yml << 'EOF'
---
# Deploy Gitea + PostgreSQL on the automation VM
# Usage: ansible-playbook playbooks/infrastructure/gitea/deploy.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Gitea | Deploy Gitea and PostgreSQL"
  hosts: automation_hosts
  gather_facts: true
  become: true
  tags: [gitea, deploy]

  vars:
    gitea_dir: /home/ansible/gitea
    gitea_data_dir: /opt/gitea/data
    postgres_data_dir: /opt/postgres/data

  tasks:

    # ── Directories ───────────────────────────────────────────────────
    - name: "Dirs | Create Gitea and PostgreSQL data directories"
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: ansible
        group: ansible
        mode: '0755'
      loop:
        - "{{ gitea_dir }}"
        - "{{ gitea_data_dir }}"
        - "{{ postgres_data_dir }}"

    # ── Docker Compose files ──────────────────────────────────────────
    - name: "Compose | Deploy docker-compose.yml"
      ansible.builtin.template:
        src: docker-compose.yml.j2
        dest: "{{ gitea_dir }}/docker-compose.yml"
        owner: ansible
        group: ansible
        mode: '0644'

    - name: "Compose | Deploy PostgreSQL init script"
      ansible.builtin.template:
        src: init-db.sql.j2
        dest: "{{ gitea_dir }}/init-db.sql"
        owner: ansible
        group: ansible
        mode: '0600'

    - name: "Compose | Deploy environment file"
      ansible.builtin.template:
        src: gitea.env.j2
        dest: "{{ gitea_dir }}/.env"
        owner: ansible
        group: ansible
        mode: '0600'

    # ── UFW rules for Gitea ───────────────────────────────────────────
    - name: "UFW | Allow Gitea web UI from management network"
      community.general.ufw:
        rule: allow
        port: "3000"
        proto: tcp
        src: "172.16.0.0/24"
        comment: "Gitea web UI"

    - name: "UFW | Allow Gitea SSH from management network"
      community.general.ufw:
        rule: allow
        port: "2222"
        proto: tcp
        src: "172.16.0.0/24"
        comment: "Gitea SSH"

    # ── Pull images and start stack ───────────────────────────────────
    - name: "Docker | Pull latest images"
      community.docker.docker_compose_v2:
        project_src: "{{ gitea_dir }}"
        pull: always
        state: present
      become: false

    - name: "Docker | Start Gitea stack"
      community.docker.docker_compose_v2:
        project_src: "{{ gitea_dir }}"
        state: present
      become: false
      register: compose_result

    # ── Wait for Gitea to be ready ────────────────────────────────────
    - name: "Gitea | Wait for web UI to respond"
      ansible.builtin.uri:
        url: "http://172.16.0.30:3000"
        status_code: 200
        timeout: 10
      register: gitea_check
      retries: 18
      delay: 10
      until: gitea_check.status == 200
      delegate_to: localhost

    - name: "Gitea | Confirm stack is running"
      ansible.builtin.debug:
        msg:
          - "════════════════════════════════════════"
          - " Gitea is running"
          - " Web UI: http://172.16.0.30:3000"
          - " SSH:    git@172.16.0.30 (port 2222)"
          - " API:    http://172.16.0.30:3000/api/v1"
          - "════════════════════════════════════════"
EOF
```

### Templates

```bash
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/gitea/templates

# docker-compose.yml.j2 — same as above but with vault variables
cat > ~/projects/ansible-network/playbooks/infrastructure/gitea/templates/gitea.env.j2 << 'EOF'
# Managed by Ansible — do not edit manually
POSTGRES_MASTER_PASSWORD={{ vault_postgres_master_password }}
GITEA_DB_PASSWORD={{ vault_gitea_db_password }}
GITEA_SECRET_KEY={{ vault_gitea_secret_key }}
GITEA_INTERNAL_TOKEN={{ vault_gitea_internal_token }}
AWX_DB_PASSWORD={{ vault_awx_db_password }}
EOF
```

### Run the deployment

```bash
# Snapshot automation VM before deploying
ansible-playbook \
  playbooks/infrastructure/lifecycle/snapshot.yml \
  -e "target_vmid=103 snap_name=pre-gitea \
      snap_description='Before Gitea install Part 35'" \
  --vault-id lab@.vault/lab.txt

# Deploy Gitea
ansible-playbook \
  playbooks/infrastructure/gitea/deploy.yml \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt

# Verify containers are running
ssh ansible@172.16.0.30 "docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"
# Expected:
# NAMES     STATUS          PORTS
# gitea     Up 2 minutes    0.0.0.0:2222->22/tcp, 0.0.0.0:3000->3000/tcp
# postgres  Up 2 minutes    5432/tcp
```

---

## 35.5 — Initial Gitea Configuration

INSTALL_LOCK=true in the environment file skips the web installer, but the admin user and initial organisation still need to be created via the Gitea CLI inside the container.

```bash
# SSH to automation VM
ssh ansible@172.16.0.30

# Create the admin user inside the Gitea container
docker exec -u git gitea \
  gitea admin user create \
  --username gitea-admin \
  --password "$(cat ~/gitea/.env | grep -v AWX | openssl rand -base64 16)" \
  --email "admin@lab.local" \
  --admin \
  --must-change-password=false

# Better approach — use a known password stored in vault
# Run this from the control node once vault_gitea_admin_password is set:
docker exec -u git gitea \
  gitea admin user create \
  --username gitea-admin \
  --password "GitAdmin#Lab2025" \
  --email "admin@lab.local" \
  --admin \
  --must-change-password=false

# Create an organisation for the lab
# (organisations group repos — mirrors enterprise Gitea/GitHub org structure)
curl -s -X POST \
  -H "Content-Type: application/json" \
  -u "gitea-admin:GitAdmin#Lab2025" \
  "http://172.16.0.30:3000/api/v1/orgs" \
  -d '{
    "username": "network-automation",
    "full_name": "Network Automation Lab",
    "description": "Ansible network automation playbooks and roles",
    "visibility": "private"
  }' | python3 -m json.tool

# Verify organisation created
curl -s \
  -u "gitea-admin:GitAdmin#Lab2025" \
  "http://172.16.0.30:3000/api/v1/orgs/network-automation" \
  | python3 -m json.tool | grep -E "name|full_name|visibility"
```

### Create an Ansible API token in Gitea

Ansible uses this token to create repos, configure mirrors, and manage webhooks via the Gitea API — not the admin password.

```bash
# Create API token for Ansible automation
GITEA_ANSIBLE_TOKEN=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -u "gitea-admin:GitAdmin#Lab2025" \
  "http://172.16.0.30:3000/api/v1/users/gitea-admin/tokens" \
  -d '{
    "name": "ansible-automation",
    "scopes": ["write:repository","write:user","write:organization","write:admin"]
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['sha1'])")

echo "Gitea Ansible token: ${GITEA_ANSIBLE_TOKEN}"
# Add this to Ansible vault as vault_gitea_ansible_token
```

---

## 35.6 — SSH Key Management

Two SSH key relationships need configuring: Gitea needs a deploy key to pull from GitHub, and the automation VM needs to push to Gitea via SSH.

```bash
# On automation VM — generate a dedicated key for Gitea → GitHub mirroring
ssh-keygen -t ed25519 \
  -C "gitea-mirror@lab.local" \
  -f ~/.ssh/gitea_github_mirror \
  -N ""

# Print the public key — add this to GitHub as a deploy key
cat ~/.ssh/gitea_github_mirror.pub
# → Add to: GitHub repo → Settings → Deploy keys → Add deploy key
#   Title: "Gitea Mirror Key"
#   Allow write access: NO (read-only mirror)

# On control node / developer workstation — generate key for pushing to Gitea
ssh-keygen -t ed25519 \
  -C "ansible-dev@lab.local" \
  -f ~/.ssh/id_gitea \
  -N ""

# Add public key to Gitea account
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: token ${GITEA_ANSIBLE_TOKEN}" \
  "http://172.16.0.30:3000/api/v1/user/keys" \
  -d "{
    \"title\": \"Ansible Control Node\",
    \"key\": \"$(cat ~/.ssh/id_gitea.pub)\",
    \"read_only\": false
  }" | python3 -m json.tool | grep -E "id|title|key"

# Update SSH config to use port 2222 for Gitea
cat >> ~/.ssh/config << 'EOF'

Host gitea-lab
  HostName 172.16.0.30
  Port 2222
  User git
  IdentityFile ~/.ssh/id_gitea
  StrictHostKeyChecking no
EOF

# Test SSH connection to Gitea
ssh -T gitea-lab
# Expected: Hi gitea-admin! You've successfully authenticated...
```

---

## 35.7 — Mirror the ansible-network Repo from GitHub

Gitea's push mirror feature pulls from GitHub on a schedule and keeps Gitea in sync. AWX always pulls from Gitea — no GitHub dependency at runtime.

### Create the mirrored repository via Ansible

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/gitea/mirror_repo.yml << 'EOF'
---
# Create a GitHub → Gitea mirror for the ansible-network repo
# Usage: ansible-playbook playbooks/infrastructure/gitea/mirror_repo.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Gitea | Create GitHub mirror"
  hosts: localhost
  gather_facts: false
  tags: [gitea, mirror]

  vars:
    gitea_url: "http://172.16.0.30:3000"
    gitea_token: "{{ vault_gitea_ansible_token }}"
    gitea_org: "network-automation"
    github_repo_url: "https://github.com/YOUR-USERNAME/ansible-network.git"
    # For private GitHub repos, use a GitHub PAT:
    # github_clone_url: "https://YOUR-TOKEN@github.com/YOUR-USERNAME/ansible-network.git"
    mirror_interval: "8h0m0s"    # Sync every 8 hours

  tasks:

    - name: "Mirror | Check if repo already exists in Gitea"
      ansible.builtin.uri:
        url: "{{ gitea_url }}/api/v1/repos/{{ gitea_org }}/ansible-network"
        method: GET
        headers:
          Authorization: "token {{ gitea_token }}"
        status_code: [200, 404]
      register: repo_check

    - name: "Mirror | Create mirrored repo from GitHub"
      ansible.builtin.uri:
        url: "{{ gitea_url }}/api/v1/repos/migrate"
        method: POST
        headers:
          Authorization: "token {{ gitea_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          clone_addr: "{{ github_repo_url }}"
          repo_name: "ansible-network"
          repo_owner: "{{ gitea_org }}"
          mirror: true
          mirror_interval: "{{ mirror_interval }}"
          private: true
          description: "Ansible network automation — mirror of GitHub"
          wiki: false
          issues: false
          pull_requests: false
          releases: false
          # auth_token: "{{ vault_github_pat }}"  # Uncomment for private repos
        status_code: 201
      when: repo_check.status == 404
      register: mirror_result

    - name: "Mirror | Trigger immediate sync"
      ansible.builtin.uri:
        url: "{{ gitea_url }}/api/v1/repos/{{ gitea_org }}/ansible-network/mirror-sync"
        method: POST
        headers:
          Authorization: "token {{ gitea_token }}"
        status_code: 200
      when: repo_check.status == 404

    - name: "Mirror | Verify repo is accessible"
      ansible.builtin.uri:
        url: "{{ gitea_url }}/api/v1/repos/{{ gitea_org }}/ansible-network"
        method: GET
        headers:
          Authorization: "token {{ gitea_token }}"
      register: repo_info

    - name: "Mirror | Report result"
      ansible.builtin.debug:
        msg:
          - "════════════════════════════════════════════"
          - " Mirror configured"
          - " Repo:     {{ repo_info.json.full_name }}"
          - " Mirror:   {{ repo_info.json.mirror }}"
          - " Interval: {{ mirror_interval }}"
          - " Clone:    {{ repo_info.json.clone_url }}"
          - " SSH:      {{ repo_info.json.ssh_url }}"
          - "════════════════════════════════════════════"
EOF
```

### Run the mirror playbook

```bash
ansible-playbook \
  playbooks/infrastructure/gitea/mirror_repo.yml \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt

# Verify the mirror in Gitea web UI
# → http://172.16.0.30:3000/network-automation/ansible-network
# → Settings → Mirror Settings — shows last sync time and interval

# Force a manual sync via API
curl -s -X POST \
  -H "Authorization: token ${GITEA_ANSIBLE_TOKEN}" \
  "http://172.16.0.30:3000/api/v1/repos/network-automation/ansible-network/mirror-sync"

# Check repo branches match GitHub
curl -s \
  -H "Authorization: token ${GITEA_ANSIBLE_TOKEN}" \
  "http://172.16.0.30:3000/api/v1/repos/network-automation/ansible-network/branches" \
  | python3 -m json.tool | grep '"name"'
```

### Instant sync via GitHub webhook (optional but recommended)

Rather than waiting up to 8 hours for the scheduled sync, a GitHub webhook tells Gitea to sync immediately on every push.

```
On GitHub:
  Repo → Settings → Webhooks → Add webhook
    Payload URL:  http://172.16.0.30:3000/network-automation/ansible-network/mirror-sync
    Content type: application/json
    Secret:       (leave blank — Gitea mirror-sync endpoint is internal-only)
    Events:       Just the push event
    Active:       ✓

Result: Push to GitHub → GitHub fires webhook → Gitea syncs within seconds
        → AWX sees the latest commit the next time it checks
```

---

## 35.8 — Configure AWX to Use Gitea (Brief)

AWX needs to know where Gitea lives and how to authenticate. Full AWX setup is in Part 38 — this section shows just the credential and project configuration so AWX can start using Gitea now.

### Create a Gitea credential in AWX

```bash
# Via AWX CLI (awx must already be installed and configured — see Part 38)
# Shown here for completeness; run after AWX is deployed

awx credentials create \
  --name "Gitea — ansible-network" \
  --credential_type "Source Control" \
  --inputs '{
    "username": "gitea-admin",
    "ssh_key_data": "'"$(cat ~/.ssh/id_gitea)"'",
    "ssh_key_unlock": ""
  }'

# Alternative: HTTP token credential
awx credentials create \
  --name "Gitea — ansible-network (token)" \
  --credential_type "Source Control" \
  --inputs '{
    "username": "gitea-admin",
    "password": "'"${GITEA_ANSIBLE_TOKEN}"'"
  }'
```

### AWX project pointing to Gitea

```bash
awx projects create \
  --name "ansible-network (Gitea)" \
  --scm_type git \
  --scm_url "http://172.16.0.30:3000/network-automation/ansible-network.git" \
  --scm_branch main \
  --scm_update_on_launch true \
  --credential "Gitea — ansible-network (token)"
```

### ### ℹ️ Info
> `scm_update_on_launch: true` means AWX syncs the project from Gitea every time a job template runs. Since Gitea mirrors GitHub every 8 hours (or on push via webhook), AWX always runs against the latest code without depending on GitHub directly. This is the key benefit of the mirror architecture.

---

## 35.9 — Gitea Webhook → AWX (Brief)

A webhook from Gitea to AWX triggers an AWX project sync (and optionally a job template run) the moment a push reaches Gitea — either from a direct push or from a GitHub mirror sync completing.

```bash
# Create a webhook on the Gitea repo pointing to AWX
# Replace AWX_TOKEN and JOB_TEMPLATE_ID after AWX is deployed in Part 38

GITEA_WEBHOOK_PAYLOAD='{
  "type": "gitea",
  "config": {
    "url": "http://172.16.0.30/api/v2/job_templates/JOB_TEMPLATE_ID/launch/",
    "content_type": "json",
    "secret": "AWX_WEBHOOK_SECRET"
  },
  "events": ["push"],
  "branch_filter": "main",
  "active": true
}'

curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: token ${GITEA_ANSIBLE_TOKEN}" \
  "http://172.16.0.30:3000/api/v1/repos/network-automation/ansible-network/hooks" \
  -d "${GITEA_WEBHOOK_PAYLOAD}" \
  | python3 -m json.tool | grep -E "id|type|active"
```

### The full push-to-deploy chain

```
Developer workstation
  │
  └── git push origin main
        │
        ▼
      GitHub (canonical remote)
        │
        └── Webhook → Gitea force-sync (immediate)
              │
              ▼
            Gitea (172.16.0.30:3000)
              │
              └── Webhook → AWX project sync + job launch
                    │
                    ▼
                  AWX (172.16.0.30:443)
                    │
                    └── Runs playbook against network devices
                          │
                          ▼
                        Containerlab fabric
                        (172.16.0.11–.51)

Total time from git push to AWX job running: < 60 seconds
```

### ### 💡 Tip
> AWX webhook integration requires AWX to be configured with a webhook credential and the job template to have webhook triggering enabled. These steps are covered fully in Part 38. For now, the Gitea webhook can be created pointing at the correct URL — it will simply fail silently until AWX is deployed and the job template is configured to accept it.

---

## 35.10 — Ongoing Gitea Management Playbook

For day-to-day Gitea operations — creating new repos, adding users, rotating tokens — an operations playbook wraps the Gitea API.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/gitea/manage.yml << 'EOF'
---
# Day-to-day Gitea management operations
# Usage examples:
#   Create a repo:
#   ansible-playbook gitea/manage.yml -e "action=create_repo repo_name=my-new-repo"
#
#   Force mirror sync:
#   ansible-playbook gitea/manage.yml -e "action=sync_mirror repo_name=ansible-network"
#
#   List all repos:
#   ansible-playbook gitea/manage.yml -e "action=list_repos"

- name: "Gitea | Management operations"
  hosts: localhost
  gather_facts: false

  vars:
    gitea_url: "http://172.16.0.30:3000"
    gitea_token: "{{ vault_gitea_ansible_token }}"
    gitea_org: "network-automation"
    action: ~
    repo_name: ~

  tasks:

    - name: "Assert action is provided"
      ansible.builtin.assert:
        that: action is not none
        fail_msg: "Provide -e 'action=<create_repo|sync_mirror|list_repos>'"

    # ── List repos ────────────────────────────────────────────────────
    - name: "List repos in {{ gitea_org }}"
      ansible.builtin.uri:
        url: "{{ gitea_url }}/api/v1/orgs/{{ gitea_org }}/repos?limit=50"
        headers:
          Authorization: "token {{ gitea_token }}"
      register: repos
      when: action == 'list_repos'

    - name: "Display repos"
      ansible.builtin.debug:
        msg: "{{ repos.json | map(attribute='name') | list }}"
      when: action == 'list_repos'

    # ── Create repo ───────────────────────────────────────────────────
    - name: "Create repo {{ repo_name }}"
      ansible.builtin.uri:
        url: "{{ gitea_url }}/api/v1/orgs/{{ gitea_org }}/repos"
        method: POST
        headers:
          Authorization: "token {{ gitea_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          name: "{{ repo_name }}"
          private: true
          auto_init: true
          default_branch: main
        status_code: 201
      when: action == 'create_repo' and repo_name is not none

    # ── Force mirror sync ─────────────────────────────────────────────
    - name: "Force sync mirror {{ repo_name }}"
      ansible.builtin.uri:
        url: "{{ gitea_url }}/api/v1/repos/{{ gitea_org }}/{{ repo_name }}/mirror-sync"
        method: POST
        headers:
          Authorization: "token {{ gitea_token }}"
        status_code: 200
      when: action == 'sync_mirror' and repo_name is not none

    - name: "Sync triggered"
      ansible.builtin.debug:
        msg: "Mirror sync triggered for {{ gitea_org }}/{{ repo_name }}"
      when: action == 'sync_mirror'
EOF
```

---

## 35.11 — Checklist

```
[ ] Prerequisites on automation VM:
    [ ] Docker and docker-compose-plugin installed
    [ ] ansible user in docker group
    [ ] /opt/gitea/data and /opt/postgres/data created

[ ] Gitea stack deployed:
    [ ] ~/gitea/docker-compose.yml created
    [ ] ~/gitea/init-db.sql created (gitea + awx databases)
    [ ] ~/gitea/.env created with generated secrets
    [ ] Secrets added to Ansible vault
    [ ] docker compose up succeeds — both containers healthy
    [ ] http://172.16.0.30:3000 returns Gitea web UI

[ ] Initial configuration:
    [ ] gitea-admin user created (via docker exec)
    [ ] network-automation organisation created
    [ ] Ansible API token created — stored in vault as vault_gitea_ansible_token
    [ ] API token test: curl returns HTTP 200 with user info

[ ] SSH key management:
    [ ] gitea_github_mirror key generated — public key added to GitHub as deploy key
    [ ] id_gitea key generated — public key added to Gitea via API
    [ ] ~/.ssh/config entry for gitea-lab host (port 2222)
    [ ] ssh -T gitea-lab returns successful authentication message

[ ] Mirror configured:
    [ ] mirror_repo.yml playbook runs successfully
    [ ] ansible-network repo appears in Gitea at
        http://172.16.0.30:3000/network-automation/ansible-network
    [ ] Repo shows mirror=true in API response
    [ ] Branches in Gitea match branches in GitHub
    [ ] Force sync via API returns HTTP 200
    [ ] (Optional) GitHub webhook configured for instant sync

[ ] AWX integration (partial — completes in Part 38):
    [ ] Gitea webhook created on ansible-network repo
        (will fail until AWX deployed — expected)
    [ ] AWX project URL noted:
        http://172.16.0.30:3000/network-automation/ansible-network.git

[ ] Ansible management playbooks committed:
    [ ] playbooks/infrastructure/gitea/deploy.yml
    [ ] playbooks/infrastructure/gitea/mirror_repo.yml
    [ ] playbooks/infrastructure/gitea/manage.yml
    [ ] git add + git commit + git push to GitHub
    [ ] Verify mirror sync brings the new playbooks into Gitea

[ ] Snapshot taken:
    ansible-playbook lifecycle/snapshot.yml \
      -e "target_vmid=103 snap_name=post-gitea \
          snap_description='Gitea running and mirroring Part 35'"

[ ] UFW rules verified:
    [ ] Port 3000 accessible from 172.16.0.0/24
    [ ] Port 2222 accessible from 172.16.0.0/24
    [ ] Port 5432 NOT accessible externally (PostgreSQL internal only)
```

---

*Gitea is running, the ansible-network repo is mirrored locally, and the foundation for a fully self-contained CI/CD loop is in place. Every push to GitHub now flows into Gitea within seconds, and AWX will pull from Gitea rather than GitHub when it's deployed in Part 38. The lab no longer has a runtime dependency on an external service.*

*Next up: **Part 36 — Netbox: Production-Grade Deployment and Stack Integration***
