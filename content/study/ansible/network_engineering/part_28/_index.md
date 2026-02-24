---
draft: true
title: '28 - AWX'
weight: 28
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 28: AWX — Enterprise Ansible Automation

> *Every playbook in this guide runs from a terminal prompt: ssh to the control node, activate the venv, type ansible-playbook, read the output. That works for one engineer. AWX takes the same playbooks — unchanged — and wraps them in a platform that lets a team manage, audit, and trigger automation through a web UI and REST API. Job history is searchable. Credentials are stored encrypted and never shown to users. RBAC controls who can run what. Schedules replace cron entries. Workflows chain playbooks with approval gates. The automation is identical; the operational experience is completely different.*

---

## 28.1 — AWX Architecture and Core Concepts

AWX runs as a set of containers orchestrated by Kubernetes. The Ansible playbooks themselves don't change — AWX is the management layer on top.

```
AWX Architecture:
┌─────────────────────────────────────────────────────┐
│  AWX Web UI (port 80/443)                           │
│  AWX REST API (/api/v2/)                            │
├──────────────┬──────────────┬───────────────────────┤
│  AWX Web     │  AWX Task    │  AWX Worker           │
│  Container   │  Container   │  Containers           │
│  (Django)    │  (Celery)    │  (Ansible runs here)  │
├──────────────┴──────────────┴───────────────────────┤
│  PostgreSQL  │  Redis       │  Receptor             │
│  (jobs/logs) │  (queue)     │  (execution mesh)     │
└─────────────────────────────────────────────────────┘
         ↕ Kubernetes manages all containers
```

### Core AWX Objects

Every object in AWX maps to something familiar from the command-line workflow:

| AWX Object | CLI Equivalent | Purpose |
|---|---|---|
| Organization | Project directory | Top-level container for all other objects |
| Team | Unix group | Collection of users with shared permissions |
| Credential | vault.yml + SSH key | Encrypted storage for passwords, keys, tokens |
| Project | Git repository | Source of playbooks (pulled from SCM) |
| Inventory | hosts.yml / netbox.yml | Device groups and variables |
| Job Template | `ansible-playbook` command | A playbook + inventory + credentials combined |
| Workflow | Shell script chaining playbooks | Ordered sequence of Job Templates |
| Schedule | crontab entry | Time-based job triggering |
| Notification | syslog alert in wrapper script | Email/Slack/webhook on job events |

The dependency chain for running a job:

```
Organization
  └── Project (Git repo with playbooks)
  └── Inventory (devices to run against)
  └── Credential (how to authenticate)
        └── Job Template (ties project + inventory + credential together)
              └── Schedule (when to run automatically)
              └── Workflow (chain of Job Templates)
```

---

## 28.2 — Installing AWX

### Option A: k3s (Recommended for this lab)

k3s is a lightweight Kubernetes distribution that runs on a single Ubuntu node with 4GB RAM minimum. It's the right choice for the lab control node.

```
Resource requirements:
  k3s alone:     ~512MB RAM
  AWX on k3s:    ~4GB RAM minimum, 8GB recommended
  Full lab:      8GB+ (AWX + Containerlab devices)
```

```bash
# ── Step 1: Install k3s ───────────────────────────────────────────
curl -sfL https://get.k3s.io | sh -

# Wait for k3s to be ready
sudo k3s kubectl get nodes
# Should show: NAME    STATUS   ROLES    AGE
#              ubuntu  Ready    master   1m

# Set up kubeconfig for your user (not root)
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(whoami):$(whoami) ~/.kube/config
# Fix the server URL (k3s uses 127.0.0.1 internally)
sed -i 's/127.0.0.1/172.16.0.10/g' ~/.kube/config   # Replace with your control node IP

# Install kubectl (k3s bundles it but this alias is cleaner)
echo 'alias kubectl="sudo k3s kubectl"' >> ~/.bashrc
source ~/.bashrc

# ── Step 2: Install AWX Operator ──────────────────────────────────
# The AWX Operator manages the AWX deployment lifecycle on Kubernetes

# Create namespace for AWX
kubectl create namespace awx

# Install the AWX Operator (check https://github.com/ansible/awx-operator for latest version)
kubectl apply -n awx -f \
  https://raw.githubusercontent.com/ansible/awx-operator/2.19.1/deploy/awx-operator.yaml

# Verify operator pod is running
kubectl get pods -n awx
# AWX Operator pod should show Running

# ── Step 3: Create the AWX instance ───────────────────────────────
cat > /tmp/awx-instance.yaml << 'YAML'
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-lab
  namespace: awx
spec:
  service_type: nodeport
  nodeport_port: 30080        # Access AWX at http://<control-node-ip>:30080
  admin_user: admin
  admin_email: admin@lab.local
  postgres_storage_class: local-path
  projects_persistence: true
  projects_storage_size: 8Gi
  extra_settings:
    - setting: AWX_TASK_ENV
      value:
        HOME: /var/lib/awx
YAML

kubectl apply -f /tmp/awx-instance.yaml

# ── Step 4: Wait for AWX to come up (takes 5-10 minutes) ──────────
kubectl get pods -n awx -w
# Watch until all pods show Running:
#   awx-lab-postgres-15-0    Running
#   awx-lab-task-xxx         Running
#   awx-lab-web-xxx          Running
#   awx-lab-redis-xxx        Running

# ── Step 5: Get the admin password ────────────────────────────────
kubectl get secret awx-lab-admin-password -n awx \
  -o jsonpath='{.data.password}' | base64 --decode && echo
# Save this password — you'll need it for first login

# AWX is now available at:
# http://172.16.0.10:30080
# Login: admin / <password from above>
```

### Option B: minikube (If Docker is already installed)

```bash
# minikube needs more resources and is slower to start
# Requires Docker or VirtualBox already installed

# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start with enough resources for AWX
minikube start \
  --cpus 4 \
  --memory 8192 \
  --addons ingress

# After minikube is running, the AWX Operator install steps are identical to k3s
```

**Recommendation: Use k3s for this lab.** It uses less RAM than minikube, starts faster, persists across reboots without `minikube start`, and is closer to what production Kubernetes looks like. The rest of this part uses k3s.

### Install the AWX CLI

The `awx` CLI communicates with the AWX REST API and is used throughout this part for scripted object creation:

```bash
source ~/ansible-venv/bin/activate
pip install awxkit --break-system-packages

# Configure the CLI to point at the AWX instance
awx login \
  --conf.host http://172.16.0.10:30080 \
  --conf.username admin \
  --conf.password "$(kubectl get secret awx-lab-admin-password -n awx \
      -o jsonpath='{.data.password}' | base64 --decode)"

# Verify
awx config
awx version
```

---

## 28.3 — Organizations and Teams

### Creating the Network Automation Organization

**UI path:** Resources → Organizations → Add

```
Name:         Network Automation Lab
Description:  Network automation for the Containerlab environment
Max hosts:    0 (unlimited)
```

**CLI equivalent:**
```bash
awx organizations create \
  --name "Network Automation Lab" \
  --description "Network automation for the Containerlab environment"

# Capture the org ID for use in later commands
ORG_ID=$(awx organizations list --name "Network Automation Lab" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")
echo "Organization ID: ${ORG_ID}"
```

### Creating Teams with RBAC Intent

Two teams represent the typical split in a network operations environment:

**Team 1 — Network Engineers:** Can run any playbook, create job templates, manage inventory.

**Team 2 — Network Operations (NOC):** Can run pre-approved job templates only. Cannot create or modify anything.

**UI path:** Resources → Teams → Add

```
Team 1:
  Name:         Network Engineers
  Organization: Network Automation Lab

Team 2:
  Name:         Network Operations
  Organization: Network Automation Lab
```

**CLI:**
```bash
# Create teams
awx teams create \
  --name "Network Engineers" \
  --organization "${ORG_ID}"

awx teams create \
  --name "Network Operations" \
  --organization "${ORG_ID}"

# Store team IDs
TEAM_ENGINEERS=$(awx teams list --name "Network Engineers" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")
TEAM_NOC=$(awx teams list --name "Network Operations" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")
```

### Creating Users

**UI path:** Access → Users → Add

```
User 1 — Engineer:
  Username:     alice
  Password:     (set at creation)
  Email:        alice@lab.local
  User type:    Normal user
  Organization: Network Automation Lab
  Teams:        Network Engineers

User 2 — NOC:
  Username:     bob
  Password:     (set at creation)
  Email:        bob@lab.local
  User type:    Normal user
  Organization: Network Automation Lab
  Teams:        Network Operations
```

**CLI:**
```bash
awx users create \
  --username alice \
  --password "AlicePassword#2025" \
  --email alice@lab.local \
  --first_name Alice \
  --last_name Engineer

awx users create \
  --username bob \
  --password "BobPassword#2025" \
  --email bob@lab.local \
  --first_name Bob \
  --last_name NOC

# Add users to teams
ALICE_ID=$(awx users list --username alice -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")
BOB_ID=$(awx users list --username bob -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

awx teams associate "${TEAM_ENGINEERS}" --users "${ALICE_ID}"
awx teams associate "${TEAM_NOC}" --users "${BOB_ID}"
```

---

## 28.4 — Credentials

Credentials in AWX are stored encrypted in the PostgreSQL database. Users with Execute permission on a Job Template can use a credential without ever seeing the underlying secret.

### Device SSH Credential

**UI path:** Resources → Credentials → Add

```
Name:            Network Device SSH
Organization:    Network Automation Lab
Credential type: Machine
  Username:      ansible
  Password:      (leave blank — using SSH key)
  SSH Private Key: (paste contents of ~/.ssh/ansible_ed25519)
  Privilege Escalation Method: enable
  Privilege Escalation Password: (vault_ios_enable_secret value)
```

**CLI:**
```bash
# Read the SSH private key
SSH_KEY=$(cat ~/.ssh/ansible_ed25519)

awx credentials create \
  --name "Network Device SSH" \
  --organization "${ORG_ID}" \
  --credential_type 1 \
  --inputs "{
    \"username\": \"ansible\",
    \"ssh_key_data\": \"${SSH_KEY}\",
    \"become_method\": \"enable\",
    \"become_password\": \"$(cat .vault/lab.txt | ansible-vault decrypt --output - \
        inventory/group_vars/all/vault.yml 2>/dev/null \
        | grep vault_ios_enable_secret | awk '{print $2}')\"
  }"
# Note: credential_type 1 = Machine (SSH) in a default AWX install
# Run: awx credential_types list to see all types and their IDs

CRED_SSH_ID=$(awx credentials list --name "Network Device SSH" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")
```

**RBAC — assign to teams:**

```
Network Engineers: Admin (can view, use, edit, delete)
Network Operations: Use (can use in job runs — cannot view the secret)
```

```bash
# Grant Network Engineers admin on this credential
awx credentials grant "${CRED_SSH_ID}" \
  --team "${TEAM_ENGINEERS}" \
  --role admin

# Grant Network Operations use-only
awx credentials grant "${CRED_SSH_ID}" \
  --team "${TEAM_NOC}" \
  --role use
```

### Vault Password Credential

**UI path:** Resources → Credentials → Add

```
Name:            Ansible Vault — Lab
Organization:    Network Automation Lab
Credential type: Vault
  Vault Password: (contents of .vault/lab.txt)
  Vault Identifier: lab (matches --vault-id lab@.vault/lab.txt)
```

**CLI:**
```bash
VAULT_PASS=$(cat ~/projects/ansible-network/.vault/lab.txt)

awx credentials create \
  --name "Ansible Vault — Lab" \
  --organization "${ORG_ID}" \
  --credential_type 3 \
  --inputs "{
    \"vault_password\": \"${VAULT_PASS}\",
    \"vault_id\": \"lab\"
  }"
# credential_type 3 = Vault in default AWX

CRED_VAULT_ID=$(awx credentials list --name "Ansible Vault — Lab" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

# Engineers only — NOC doesn't need vault access
awx credentials grant "${CRED_VAULT_ID}" \
  --team "${TEAM_ENGINEERS}" \
  --role admin
```

### Netbox API Token Credential

AWX doesn't have a built-in Netbox credential type — use a custom credential type:

**UI path:** Administration → Credential Types → Add

```
Name:        Netbox API Token
Kind:        Cloud
Input config (YAML):
  fields:
    - id: netbox_token
      type: string
      label: Netbox API Token
      secret: true
    - id: netbox_url
      type: string
      label: Netbox URL

Injector config (YAML):
  extra_vars:
    netbox_token: "{{ netbox_token }}"
    netbox_url: "{{ netbox_url }}"
```

**CLI:**
```bash
# Create the custom credential type first
awx credential_types create \
  --name "Netbox API Token" \
  --kind cloud \
  --inputs '{
    "fields": [
      {"id": "netbox_token", "type": "string", "label": "Netbox API Token", "secret": true},
      {"id": "netbox_url", "type": "string", "label": "Netbox URL"}
    ]
  }' \
  --injectors '{
    "extra_vars": {
      "netbox_token": "{{ netbox_token }}",
      "netbox_url": "{{ netbox_url }}"
    }
  }'

NETBOX_TYPE_ID=$(awx credential_types list --name "Netbox API Token" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

# Create the credential using the custom type
awx credentials create \
  --name "Netbox — Lab" \
  --organization "${ORG_ID}" \
  --credential_type "${NETBOX_TYPE_ID}" \
  --inputs "{
    \"netbox_token\": \"$(grep vault_netbox_token \
        ~/projects/ansible-network/inventory/group_vars/all/vault.yml \
        | awk '{print $2}')\",
    \"netbox_url\": \"http://172.16.0.250:8000\"
  }"
```

---

## 28.5 — Projects (Connecting to GitHub)

A Project in AWX is a pointer to a Git repository containing playbooks. AWX pulls the repo on sync and serves playbooks from it — no files are managed manually on the AWX node.

### Create a GitHub Personal Access Token

```
GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
  Repository access: Only select repositories → ansible-network
  Permissions:
    Contents: Read-only
    Metadata: Read-only
```

### Create the SCM Credential

**UI path:** Resources → Credentials → Add

```
Name:            GitHub — ansible-network
Organization:    Network Automation Lab
Credential type: Source Control
  Username:      your-github-username
  Password:      (GitHub PAT from above)
```

```bash
awx credentials create \
  --name "GitHub — ansible-network" \
  --organization "${ORG_ID}" \
  --credential_type 2 \
  --inputs "{
    \"username\": \"your-github-username\",
    \"password\": \"ghp_your_token_here\"
  }"
# credential_type 2 = Source Control in default AWX

CRED_SCM_ID=$(awx credentials list --name "GitHub — ansible-network" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

# Engineers only — full admin on the project
awx credentials grant "${CRED_SCM_ID}" --team "${TEAM_ENGINEERS}" --role admin
```

### Create the Project

**UI path:** Resources → Projects → Add

```
Name:              Ansible Network Automation
Organization:      Network Automation Lab
Source Control Type: Git
Source Control URL: https://github.com/your-username/ansible-network.git
Source Control Branch: main
Source Control Credential: GitHub — ansible-network
Options:
  ✓ Clean         (discard local changes before sync)
  ✓ Update revision on launch (always pull latest before job runs)
```

```bash
awx projects create \
  --name "Ansible Network Automation" \
  --organization "${ORG_ID}" \
  --scm_type git \
  --scm_url "https://github.com/your-username/ansible-network.git" \
  --scm_branch main \
  --credential "${CRED_SCM_ID}" \
  --scm_clean true \
  --scm_update_on_launch true

PROJECT_ID=$(awx projects list --name "Ansible Network Automation" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

# Trigger initial project sync (pulls the repo)
awx projects update "${PROJECT_ID}"

# Wait for sync to complete
awx projects status "${PROJECT_ID}"

# Grant Engineers admin, NOC read
awx projects grant "${PROJECT_ID}" --team "${TEAM_ENGINEERS}" --role admin
awx projects grant "${PROJECT_ID}" --team "${TEAM_NOC}" --role read
```

---

## 28.6 — Inventories

AWX inventories can be static (pasted YAML/INI) or dynamic (sourced from Netbox, AWS, etc.). The Netbox dynamic inventory from Part 26 maps directly to an AWX inventory source.

### Create the Inventory Object

**UI path:** Resources → Inventories → Add

```
Name:         Containerlab Devices
Organization: Network Automation Lab
Description:  Lab devices — sourced from Netbox
```

```bash
awx inventories create \
  --name "Containerlab Devices" \
  --organization "${ORG_ID}" \
  --description "Lab devices sourced from Netbox dynamic inventory"

INV_ID=$(awx inventories list --name "Containerlab Devices" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

# RBAC: Engineers admin, NOC read (NOC can see inventory but not modify)
awx inventories grant "${INV_ID}" --team "${TEAM_ENGINEERS}" --role admin
awx inventories grant "${INV_ID}" --team "${TEAM_NOC}" --role read
```

### Add Netbox as an Inventory Source

**UI path:** Resources → Inventories → Containerlab Devices → Sources → Add

```
Name:               Netbox Dynamic
Source:             Sourced from a Project
Project:            Ansible Network Automation
Inventory file:     inventory/netbox/netbox.yml
Credential:         Netbox — Lab
Update options:
  ✓ Overwrite       (replace inventory on each sync)
  ✓ Update on launch (sync before each job run)
```

```bash
awx inventory_sources create \
  --name "Netbox Dynamic" \
  --inventory "${INV_ID}" \
  --source scm \
  --source_project "${PROJECT_ID}" \
  --source_path "inventory/netbox/netbox.yml" \
  --credential "${CRED_NETBOX_ID}" \
  --overwrite true \
  --update_on_launch true

# Trigger initial sync
INV_SRC_ID=$(awx inventory_sources list --name "Netbox Dynamic" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

awx inventory_sources update "${INV_SRC_ID}"
```

---

## 28.7 — Job Templates

A Job Template is the combination of a playbook, inventory, and credentials that AWX executes as a unit. It replaces the full `ansible-playbook` command.

### Create Job Templates for the Core Playbooks

**Backup Job Template:**

**UI path:** Resources → Job Templates → Add

```
Name:           Network Backup — All Devices
Job type:       Run
Inventory:      Containerlab Devices
Project:        Ansible Network Automation
Playbook:       playbooks/backup/backup_all.yml
Credentials:    Network Device SSH
                Ansible Vault — Lab
Verbosity:      1 (Normal)
Options:
  ✓ Enable fact storage (stores facts in AWX for later use)
```

```bash
# Create the backup job template
awx job_templates create \
  --name "Network Backup — All Devices" \
  --job_type run \
  --inventory "${INV_ID}" \
  --project "${PROJECT_ID}" \
  --playbook "playbooks/backup/backup_all.yml" \
  --verbosity 1

JT_BACKUP_ID=$(awx job_templates list --name "Network Backup — All Devices" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

# Attach credentials
awx job_templates associate "${JT_BACKUP_ID}" \
  --credential "${CRED_SSH_ID}"
awx job_templates associate "${JT_BACKUP_ID}" \
  --credential "${CRED_VAULT_ID}"

# RBAC: Engineers admin, NOC execute (can run but not edit)
awx job_templates grant "${JT_BACKUP_ID}" --team "${TEAM_ENGINEERS}" --role admin
awx job_templates grant "${JT_BACKUP_ID}" --team "${TEAM_NOC}" --role execute
```

**Validation Job Template:**

```bash
awx job_templates create \
  --name "Network Validate — All Devices" \
  --job_type run \
  --inventory "${INV_ID}" \
  --project "${PROJECT_ID}" \
  --playbook "playbooks/validate/validate_network.yml" \
  --verbosity 1

JT_VALIDATE_ID=$(awx job_templates list --name "Network Validate — All Devices" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

awx job_templates associate "${JT_VALIDATE_ID}" --credential "${CRED_SSH_ID}"
awx job_templates associate "${JT_VALIDATE_ID}" --credential "${CRED_VAULT_ID}"

awx job_templates grant "${JT_VALIDATE_ID}" --team "${TEAM_ENGINEERS}" --role admin
awx job_templates grant "${JT_VALIDATE_ID}" --team "${TEAM_NOC}" --role execute
```

**IOS Deploy Job Template (Engineers only — NOC cannot run):**

```bash
awx job_templates create \
  --name "Deploy IOS CCNA Config" \
  --job_type run \
  --inventory "${INV_ID}" \
  --project "${PROJECT_ID}" \
  --playbook "playbooks/deploy/deploy_ios_ccna.yml" \
  --verbosity 1 \
  --ask_limit_on_launch true     # Prompt for --limit so engineer specifies target

JT_DEPLOY_ID=$(awx job_templates list --name "Deploy IOS CCNA Config" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

awx job_templates associate "${JT_DEPLOY_ID}" --credential "${CRED_SSH_ID}"
awx job_templates associate "${JT_DEPLOY_ID}" --credential "${CRED_VAULT_ID}"

# Engineers only — NOC has NO permission on this template
awx job_templates grant "${JT_DEPLOY_ID}" --team "${TEAM_ENGINEERS}" --role admin
# Bob (NOC) cannot see or run this template at all
```

### Launching a Job from the UI

**UI path:** Resources → Job Templates → Network Backup — All Devices → Launch (rocket icon)

The launch dialog shows:
- Inventory: Containerlab Devices (pre-filled, not changeable unless `ask_inventory_on_launch` is set)
- Credentials: Network Device SSH, Ansible Vault — Lab (pre-filled)
- Variables: (empty unless `ask_variables_on_launch` is set)
- Limit: (empty unless `ask_limit_on_launch` is set — our deploy template prompts here)

Click **Launch** → job starts → output streams live in the browser.

### Launching from CLI

```bash
# Launch the backup job and follow output
awx jobs launch --job_template "${JT_BACKUP_ID}" --monitor

# Launch deploy job with a limit
awx jobs launch \
  --job_template "${JT_DEPLOY_ID}" \
  --limit "wan-r1" \
  --monitor

# Launch and get job ID without waiting
JOB_ID=$(awx jobs launch --job_template "${JT_BACKUP_ID}" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

# Check job status
awx jobs get "${JOB_ID}"

# Stream job output
awx jobs stdout "${JOB_ID}"
```

---

## 28.8 — Schedules (Replacing Cron)

AWX schedules replace the crontab entries from Part 27. The advantage: schedules appear in the UI, have a next-run timestamp, can be paused without editing a file, and their history is visible in the job list.

```bash
# Nightly backup — 02:00 every day
awx schedules create \
  --name "Nightly Backup — 02:00" \
  --unified_job_template "${JT_BACKUP_ID}" \
  --rrule "DTSTART:20250101T020000Z RRULE:FREQ=DAILY;INTERVAL=1"

# Daily validation — 06:00 every day
awx schedules create \
  --name "Daily Validation — 06:00" \
  --unified_job_template "${JT_VALIDATE_ID}" \
  --rrule "DTSTART:20250101T060000Z RRULE:FREQ=DAILY;INTERVAL=1"

# Weekly maintenance check — Monday 07:00
JT_MAINT_ID=$(awx job_templates list --name "Maintenance Check" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

awx schedules create \
  --name "Weekly Maintenance Check — Monday 07:00" \
  --unified_job_template "${JT_MAINT_ID}" \
  --rrule "DTSTART:20250101T070000Z RRULE:FREQ=WEEKLY;BYDAY=MO"
```

**RRULE format** is iCalendar recurrence rule syntax. Common patterns:

```
Daily:              RRULE:FREQ=DAILY;INTERVAL=1
Every 6 hours:      RRULE:FREQ=HOURLY;INTERVAL=6
Weekly on Monday:   RRULE:FREQ=WEEKLY;BYDAY=MO
Monthly on 1st:     RRULE:FREQ=MONTHLY;BYMONTHDAY=1
Weekdays only:      RRULE:FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR
```

**Pause and resume schedules from the UI:**

UI path: Resources → Job Templates → Network Backup — All Devices → Schedules → toggle the Active switch

---

## 28.9 — Workflow Job Templates

A Workflow chains Job Templates together with success/failure branching. The backup → validate workflow runs a backup, and if it succeeds, runs validation. An approval gate pauses the workflow and waits for a human to review and approve before proceeding.

### Backup → Validate Workflow with Approval Gate

**UI path:** Resources → Workflow Job Templates → Add

```
Name:         Backup and Validate Pipeline
Organization: Network Automation Lab
```

**UI Workflow Visualizer (drag-and-drop):**

```
[START] → [Network Backup — All Devices]
                    │
              (on success)
                    │
                    ↓
         [Approval Gate: Review backup]
         "Backup complete. Approve to run validation?"
                    │
              (on approve)
                    │
                    ↓
         [Network Validate — All Devices]
                    │
         ┌──────────┴──────────┐
    (on success)          (on failure)
         │                    │
     [FINISH ✓]         [FINISH ✗ — alert]
```

**CLI — create the workflow and nodes:**

```bash
# Create the workflow template
awx workflow_job_templates create \
  --name "Backup and Validate Pipeline" \
  --organization "${ORG_ID}" \
  --description "Run backup, review, then validate"

WF_ID=$(awx workflow_job_templates list \
  --name "Backup and Validate Pipeline" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

# Node 1: Backup job template
awx workflow_job_template_nodes create \
  --workflow_job_template "${WF_ID}" \
  --unified_job_template "${JT_BACKUP_ID}"

NODE1_ID=$(awx workflow_job_template_nodes list \
  --workflow_job_template "${WF_ID}" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")

# Node 2: Approval gate (no unified_job_template = approval node)
awx workflow_job_template_nodes create \
  --workflow_job_template "${WF_ID}" \
  --approval_node "{
    \"name\": \"Review backup results\",
    \"description\": \"Backup complete. Review logs then approve to run validation.\",
    \"timeout\": 3600
  }"
# timeout: 3600 = approval expires after 1 hour if no one acts

NODE2_ID=$(awx workflow_job_template_nodes list \
  --workflow_job_template "${WF_ID}" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][-1]['id'])")

# Node 3: Validation job template
awx workflow_job_template_nodes create \
  --workflow_job_template "${WF_ID}" \
  --unified_job_template "${JT_VALIDATE_ID}"

NODE3_ID=$(awx workflow_job_template_nodes list \
  --workflow_job_template "${WF_ID}" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][-1]['id'])")

# Link nodes: Backup → (success) → Approval gate
awx workflow_job_template_nodes create_success_node \
  "${NODE1_ID}" --id "${NODE2_ID}"

# Link nodes: Approval gate → (approve) → Validate
awx workflow_job_template_nodes create_success_node \
  "${NODE2_ID}" --id "${NODE3_ID}"

# RBAC on the workflow: Engineers admin, NOC execute
awx workflow_job_templates grant "${WF_ID}" \
  --team "${TEAM_ENGINEERS}" --role admin
awx workflow_job_templates grant "${WF_ID}" \
  --team "${TEAM_NOC}" --role execute
```

**Approving the gate from the UI:**

When the workflow reaches the approval node it pauses. Any user with approval permission sees a banner in the AWX UI: *"Backup and Validate Pipeline is waiting for approval"*. They can click to view the backup job output, then Approve or Deny. Denial stops the workflow; approval continues to validation.

**Approving from the CLI or API:**

```bash
# List pending approvals
awx workflow_approvals list --status pending

# Approve (replace ID with actual approval ID)
awx workflow_approvals approve <approval-id>

# Deny
awx workflow_approvals deny <approval-id>
```

---

## 28.10 — Notifications

Notifications fire on job events: start, success, failure, or approval required. Configure them once and attach to any job template.

### Create a Slack Notification

**UI path:** Administration → Notifiers → Add

```
Name:            Slack — #network-automation
Organization:    Network Automation Lab
Type:            Slack
Token:           (Slack Bot OAuth token)
Destination channels: #network-automation
```

```bash
awx notification_templates create \
  --name "Slack — #network-automation" \
  --organization "${ORG_ID}" \
  --notification_type slack \
  --notification_configuration "{
    \"token\": \"xoxb-your-slack-bot-token\",
    \"channels\": [\"#network-automation\"]
  }"

NOTIF_ID=$(awx notification_templates list \
  --name "Slack — #network-automation" -f json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['id'])")
```

### Attach Notifications to Job Templates

```bash
# Backup template: notify on failure only (success is expected, failure is the signal)
awx job_templates notification_templates_error \
  "${JT_BACKUP_ID}" --id "${NOTIF_ID}"

# Validation template: notify on both success and failure
awx job_templates notification_templates_success \
  "${JT_VALIDATE_ID}" --id "${NOTIF_ID}"
awx job_templates notification_templates_error \
  "${JT_VALIDATE_ID}" --id "${NOTIF_ID}"

# Workflow: notify on approval required (tells engineers to act)
awx workflow_job_templates notification_templates_approvals \
  "${WF_ID}" --id "${NOTIF_ID}"
awx workflow_job_templates notification_templates_error \
  "${WF_ID}" --id "${NOTIF_ID}"
```

---

## 28.11 — Job History and Output

Every job run is stored in AWX with full stdout, input variables, inventory used, credentials used, and the user who launched it. This is the audit trail that cron cannot provide.

### Browsing Job History in the UI

**UI path:** Views → Jobs

The jobs list shows:
- Job ID, name, status (green/red), start time, duration
- Who launched it (user name or "Schedule")
- Which inventory and which limit was used

Clicking any job opens the full stdout output with ANSI color, searchable, downloadable.

### Querying Job History from the CLI

```bash
# List last 10 jobs
awx jobs list --order_by -started --page_size 10

# List all failed jobs in the last 24 hours
awx jobs list \
  --status failed \
  --started__gt "$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)"

# Get stdout of a specific job
awx jobs stdout 42

# Get job details including who launched it
awx jobs get 42 -f json \
  | python3 -c "
import sys, json
j = json.load(sys.stdin)
print(f\"Job: {j['name']}\")
print(f\"Status: {j['status']}\")
print(f\"Started: {j['started']}\")
print(f\"Launched by: {j.get('summary_fields', {}).get('launched_by', {}).get('name', 'schedule')}\")
print(f\"Inventory: {j.get('summary_fields', {}).get('inventory', {}).get('name', 'N/A')}\")
print(f\"Limit: {j.get('limit', 'all')}\")
"
```

---

## 28.12 — AWX API Basics

Every action in the AWX UI is available via the REST API at `/api/v2/`. This enables programmatic job triggering from CI/CD pipelines, ITSM tools, and scripts.

### Authentication

```bash
# Token authentication (preferred for automation)
curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"username": "alice", "password": "AlicePassword#2025"}' \
  http://172.16.0.10:30080/api/v2/tokens/ \
  | python3 -m json.tool | grep '"token"'
# Returns a token string — use in subsequent requests

# Store it
AWX_TOKEN="your-token-here"
AWX_URL="http://172.16.0.10:30080"
```

### Trigger a Job via API

```bash
# Launch a job template by ID
curl -s -X POST \
  -H "Authorization: Bearer ${AWX_TOKEN}" \
  -H "Content-Type: application/json" \
  "${AWX_URL}/api/v2/job_templates/${JT_BACKUP_ID}/launch/" \
  | python3 -m json.tool | grep '"id"'
# Returns the job ID

# Launch with extra variables and limit
curl -s -X POST \
  -H "Authorization: Bearer ${AWX_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"limit\": \"wan-r1\",
    \"extra_vars\": {\"validate_fail_on_error\": false}
  }" \
  "${AWX_URL}/api/v2/job_templates/${JT_DEPLOY_ID}/launch/"
```

### Poll Job Status

```bash
JOB_ID=42

# Poll until complete
while true; do
  STATUS=$(curl -s \
    -H "Authorization: Bearer ${AWX_TOKEN}" \
    "${AWX_URL}/api/v2/jobs/${JOB_ID}/" \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  echo "Job ${JOB_ID} status: ${STATUS}"
  if [[ "${STATUS}" == "successful" || "${STATUS}" == "failed" || "${STATUS}" == "error" ]]; then
    break
  fi
  sleep 10
done

echo "Final status: ${STATUS}"
```

### CI/CD Integration Pattern

The most common use of the AWX API is triggering deployment jobs from a CI/CD pipeline when playbooks are merged to the main branch:

```yaml
# .github/workflows/deploy.yml (GitHub Actions example)
name: Deploy on merge to main

on:
  push:
    branches: [main]
    paths:
      - 'playbooks/**'
      - 'inventory/**'
      - 'roles/**'

jobs:
  trigger-awx:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger AWX backup job
        run: |
          curl -s -X POST \
            -H "Authorization: Bearer ${{ secrets.AWX_TOKEN }}" \
            -H "Content-Type: application/json" \
            "${{ secrets.AWX_URL }}/api/v2/job_templates/${{ secrets.JT_BACKUP_ID }}/launch/"
```

---



AWX is now fully operational — organizations, teams, RBAC, credentials, SCM-backed projects, dynamic Netbox inventory, job templates, scheduled runs, a workflow with an approval gate, Slack notifications, and API triggering. The full guide is now complete: 28 parts from Ubuntu control node setup through enterprise automation management.

