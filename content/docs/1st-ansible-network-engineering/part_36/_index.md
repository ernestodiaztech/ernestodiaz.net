---
draft: true
title: '36 - Netbox'
description: "Part 36 of my Ansible learning geared towards Network Engineering."
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

## Part 36: Netbox — Production-Grade Deployment and Stack Integration

> *Part 26 covered Netbox as a source of truth for network device data — what it stores, how Ansible queries it, how to populate it. That was Netbox as a tool. This part covers Netbox as a service — deployed properly on its own VM, backed up automatically to Git, integrated with the monitoring stack so its health is visible, and wired to AWX so a device change in Netbox triggers an Ansible playbook without anyone typing a command. The difference between Part 26 and Part 36 is the difference between knowing how to use a tool and knowing how to run one.*

---

## 36.1 — PostgreSQL: Dedicated vs Shared

Before deploying anything, the database question needs a clear answer because it affects architecture across the whole stack.

### Option comparison

```
Option A: Dedicated PostgreSQL on the netbox VM (172.16.0.20)
  ┌─────────────────────────┐
  │      netbox VM          │
  │  Netbox app             │
  │  PostgreSQL (netbox DB) │
  │  Redis                  │
  └─────────────────────────┘

  Pro:  Complete isolation — netbox VM is self-contained
        Snapshot the VM = snapshot the database
        No cross-VM dependency for Netbox to function
        Simpler firewall rules (no inter-VM DB traffic)
  Con:  Another PostgreSQL instance to patch and monitor
        Wastes RAM on a 4GB VM running two PostgreSQL instances
        (one per VM that needs a DB)

Option B: Shared PostgreSQL on automation VM (172.16.0.30)
  ┌─────────────────────────┐    ┌──────────────────────────┐
  │      netbox VM          │    │    automation VM         │
  │  Netbox app             │───►│  PostgreSQL              │
  │  Redis                  │    │    DB: gitea             │
  └─────────────────────────┘    │    DB: awx               │
                                 │    DB: netbox  ← new     │
                                 └──────────────────────────┘

  Pro:  One PostgreSQL instance to manage, patch, back up
        More RAM available on the netbox VM for Netbox itself
        Consistent database management across all services
  Con:  Cross-VM network dependency — netbox VM needs port 5432
        on automation VM
        Automation VM outage takes down Netbox database
        Snapshot of netbox VM doesn't include its database

Recommendation: Dedicated PostgreSQL on the netbox VM.

Rationale: The netbox VM has 4GB RAM — PostgreSQL for a small lab
database uses < 200MB. The isolation benefit outweighs the minor
RAM overhead. More importantly, snapshotting the netbox VM captures
the complete Netbox state (app + data) in a single operation, which
is exactly what you want before a Netbox version upgrade. The shared
PostgreSQL approach makes more sense when VMs are RAM-constrained
(16GB total) — at 64GB it's unnecessary.

This part uses Option A: dedicated PostgreSQL on the netbox VM.
```

---

## 36.2 — Architecture

```
netbox VM (172.16.0.20 / 192.168.100.20)
│
├── Docker: netbox          — Netbox app (gunicorn, port 8080)
├── Docker: netbox-worker   — Netbox background task worker
├── Docker: netbox-housekeeping — Scheduled cleanup tasks
├── Docker: postgres        — PostgreSQL 16 (dedicated, port 5432 internal)
├── Docker: redis           — Redis cache + task queue (port 6379 internal)
└── Docker: nginx           — Reverse proxy (port 80 external → netbox:8080)

Access:
  Web UI:  http://172.16.0.20       (nginx → Netbox)
  API:     http://172.16.0.20/api/
  GraphQL: http://172.16.0.20/graphql/

Integration points:
  Ansible  →  Netbox API  (dynamic inventory, device queries)
  Netbox   →  AWX webhook (device change triggers playbook)
  Ansible  →  Netbox backup → Gitea (scheduled export + commit)
  Zabbix   →  Netbox HTTP check (health monitoring — Part 37)
  Prometheus → Netbox exporter (metrics — Part 38)
```

---

## 36.3 — Deploy Netbox with Docker Compose

Netbox's official Docker project (`netbox-community/netbox-docker`) is the most reliable way to run Netbox in Docker — it handles the multi-container setup, initialisation, and upgrade paths correctly.

```bash
# SSH to netbox VM
ssh ansible@172.16.0.20

# Install Docker
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker ansible
newgrp docker

# Clone the official netbox-docker project
sudo mkdir -p /opt/netbox
sudo chown ansible:ansible /opt/netbox
git clone https://github.com/netbox-community/netbox-docker.git /opt/netbox
cd /opt/netbox

# Pin to a specific release (check latest at github.com/netbox-community/netbox-docker/releases)
git checkout 3.1.3
```

### Override compose file

The netbox-docker project uses a `docker-compose.yml` and an override file. The override sets lab-specific values without modifying the upstream file — making future upgrades a `git pull` away.

```bash
cat > /opt/netbox/docker-compose.override.yml << 'EOF'
---
version: "3.4"

services:

  netbox:
    ports:
      - "8080:8080"    # Expose to nginx on same host
    environment:
      SKIP_SUPERUSER: "false"
      SUPERUSER_NAME: "admin"
      SUPERUSER_EMAIL: "admin@lab.local"
      SUPERUSER_PASSWORD: "${NETBOX_SUPERUSER_PASSWORD}"
      SUPERUSER_API_TOKEN: "${NETBOX_API_TOKEN}"
    healthcheck:
      test: curl -f http://localhost:8080/api/ || exit 1
      interval: 30s
      timeout: 10s
      retries: 5

  postgres:
    environment:
      POSTGRES_PASSWORD: "${NETBOX_DB_PASSWORD}"

  redis:
    command: redis-server --requirepass "${REDIS_PASSWORD}"

  redis-cache:
    command: redis-server --requirepass "${REDIS_CACHE_PASSWORD}"

  nginx:
    ports:
      - "80:8080"
    image: nginx:alpine
    depends_on:
      - netbox
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - netbox-media-files:/opt/netbox/netbox/media:ro
    restart: unless-stopped

volumes:
  netbox-media-files:
    driver: local
EOF
```

### Nginx config

```bash
cat > /opt/netbox/nginx.conf << 'EOF'
# Managed by Ansible — do not edit manually
server {
    listen 8080;
    server_name 172.16.0.20;

    client_max_body_size 25m;

    location /static/ {
        alias /opt/netbox/netbox/static/;
    }

    location / {
        proxy_pass http://netbox:8080;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

### Environment file

```bash
# Generate secrets
NETBOX_DB_PASSWORD=$(openssl rand -base64 32)
REDIS_PASSWORD=$(openssl rand -base64 32)
REDIS_CACHE_PASSWORD=$(openssl rand -base64 32)
NETBOX_SECRET_KEY=$(openssl rand -base64 64 | tr -d '\n')
NETBOX_SUPERUSER_PASSWORD=$(openssl rand -base64 16)
NETBOX_API_TOKEN=$(openssl rand -hex 20)

cat > /opt/netbox/.env << EOF
NETBOX_DB_PASSWORD=${NETBOX_DB_PASSWORD}
REDIS_PASSWORD=${REDIS_PASSWORD}
REDIS_CACHE_PASSWORD=${REDIS_CACHE_PASSWORD}
SECRET_KEY=${NETBOX_SECRET_KEY}
NETBOX_SUPERUSER_PASSWORD=${NETBOX_SUPERUSER_PASSWORD}
NETBOX_API_TOKEN=${NETBOX_API_TOKEN}
EOF
chmod 600 /opt/netbox/.env

echo "API token: ${NETBOX_API_TOKEN}"
echo "Admin password: ${NETBOX_SUPERUSER_PASSWORD}"
echo "Add both to Ansible vault as vault_netbox_api_token and vault_netbox_admin_password"
```

### Start Netbox

```bash
cd /opt/netbox

# Pull all images
docker compose pull

# Start the stack (first start takes 2-3 minutes — database initialisation)
docker compose up -d

# Watch initialisation logs — wait for "Starting development server"
docker compose logs -f netbox

# Verify all containers are healthy
docker compose ps
# Expected output — all containers should show "healthy" or "running":
# NAME                    STATUS
# netbox-netbox-1         Up (healthy)
# netbox-netbox-worker-1  Up
# netbox-netbox-housekeeping-1  Up
# netbox-postgres-1       Up (healthy)
# netbox-redis-1          Up
# netbox-redis-cache-1    Up
# netbox-nginx-1          Up

# Test API is responding
curl -s http://172.16.0.20/api/ | python3 -m json.tool | head -5
```

---

## 36.4 — Ansible Playbook: Deploy Netbox

This playbook runs from the control node and deploys the full Netbox stack to the netbox VM — idempotent, vault-driven, snapshotted before running.

```bash
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/netbox/templates

cat > ~/projects/ansible-network/playbooks/infrastructure/netbox/deploy.yml << 'EOF'
---
# Deploy Netbox on the netbox VM
# Usage: ansible-playbook playbooks/infrastructure/netbox/deploy.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Netbox | Deploy Netbox stack"
  hosts: netbox_hosts
  gather_facts: true
  become: true
  tags: [netbox, deploy]

  vars:
    netbox_dir: /opt/netbox
    netbox_version: "3.1.3"    # Pin to tested release

  tasks:

    # ── Docker ────────────────────────────────────────────────────────
    - name: "Docker | Install Docker"
      ansible.builtin.apt:
        name:
          - docker.io
          - docker-compose-plugin
        state: present
        update_cache: true

    - name: "Docker | Add ansible user to docker group"
      ansible.builtin.user:
        name: ansible
        groups: docker
        append: true

    # ── Clone netbox-docker ───────────────────────────────────────────
    - name: "Netbox | Clone netbox-docker project"
      ansible.builtin.git:
        repo: "https://github.com/netbox-community/netbox-docker.git"
        dest: "{{ netbox_dir }}"
        version: "{{ netbox_version }}"
        force: false
      become: false

    # ── Config files ──────────────────────────────────────────────────
    - name: "Netbox | Deploy override compose file"
      ansible.builtin.template:
        src: docker-compose.override.yml.j2
        dest: "{{ netbox_dir }}/docker-compose.override.yml"
        owner: ansible
        group: ansible
        mode: '0644'

    - name: "Netbox | Deploy nginx config"
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: "{{ netbox_dir }}/nginx.conf"
        owner: ansible
        group: ansible
        mode: '0644'

    - name: "Netbox | Deploy environment file"
      ansible.builtin.template:
        src: netbox.env.j2
        dest: "{{ netbox_dir }}/.env"
        owner: ansible
        group: ansible
        mode: '0600'

    # ── UFW ───────────────────────────────────────────────────────────
    - name: "UFW | Allow Netbox web from management network"
      community.general.ufw:
        rule: allow
        port: "80"
        proto: tcp
        src: "172.16.0.0/24"
        comment: "Netbox web UI and API"

    - name: "UFW | Allow Netbox web from telemetry network"
      community.general.ufw:
        rule: allow
        port: "80"
        proto: tcp
        src: "192.168.100.0/24"
        comment: "Netbox API from telemetry network (Ansible, monitoring)"

    # ── Start stack ───────────────────────────────────────────────────
    - name: "Docker | Pull Netbox images"
      community.docker.docker_compose_v2:
        project_src: "{{ netbox_dir }}"
        pull: always
        state: present
      become: false

    - name: "Docker | Start Netbox stack"
      community.docker.docker_compose_v2:
        project_src: "{{ netbox_dir }}"
        state: present
      become: false

    # ── Health check ──────────────────────────────────────────────────
    - name: "Netbox | Wait for API to respond"
      ansible.builtin.uri:
        url: "http://172.16.0.20/api/"
        headers:
          Authorization: "Token {{ vault_netbox_api_token }}"
        status_code: 200
        timeout: 15
      register: netbox_api
      retries: 20
      delay: 15
      until: netbox_api.status == 200
      delegate_to: localhost

    - name: "Netbox | Report deployment result"
      ansible.builtin.debug:
        msg:
          - "════════════════════════════════════════"
          - " Netbox is running"
          - " Web UI: http://172.16.0.20"
          - " API:    http://172.16.0.20/api/"
          - " Version: {{ netbox_api.json.netbox-version | default('check UI') }}"
          - "════════════════════════════════════════"
EOF
```

### Environment template

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/netbox/templates/netbox.env.j2 << 'EOF'
# Managed by Ansible — do not edit manually
NETBOX_DB_PASSWORD={{ vault_netbox_db_password }}
REDIS_PASSWORD={{ vault_redis_password }}
REDIS_CACHE_PASSWORD={{ vault_redis_cache_password }}
SECRET_KEY={{ vault_netbox_secret_key }}
NETBOX_SUPERUSER_PASSWORD={{ vault_netbox_admin_password }}
NETBOX_API_TOKEN={{ vault_netbox_api_token }}
EOF
```

### Run the deployment

```bash
# Snapshot netbox VM before deploying
ansible-playbook \
  playbooks/infrastructure/lifecycle/snapshot.yml \
  -e "target_vmid=102 snap_name=pre-netbox \
      snap_description='Before Netbox deploy Part 36'" \
  --vault-id lab@.vault/lab.txt

# Deploy
ansible-playbook \
  playbooks/infrastructure/netbox/deploy.yml \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt

# Verify from control node
curl -s \
  -H "Authorization: Token $(ansible-vault view \
      inventory/group_vars/all/vault.yml \
      --vault-id lab@.vault/lab.txt \
      | grep vault_netbox_api_token | awk '{print $2}')" \
  "http://172.16.0.20/api/dcim/devices/?limit=5" \
  | python3 -m json.tool | grep '"count"'
```

---

## 36.5 — Netbox Configuration for the Lab

Part 26 populated Netbox with devices. This section adds the production-grade configuration that a properly managed Netbox instance needs — config context, custom fields, and the settings that make the API more useful for automation.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/netbox/configure.yml << 'EOF'
---
# Configure Netbox for the lab environment
# Idempotent — safe to run multiple times
# Builds on Part 26 device population

- name: "Netbox | Configure lab environment"
  hosts: localhost
  gather_facts: false
  tags: [netbox, configure]

  vars:
    netbox_url: "http://172.16.0.20"
    netbox_token: "{{ vault_netbox_api_token }}"

  tasks:

    # ── Custom fields ─────────────────────────────────────────────────
    # Custom fields add structured metadata to devices — useful for
    # Ansible to query specific attributes via the API

    - name: "Custom fields | Add ansible_group field to devices"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/extras/custom-fields/"
        method: POST
        headers:
          Authorization: "Token {{ netbox_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          name: "ansible_group"
          label: "Ansible Group"
          type: "text"
          object_types: ["dcim.device"]
          description: "Primary Ansible inventory group for this device"
          required: false
          weight: 100
        status_code: [201, 400]   # 400 = already exists — idempotent
      register: cf_result
      changed_when: cf_result.status == 201

    - name: "Custom fields | Add management_protocol field"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/extras/custom-fields/"
        method: POST
        headers:
          Authorization: "Token {{ netbox_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          name: "management_protocol"
          label: "Management Protocol"
          type: "select"
          object_types: ["dcim.device"]
          description: "How Ansible connects to this device"
          required: false
          choices:
            - network_cli
            - netconf
            - httpapi
            - ssh
          weight: 110
        status_code: [201, 400]
      changed_when: false

    # ── Config context ────────────────────────────────────────────────
    # Config context allows storing structured YAML data against devices
    # or device types — Ansible can retrieve this via the API and use it
    # as host variables, replacing static host_vars files with Netbox data

    - name: "Config context | Create IOS-XE connection context"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/extras/config-contexts/"
        method: POST
        headers:
          Authorization: "Token {{ netbox_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          name: "IOS-XE Connection Settings"
          weight: 100
          description: "Ansible connection settings for Cisco IOS-XE devices"
          is_active: true
          platforms:
            - slug: "ios-xe"
          data:
            ansible_connection: network_cli
            ansible_network_os: "cisco.ios.ios"
            ansible_become: true
            ansible_become_method: enable
        status_code: [201, 400]
      changed_when: false

    - name: "Config context | Create NX-OS connection context"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/extras/config-contexts/"
        method: POST
        headers:
          Authorization: "Token {{ netbox_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          name: "NX-OS Connection Settings"
          weight: 100
          description: "Ansible connection settings for Cisco NX-OS devices"
          is_active: true
          platforms:
            - slug: "nxos"
          data:
            ansible_connection: network_cli
            ansible_network_os: "cisco.nxos.nxos"
        status_code: [201, 400]
      changed_when: false

    - name: "Config context | Create PAN-OS connection context"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/extras/config-contexts/"
        method: POST
        headers:
          Authorization: "Token {{ netbox_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          name: "PAN-OS Connection Settings"
          weight: 100
          is_active: true
          platforms:
            - slug: "panos"
          data:
            ansible_connection: ansible.netcommon.httpapi
            ansible_network_os: "paloaltonetworks.panos.panos"
            ansible_httpapi_use_ssl: true
            ansible_httpapi_validate_certs: false
        status_code: [201, 400]
      changed_when: false

    # ── Webhook setup — see Section 36.6 ─────────────────────────────
    - name: "Netbox | Confirm configuration complete"
      ansible.builtin.debug:
        msg:
          - "Custom fields and config contexts configured"
          - "Netbox API: http://172.16.0.20/api/"
          - "Devices: http://172.16.0.20/dcim/devices/"
EOF
```

---

## 36.6 — Automated Backup to Git

Rather than a pg_dump, this strategy exports Netbox data as structured JSON using Netbox's built-in export API, then commits the result to a dedicated backup repo in Gitea. The result is a human-readable, version-controlled history of every device, prefix, and IP change.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/netbox/backup.yml << 'EOF'
---
# Export Netbox data as JSON and commit to Gitea
# Run on a schedule (cron/AWX) or manually before changes
# Usage: ansible-playbook playbooks/infrastructure/netbox/backup.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Netbox | Export and backup to Git"
  hosts: netbox_hosts
  gather_facts: true
  become: false
  tags: [netbox, backup]

  vars:
    netbox_url: "http://172.16.0.20"
    netbox_token: "{{ vault_netbox_api_token }}"
    backup_dir: "/home/ansible/netbox-backup"
    gitea_url: "http://172.16.0.30:3000"
    gitea_token: "{{ vault_gitea_ansible_token }}"
    gitea_org: "network-automation"
    backup_repo: "netbox-backup"
    timestamp: "{{ lookup('pipe', 'date +%Y-%m-%d_%H-%M-%S') }}"

    # Netbox API endpoints to export
    export_endpoints:
      - { name: "devices",          path: "/api/dcim/devices/?limit=1000" }
      - { name: "interfaces",       path: "/api/dcim/interfaces/?limit=1000" }
      - { name: "ip_addresses",     path: "/api/ipam/ip-addresses/?limit=1000" }
      - { name: "prefixes",         path: "/api/ipam/prefixes/?limit=1000" }
      - { name: "vlans",            path: "/api/ipam/vlans/?limit=1000" }
      - { name: "cables",           path: "/api/dcim/cables/?limit=1000" }
      - { name: "sites",            path: "/api/dcim/sites/?limit=1000" }
      - { name: "device_types",     path: "/api/dcim/device-types/?limit=1000" }
      - { name: "platforms",        path: "/api/dcim/platforms/?limit=1000" }
      - { name: "config_contexts",  path: "/api/extras/config-contexts/?limit=1000" }
      - { name: "custom_fields",    path: "/api/extras/custom-fields/?limit=1000" }
      - { name: "webhooks",         path: "/api/extras/webhooks/?limit=1000" }

  tasks:

    # ── Ensure backup repo exists in Gitea ────────────────────────────
    - name: "Gitea | Check if backup repo exists"
      ansible.builtin.uri:
        url: "{{ gitea_url }}/api/v1/repos/{{ gitea_org }}/{{ backup_repo }}"
        method: GET
        headers:
          Authorization: "token {{ gitea_token }}"
        status_code: [200, 404]
      register: repo_check
      delegate_to: localhost

    - name: "Gitea | Create backup repo if missing"
      ansible.builtin.uri:
        url: "{{ gitea_url }}/api/v1/orgs/{{ gitea_org }}/repos"
        method: POST
        headers:
          Authorization: "token {{ gitea_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          name: "{{ backup_repo }}"
          description: "Automated Netbox data exports"
          private: true
          auto_init: true
          default_branch: main
        status_code: 201
      when: repo_check.status == 404
      delegate_to: localhost

    # ── Set up local git repo ─────────────────────────────────────────
    - name: "Backup | Create backup directory"
      ansible.builtin.file:
        path: "{{ backup_dir }}"
        state: directory
        mode: '0755'

    - name: "Backup | Clone or update backup repo from Gitea"
      ansible.builtin.git:
        repo: "http://gitea-admin:{{ vault_gitea_ansible_token }}@172.16.0.30:3000/{{ gitea_org }}/{{ backup_repo }}.git"
        dest: "{{ backup_dir }}"
        version: main
        update: true
        force: true

    - name: "Backup | Create dated export directory"
      ansible.builtin.file:
        path: "{{ backup_dir }}/exports/{{ timestamp }}"
        state: directory
        mode: '0755'

    # ── Export each Netbox endpoint ───────────────────────────────────
    - name: "Export | Fetch {{ item.name }} from Netbox API"
      ansible.builtin.uri:
        url: "{{ netbox_url }}{{ item.path }}"
        method: GET
        headers:
          Authorization: "Token {{ netbox_token }}"
        return_content: true
        timeout: 30
      register: api_responses
      loop: "{{ export_endpoints }}"
      loop_control:
        label: "{{ item.name }}"
      delegate_to: localhost

    - name: "Export | Write {{ item.item.name }}.json to backup directory"
      ansible.builtin.copy:
        content: "{{ item.json | to_nice_json }}"
        dest: "{{ backup_dir }}/exports/{{ timestamp }}/{{ item.item.name }}.json"
        mode: '0644'
      loop: "{{ api_responses.results }}"
      loop_control:
        label: "{{ item.item.name }}.json"

    # ── Write latest symlink and summary ──────────────────────────────
    - name: "Export | Write backup summary"
      ansible.builtin.copy:
        content: |
          Netbox Backup Summary
          =====================
          Timestamp:  {{ timestamp }}
          Netbox URL: {{ netbox_url }}
          Generated:  {{ lookup('pipe', 'date') }}

          Exported objects:
          {% for item in api_responses.results %}
          - {{ item.item.name }}: {{ item.json.count }} records
          {% endfor %}
        dest: "{{ backup_dir }}/exports/{{ timestamp }}/SUMMARY.txt"
        mode: '0644'

    - name: "Export | Update latest symlink"
      ansible.builtin.file:
        src: "{{ backup_dir }}/exports/{{ timestamp }}"
        dest: "{{ backup_dir }}/exports/latest"
        state: link
        force: true

    # ── Commit and push to Gitea ──────────────────────────────────────
    - name: "Git | Configure git identity on netbox VM"
      ansible.builtin.git_config:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        scope: global
      loop:
        - { name: user.email, value: "ansible@lab.local" }
        - { name: user.name,  value: "Ansible Backup" }

    - name: "Git | Stage all export files"
      ansible.builtin.command:
        cmd: git add -A
        chdir: "{{ backup_dir }}"
      changed_when: true

    - name: "Git | Check if there are changes to commit"
      ansible.builtin.command:
        cmd: git status --porcelain
        chdir: "{{ backup_dir }}"
      register: git_status
      changed_when: false

    - name: "Git | Commit export"
      ansible.builtin.command:
        cmd: >
          git commit -m
          "backup: Netbox export {{ timestamp }}
          
          Exported {{ export_endpoints | length }} object types.
          Triggered by: Ansible backup playbook"
        chdir: "{{ backup_dir }}"
      when: git_status.stdout | length > 0
      changed_when: true

    - name: "Git | Push to Gitea"
      ansible.builtin.command:
        cmd: git push origin main
        chdir: "{{ backup_dir }}"
      environment:
        GIT_ASKPASS: echo
        GIT_USERNAME: gitea-admin
        GIT_PASSWORD: "{{ vault_gitea_ansible_token }}"
      when: git_status.stdout | length > 0
      changed_when: true
      no_log: true

    - name: "Backup | Report result"
      ansible.builtin.debug:
        msg:
          - "Netbox backup complete"
          - "Timestamp: {{ timestamp }}"
          - "Repo: {{ gitea_url }}/{{ gitea_org }}/{{ backup_repo }}"
          - "Changes committed: {{ git_status.stdout | length > 0 }}"
EOF
```

### Schedule the backup via AWX

```bash
# After AWX is deployed in Part 38, create a scheduled job for backups.
# For now, test manually:
ansible-playbook \
  playbooks/infrastructure/netbox/backup.yml \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt

# Verify backup repo in Gitea
curl -s \
  -H "Authorization: token ${GITEA_ANSIBLE_TOKEN}" \
  "http://172.16.0.30:3000/api/v1/repos/network-automation/netbox-backup/contents/exports" \
  | python3 -m json.tool | grep '"name"'
```

### What the backup repo looks like

```
netbox-backup/
└── exports/
    ├── latest → 2025-03-01_02-00-00/   (symlink to most recent)
    ├── 2025-03-01_02-00-00/
    │   ├── SUMMARY.txt
    │   ├── devices.json          ← all devices with full detail
    │   ├── interfaces.json
    │   ├── ip_addresses.json
    │   ├── prefixes.json
    │   ├── vlans.json
    │   ├── cables.json
    │   ├── sites.json
    │   ├── device_types.json
    │   ├── platforms.json
    │   ├── config_contexts.json
    │   ├── custom_fields.json
    │   └── webhooks.json
    └── 2025-02-29_02-00-00/
        └── ...

Git history shows exactly when each device was added, modified,
or removed — a searchable audit trail for every Netbox change.
```

---

## 36.7 — Netbox Webhooks → AWX

When a device is created or updated in Netbox, a webhook fires to AWX and triggers a job template. This closes the loop between the source of truth and the automation engine — a network engineer adds a device in Netbox, and Ansible automatically runs a discovery or configuration playbook against it.

### Create the webhook in Netbox

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/netbox/webhooks.yml << 'EOF'
---
# Configure Netbox webhooks to trigger AWX job templates
# Run after AWX is deployed and job templates exist (Part 38+)
# Usage: ansible-playbook playbooks/infrastructure/netbox/webhooks.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt
#        -e "awx_job_template_id=5"

- name: "Netbox | Configure webhooks"
  hosts: localhost
  gather_facts: false
  tags: [netbox, webhooks]

  vars:
    netbox_url: "http://172.16.0.20"
    netbox_token: "{{ vault_netbox_api_token }}"
    awx_url: "http://172.16.0.30"
    awx_job_template_id: ~     # Pass via -e "awx_job_template_id=5"

  pre_tasks:
    - name: "Gate | Require AWX job template ID"
      ansible.builtin.assert:
        that: awx_job_template_id is not none
        fail_msg: "Pass -e 'awx_job_template_id=<ID>' — find it in AWX UI under Templates"

  tasks:

    - name: "Webhook | Check for existing device webhook"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/extras/webhooks/?name=AWX+Device+Change"
        method: GET
        headers:
          Authorization: "Token {{ netbox_token }}"
        status_code: 200
      register: existing_webhooks

    - name: "Webhook | Create device change → AWX webhook"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/extras/webhooks/"
        method: POST
        headers:
          Authorization: "Token {{ netbox_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          name: "AWX Device Change"
          content_types:
            - "dcim.device"
          enabled: true
          type_create: true
          type_update: true
          type_delete: false
          payload_url: "{{ awx_url }}/api/v2/job_templates/{{ awx_job_template_id }}/launch/"
          http_method: "POST"
          http_content_type: "application/json"
          additional_headers: "Authorization: Bearer {{ vault_awx_webhook_token }}"
          body_template: |
            {
              "extra_vars": {
                "netbox_event": "{{ '{{' }} data.event {{ '}}' }}",
                "netbox_device": "{{ '{{' }} data.object.name {{ '}}' }}",
                "netbox_device_id": "{{ '{{' }} data.object.id {{ '}}' }}",
                "netbox_status": "{{ '{{' }} data.object.status.value {{ '}}' }}"
              }
            }
          ssl_verification: false
          secret: "{{ vault_awx_webhook_secret }}"
        status_code: [201, 400]
      when: existing_webhooks.json.count == 0
      register: webhook_result
      changed_when: webhook_result.status == 201

    - name: "Webhook | Report"
      ansible.builtin.debug:
        msg:
          - "Webhook configured: Netbox device change → AWX job template {{ awx_job_template_id }}"
          - "Test: Add or update a device in Netbox UI and check AWX for a new job"
EOF
```

### The automation flow this creates

```
Network engineer opens Netbox UI
  → Adds new device: dist-02 (Cat9Kv, 172.16.0.22)
  → Clicks Save

Netbox fires webhook to AWX:
  POST http://172.16.0.30/api/v2/job_templates/5/launch/
  Body: {
    "extra_vars": {
      "netbox_event": "created",
      "netbox_device": "dist-02",
      "netbox_device_id": 8,
      "netbox_status": "active"
    }
  }

AWX launches job template 5 (e.g. "Netbox Device Sync")
  → Playbook queries Netbox API for dist-02 details
  → Runs connectivity check against 172.16.0.22
  → Creates host_vars/dist-02/ in Git
  → Pushes to Gitea
  → Sends Slack/email notification

Total time: device appears in Netbox → Ansible has acted on it: < 2 minutes
```

### AWX job template for Netbox device events

```yaml
# playbooks/infrastructure/netbox/device_sync.yml
# This playbook is triggered by the Netbox webhook
---
- name: "Netbox | Handle device change event"
  hosts: localhost
  gather_facts: false
  tags: [netbox, device-sync]

  vars:
    netbox_url: "http://172.16.0.20"
    netbox_token: "{{ vault_netbox_api_token }}"
    # These vars come from the webhook extra_vars:
    # netbox_device, netbox_device_id, netbox_event, netbox_status

  tasks:

    - name: "Event | Log received webhook"
      ansible.builtin.debug:
        msg:
          - "Netbox webhook received"
          - "Event:  {{ netbox_event | default('unknown') }}"
          - "Device: {{ netbox_device | default('unknown') }}"
          - "Status: {{ netbox_status | default('unknown') }}"

    - name: "Netbox | Fetch full device details"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/dcim/devices/{{ netbox_device_id }}/"
        method: GET
        headers:
          Authorization: "Token {{ netbox_token }}"
      register: device_detail
      when: netbox_event in ['created', 'updated']

    - name: "Netbox | Fetch device interfaces"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/dcim/interfaces/?device_id={{ netbox_device_id }}&limit=50"
        method: GET
        headers:
          Authorization: "Token {{ netbox_token }}"
      register: device_interfaces
      when: netbox_event in ['created', 'updated']

    - name: "Report | Show device data"
      ansible.builtin.debug:
        msg:
          - "Device:    {{ device_detail.json.name }}"
          - "Platform:  {{ device_detail.json.platform.slug | default('unset') }}"
          - "Primary IP: {{ device_detail.json.primary_ip.address | default('unset') }}"
          - "Interfaces: {{ device_interfaces.json.count }}"
      when: netbox_event in ['created', 'updated']

    # Additional tasks for device_sync role would go here:
    # - Write host_vars file to Git
    # - Add device to Zabbix monitoring
    # - Add device to Prometheus scrape config
    # (Covered as these services are deployed in Parts 37-38)
```

---

## 36.8 — Health Monitoring Integration

Netbox's own health is monitored by Zabbix (Part 37) and Prometheus (Part 38). This section creates the groundwork — a Netbox health endpoint check and a simple status script.

```bash
# Netbox exposes a health endpoint built-in
curl -s http://172.16.0.20/health/ | python3 -m json.tool
# Expected:
# {
#   "netbox-version": "4.x.x",
#   "python-version": "3.x.x",
#   "plugins": [],
#   "rq-workers-running": 1
# }

# Key things to monitor (covered in Part 37 and 38):
# HTTP 200 on /health/           — Netbox app is up
# rq-workers-running > 0        — background worker is running
# HTTP 200 on /api/              — API is responsive
# PostgreSQL connection          — database reachable
# Redis connection               — cache working

# Create a simple local health-check script (also used by Zabbix agent)
cat > /opt/netbox/healthcheck.sh << 'EOF'
#!/bin/bash
# Returns 0 (healthy) or 1 (unhealthy) — used by Zabbix and monitoring

HEALTH=$(curl -sf http://localhost/health/ 2>/dev/null)
if [ $? -ne 0 ]; then
  echo "UNHEALTHY: Netbox not responding"
  exit 1
fi

WORKERS=$(echo "$HEALTH" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('rq-workers-running',0))")
if [ "$WORKERS" -lt 1 ]; then
  echo "UNHEALTHY: No RQ workers running"
  exit 1
fi

echo "HEALTHY: Netbox running, ${WORKERS} worker(s)"
exit 0
EOF
chmod +x /opt/netbox/healthcheck.sh
```

---

## 36.9 — Upgrade Procedure

One of the main reasons for running Netbox via Docker Compose is that upgrades are clean and reversible.

```bash
# Step 1 — Snapshot the VM before upgrading
ansible-playbook \
  playbooks/infrastructure/lifecycle/snapshot.yml \
  -e "target_vmid=102 snap_name=pre-netbox-upgrade-$(date +%Y%m%d) \
      snap_description='Before Netbox version upgrade'" \
  --vault-id lab@.vault/lab.txt

# Step 2 — Back up current data
ansible-playbook \
  playbooks/infrastructure/netbox/backup.yml \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt

# Step 3 — On the netbox VM, update the pinned version
ssh ansible@172.16.0.20
cd /opt/netbox

# Check current version
docker compose exec netbox python3 -m netbox --version

# Pull the new version tag from netbox-docker releases
git fetch origin
git checkout 3.2.0    # example — check actual latest release

# Pull new images
docker compose pull

# Run database migrations (netbox-docker does this automatically on startup)
docker compose up -d

# Watch migration logs
docker compose logs -f netbox | grep -E "migration|error|started"

# Verify health after upgrade
curl -s http://172.16.0.20/health/ | python3 -m json.tool

# Step 4 — If upgrade fails, restore the snapshot
# ansible-playbook lifecycle/restore.yml -e "target_vmid=102 snap_name=pre-netbox-upgrade-YYYYMMDD"
```

---

## 36.10 — Checklist

```
[ ] PostgreSQL decision documented:
    [ ] Using dedicated PostgreSQL on netbox VM (Option A)
    [ ] Rationale understood: isolation, snapshot simplicity

[ ] Netbox stack deployed:
    [ ] netbox-docker cloned at /opt/netbox, pinned to release
    [ ] docker-compose.override.yml in place
    [ ] nginx.conf in place
    [ ] .env file created with generated secrets
    [ ] All secrets added to Ansible vault:
        vault_netbox_db_password
        vault_redis_password
        vault_redis_cache_password
        vault_netbox_secret_key
        vault_netbox_admin_password
        vault_netbox_api_token
    [ ] docker compose ps — all 7 containers healthy/running
    [ ] http://172.16.0.20 loads Netbox web UI
    [ ] http://172.16.0.20/api/ returns JSON with netbox-version
    [ ] Admin login works with vault_netbox_admin_password

[ ] Ansible deploy playbook:
    [ ] playbooks/infrastructure/netbox/deploy.yml created
    [ ] netbox.env.j2 template created
    [ ] Playbook runs idempotently — second run shows no unexpected CHANGED

[ ] Netbox configuration:
    [ ] configure.yml run — custom fields created:
        [ ] ansible_group field on devices
        [ ] management_protocol field on devices
    [ ] Config contexts created for IOS-XE, NX-OS, PAN-OS
    [ ] Devices from Part 26 still present (or re-populated)

[ ] Backup to Git:
    [ ] backup.yml playbook runs successfully
    [ ] netbox-backup repo created in Gitea (network-automation org)
    [ ] Backup directory shows dated export folder with 12 JSON files
    [ ] SUMMARY.txt shows correct record counts per object type
    [ ] Git log in backup repo shows commit with timestamp message
    [ ] Second backup run only commits if data changed (idempotent)

[ ] Webhooks (partial — completes when AWX deployed in Part 38):
    [ ] webhooks.yml playbook created
    [ ] device_sync.yml playbook created
    [ ] Webhook URL noted for AWX job template creation

[ ] Health monitoring groundwork:
    [ ] /opt/netbox/healthcheck.sh created and executable
    [ ] curl http://172.16.0.20/health/ returns healthy JSON
    [ ] rq-workers-running > 0 confirmed

[ ] UFW rules verified:
    [ ] Port 80 open from 172.16.0.0/24 (management)
    [ ] Port 80 open from 192.168.100.0/24 (telemetry/Ansible)
    [ ] Port 5432 NOT externally accessible

[ ] Upgrade procedure understood and documented

[ ] All playbooks committed to Git and mirrored to Gitea:
    [ ] playbooks/infrastructure/netbox/deploy.yml
    [ ] playbooks/infrastructure/netbox/configure.yml
    [ ] playbooks/infrastructure/netbox/backup.yml
    [ ] playbooks/infrastructure/netbox/webhooks.yml
    [ ] playbooks/infrastructure/netbox/device_sync.yml

[ ] Snapshot taken post-deployment:
    ansible-playbook lifecycle/snapshot.yml \
      -e "target_vmid=102 snap_name=post-netbox \
          snap_description='Netbox running and configured Part 36'"
```

---

*Netbox is running as a properly managed service — deployed by Ansible, backed up automatically to Git, with webhooks ready to fire into AWX the moment a device changes. The backup repo in Gitea is already accumulating an audit trail of every device, prefix, and IP in the lab. When Zabbix and Prometheus are deployed in Parts 37 and 38, Netbox's health endpoint slots straight into their monitoring configs. The source of truth is now itself a managed, observed, and recoverable service.*

*Next up: **Part 37 — Traditional Monitoring: Zabbix + SNMP***
