---
draft: false
title: '32- Testing CI/CD'
description: "Part 32 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 32
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 32: Testing & CI/CD for Network Playbooks

> *A playbook that only gets tested when it runs in production isn't tested — it's gambled with. The disciplines of software engineering apply to network automation: catch problems early, catch them automatically, and catch them before they affect a device anyone cares about. The pipeline built in this part runs on every Git push, catches syntax errors and style violations within seconds, blocks broken code from merging, and automatically triggers AWX deployment jobs when changes land on main. The cost of setting this up once is paid back the first time it catches a bug before a change window.*

---

## 32.1 — The Network Automation Pipeline

### Pipeline Diagram

```
Developer workstation
        │
        │  git commit
        │
        ▼
┌───────────────────┐
│   pre-commit      │  ← Runs locally before commit is accepted
│                   │    yamllint, ansible-lint, trailing whitespace,
│                   │    syntax-check on changed files
└────────┬──────────┘
         │ git push
         ▼
┌───────────────────┐
│   GitHub          │
│   Pull Request    │  ← Code review gate — human approval required
│                   │    CI checks must pass before merge is allowed
└────────┬──────────┘
         │ CI triggered on push/PR
         ▼
┌───────────────────────────────────────────┐
│   GitHub Actions — CI Workflow            │
│                                           │
│   Job 1: Lint                             │
│     ├── yamllint (all YAML files)         │
│     ├── ansible-lint (all playbooks)      │
│     └── FAIL → PR blocked, dev notified  │
│                                           │
│   Job 2: Syntax Check (needs: lint)       │
│     ├── ansible-playbook --syntax-check   │
│     │   (all playbooks, all platforms)    │
│     └── FAIL → PR blocked                │
│                                           │
│   Both pass → PR can be reviewed/merged  │
└────────┬──────────────────────────────────┘
         │ merge to main
         ▼
┌───────────────────┐
│   GitHub Actions  │  ← Triggered only on merge to main
│   Deploy Workflow │
│                   │
│   AWX API call →  │
│   Backup job      │
│   Validate job    │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   AWX             │  ← Runs the actual playbooks against devices
│   Job History     │    Full output logged, Slack notification
└───────────────────┘
```

### What Each Stage Enforces

```
pre-commit (local, before push):
  Catches:  Syntax errors, lint violations, trailing whitespace, large files
  Blocks:   The git commit itself — never reaches GitHub
  Speed:    2-10 seconds
  Who sees: Only the developer — private, no CI minutes consumed

GitHub Actions CI (on every push and PR):
  Catches:  Anything pre-commit missed, full project lint (not just changed files)
  Blocks:   The pull request merge via branch protection rules
  Speed:    2-5 minutes
  Who sees: The whole team — PR shows pass/fail status

GitHub Actions Deploy (on merge to main only):
  Does:     Triggers AWX to run backup and/or validation jobs
  Speed:    Depends on AWX job runtime
  Who sees: AWX job history, Slack notification

The three-layer approach means problems are caught at the cheapest
possible point — local before they consume CI minutes, CI before
they consume code review time, and AWX only runs clean code.
```

---

## 32.2 — pre-commit Hooks

`pre-commit` is a framework that runs a configured set of checks before every `git commit`. If any check fails, the commit is rejected — the code never reaches GitHub in a broken state.

### Installation

```bash
source ~/ansible-venv/bin/activate
pip install pre-commit --break-system-packages

# Verify
pre-commit --version
```

### The .pre-commit-config.yaml File

```bash
cat > ~/projects/ansible-network/.pre-commit-config.yaml << 'EOF'
---
# .pre-commit-config.yaml
# Install hooks: pre-commit install
# Run manually against all files: pre-commit run --all-files
# Update hook versions: pre-commit autoupdate
#
# Hooks run in order — fail fast on the first failure.
# Each hook runs only against staged files (git add) unless run with --all-files.

default_language_version:
  python: python3

repos:

  # ── General file hygiene ──────────────────────────────────────────
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
        # Removes trailing spaces that cause noisy git diffs
        args: [--markdown-linebreak-ext=md]

      - id: end-of-file-fixer
        # Ensures every file ends with a newline

      - id: check-yaml
        # Basic YAML syntax check — catches unclosed brackets, bad indentation
        # Faster than yamllint but less thorough — both run in this config
        args: [--unsafe]
        # --unsafe allows custom YAML tags (needed for some Ansible constructs)

      - id: check-added-large-files
        args: [--maxkb=500]
        # Blocks accidental commits of large binary files or config backups

      - id: check-merge-conflict
        # Catches un-resolved merge conflict markers (<<<<<<, =======, >>>>>>>)

      - id: detect-private-key
        # Blocks commits containing private key material
        # SSH private keys, PEM files, etc.
        # Note: ansible-vault encrypted strings are not flagged (they're encrypted)

      - id: no-commit-to-branch
        args: [--branch, main, --branch, master]
        # Prevents direct commits to main/master
        # All changes must go through a pull request

  # ── YAML linting ──────────────────────────────────────────────────
  - repo: https://github.com/adrienverge/yamllint
    rev: v1.35.1
    hooks:
      - id: yamllint
        args:
          - --config-file=.yamllint
          - --strict
        # Uses the project's .yamllint config (created in Part 22)
        # --strict treats warnings as errors — enforces consistent style
        files: \.(yml|yaml)$
        exclude: |
          (?x)^(
            collections/.*|          # Exclude installed collections
            backups/.*|              # Exclude config backups
            reports/.*|              # Exclude generated reports
            \.github/workflows/.*    # GitHub Actions YAML uses different conventions
          )$

  # ── Ansible linting ───────────────────────────────────────────────
  - repo: https://github.com/ansible/ansible-lint
    rev: v25.1.3
    hooks:
      - id: ansible-lint
        name: ansible-lint
        language: python
        entry: ansible-lint
        args:
          - --config-file=.ansible-lint
          - --fix=none    # Report violations, don't auto-fix (fix manually)
        files: \.(yml|yaml)$
        exclude: |
          (?x)^(
            molecule/.*|             # Molecule test files have different conventions
            collections/.*|          # Installed collections
            backups/.*|
            reports/.*
          )$
        additional_dependencies:
          - ansible-core>=2.17
        # ansible-lint checks FQCN, name casing, no-changed-when, and more
        # See .ansible-lint for full config (Part 22)

  # ── Ansible syntax check ──────────────────────────────────────────
  - repo: local
    hooks:
      - id: ansible-syntax-check
        name: Ansible syntax check (changed playbooks)
        language: system
        entry: bash -c '
          source ~/ansible-venv/bin/activate &&
          for f in "$@"; do
            echo "Syntax check: $f"
            ansible-playbook --syntax-check "$f" \
              -i inventory/hosts.yml \
              --vault-password-file .vault/lab.txt 2>&1 || exit 1
          done' --
        files: ^playbooks/.*\.(yml|yaml)$
        exclude: |
          (?x)^(
            playbooks/backup/.*|     # Backup playbooks need live devices
            playbooks/netbox/.*      # Netbox playbooks need Netbox running
          )$
        pass_filenames: true
        # Runs --syntax-check on each changed playbook file
        # Only runs against playbooks/ (not roles, inventory, group_vars)
        # Excluded: playbooks that reference external services not in CI

  # ── Secret scanning ───────────────────────────────────────────────
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        name: Detect hardcoded secrets
        args: [--baseline, .secrets.baseline]
        # Scans for hardcoded passwords, API keys, tokens
        # Vault-encrypted strings are excluded (they're not plaintext secrets)
        # Run 'detect-secrets scan > .secrets.baseline' to initialize
        exclude: |
          (?x)^(
            .*vault\.yml$|           # Vault files contain encrypted (not plaintext) secrets
            \.secrets\.baseline$     # The baseline itself
          )$
EOF
```

### Installing and Testing the Hooks

```bash
cd ~/projects/ansible-network

# Install the hooks into .git/hooks/
pre-commit install
# → pre-commit installed at .git/hooks/pre-commit

# Initialize the detect-secrets baseline (run once)
detect-secrets scan \
  --exclude-files '.*vault\.yml$' \
  --exclude-files '.*\.secrets\.baseline$' \
  > .secrets.baseline
git add .secrets.baseline

# Test all hooks against all files without making a commit
pre-commit run --all-files

# Expected output (clean project):
# Trim Trailing Whitespace.............................Passed
# Fix End of Files.....................................Passed
# Check Yaml...........................................Passed
# Check for added large files..........................Passed
# Check for merge conflicts............................Passed
# Detect Private Key...................................Passed
# Don't commit to branch...............................Passed
# yamllint.............................................Passed
# ansible-lint.........................................Passed
# Ansible syntax check (changed playbooks).............Passed
# Detect hardcoded secrets.............................Passed

# Test against a single file
pre-commit run --files playbooks/deploy/deploy_ios_ccna.yml

# Skip a hook for one commit (emergency use only — document why)
SKIP=ansible-lint git commit -m "wip: incomplete — skip lint"

# Update all hooks to latest versions
pre-commit autoupdate
# Review the version bumps in .pre-commit-config.yaml before committing
```

### What Each Hook Catches in Practice

```
trailing-whitespace:
  Catches: "  enabled: true   " (invisible spaces at line end)
  Why it matters: Git diffs highlight trailing whitespace as changes,
  making real changes hard to read

detect-private-key:
  Catches: -----BEGIN RSA PRIVATE KEY-----
  Catches: -----BEGIN OPENSSH PRIVATE KEY-----
  Does NOT catch: $ANSIBLE_VAULT;1.1;AES256 (encrypted — safe)

no-commit-to-branch:
  Catches: git commit on the main branch directly
  Forces: all changes go through PRs — prevents force-push accidents

ansible-lint:
  Catches (most common violations for this project):
    fqcn[action]: ios_config instead of cisco.ios.ios_config
    name[missing]: tasks without name: fields
    no-changed-when: command/shell tasks without changed_when
    yaml[truthy]: yes/no instead of true/false

detect-secrets:
  Catches: password: MyPlaintextPassword2025
  Does NOT catch: password: "{{ vault_password }}" (variable reference)
  Does NOT catch: $ANSIBLE_VAULT;1.1;AES256... (vault encrypted)
```

---

## 32.3 — GitHub Actions CI Workflow

The CI workflow runs on every push and every pull request. It has two jobs: `lint` and `syntax-check`. The `syntax-check` job only runs if `lint` passes — no point checking syntax of files that fail style rules.

### Repository Secrets Setup

Before creating the workflow, add these secrets in GitHub:

```
GitHub repo → Settings → Secrets and variables → Actions → New repository secret

AWX_URL         = http://your-awx-ip:30080
AWX_TOKEN       = (token from Part 28 — alice's token)
AWX_JT_BACKUP   = (job template ID for Network Backup — All Devices)
AWX_JT_VALIDATE = (job template ID for Network Validate — All Devices)
VAULT_PASSWORD  = (contents of .vault/lab.txt — the vault password)
SLACK_WEBHOOK   = (Slack incoming webhook URL for CI notifications)
```

### The CI Workflow File

```bash
mkdir -p ~/projects/ansible-network/.github/workflows

cat > ~/projects/ansible-network/.github/workflows/ci.yml << 'EOF'
---
# .github/workflows/ci.yml
# CI pipeline: lint → syntax-check → (on merge) AWX deploy trigger
#
# Triggers:
#   push:        all branches (lint + syntax-check)
#   pull_request: to main (lint + syntax-check, required for merge)
#   workflow_dispatch: manual trigger (full pipeline including AWX)

name: Network Automation CI

on:
  push:
    branches: ['**']
    paths:
      # Only run CI when automation files change — skip on docs-only changes
      - 'playbooks/**'
      - 'roles/**'
      - 'inventory/**'
      - 'collections/requirements.yml'
      - '.github/workflows/**'
      - '.yamllint'
      - '.ansible-lint'
  pull_request:
    branches: [main]
  workflow_dispatch:
    # Allows manual trigger from GitHub Actions tab

env:
  PYTHON_VERSION: "3.12"
  ANSIBLE_FORCE_COLOR: "1"    # Coloured output in CI logs
  ANSIBLE_HOST_KEY_CHECKING: "false"

jobs:

  # ════════════════════════════════════════════════════════════
  # JOB 1: Lint — yamllint + ansible-lint
  # ════════════════════════════════════════════════════════════
  lint:
    name: "Lint (yamllint + ansible-lint)"
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: "Checkout repository"
        uses: actions/checkout@v4
        with:
          fetch-depth: 0    # Full history needed for some lint rules

      - name: "Set up Python ${{ env.PYTHON_VERSION }}"
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip           # Cache pip downloads between runs

      - name: "Install Python dependencies"
        run: |
          pip install --upgrade pip
          pip install \
            ansible-core==2.17.1 \
            ansible-lint==25.1.3 \
            yamllint==1.35.1 \
            netaddr \
            jmespath \
            pynetbox

      - name: "Install Ansible collections"
        run: |
          ansible-galaxy collection install \
            -r collections/requirements.yml \
            --force
        # --force ensures pinned versions are used even if cached versions differ

      - name: "Run yamllint"
        run: |
          yamllint \
            --config-file .yamllint \
            --strict \
            --format github \
            playbooks/ roles/ inventory/ group_vars/ host_vars/ \
            collections/requirements.yml
        # --format github: outputs annotations that appear inline in PR diffs
        # Shows exactly which line failed and why

      - name: "Run ansible-lint"
        run: |
          ansible-lint \
            --config-file .ansible-lint \
            --format pep8 \
            --strict \
            playbooks/ roles/
        # --format pep8: GitHub understands this format and shows inline annotations
        # --strict: warnings are treated as errors
        env:
          ANSIBLE_VAULT_PASSWORD_FILE: ""    # No vault needed for lint

      - name: "Notify Slack on lint failure"
        if: failure()
        uses: slackapi/slack-github-action@v2
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            {
              "text": "❌ Lint failed on *${{ github.repository }}*\nBranch: `${{ github.ref_name }}`\nCommit: `${{ github.sha }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View details>"
            }

  # ════════════════════════════════════════════════════════════
  # JOB 2: Syntax Check — ansible-playbook --syntax-check
  # Only runs if lint passes
  # ════════════════════════════════════════════════════════════
  syntax-check:
    name: "Syntax check (all playbooks)"
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: lint    # Only runs if lint job passed

    steps:
      - name: "Checkout repository"
        uses: actions/checkout@v4

      - name: "Set up Python ${{ env.PYTHON_VERSION }}"
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip

      - name: "Install Python dependencies"
        run: |
          pip install --upgrade pip
          pip install \
            ansible-core==2.17.1 \
            netaddr \
            jmespath \
            pynetbox \
            ncclient \
            paramiko

      - name: "Install Ansible collections"
        run: |
          ansible-galaxy collection install \
            -r collections/requirements.yml \
            --force

      - name: "Write vault password file"
        run: |
          echo "${{ secrets.VAULT_PASSWORD }}" > /tmp/vault-password
          chmod 600 /tmp/vault-password
        # Vault password is needed for --syntax-check to parse vault-encrypted vars

      - name: "Syntax check — IOS/NX-OS playbooks"
        run: |
          find playbooks/ -name "*.yml" \
            ! -path "*/netbox/*" \
            ! -path "*/backup/restore*" \
            ! -name "change_workflow*" \
            | sort \
            | while read -r playbook; do
                echo "→ Checking: $playbook"
                ansible-playbook \
                  --syntax-check \
                  --vault-password-file /tmp/vault-password \
                  -i inventory/hosts.yml \
                  "$playbook" || {
                  echo "FAIL: $playbook"
                  exit 1
                }
              done
        env:
          ANSIBLE_ROLES_PATH: roles/
          ANSIBLE_COLLECTIONS_PATH: ~/.ansible/collections

      - name: "Verify collections/requirements.yml is valid"
        run: |
          ansible-galaxy collection install \
            -r collections/requirements.yml \
            --dry-run
        # Checks that all listed collections and versions actually exist
        # Catches typos in collection names before anyone runs the install

      - name: "Clean up vault password file"
        if: always()    # Always runs — even if syntax check failed
        run: rm -f /tmp/vault-password

      - name: "Notify Slack on syntax-check failure"
        if: failure()
        uses: slackapi/slack-github-action@v2
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            {
              "text": "❌ Syntax check failed on *${{ github.repository }}*\nBranch: `${{ github.ref_name }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View details>"
            }

  # ════════════════════════════════════════════════════════════
  # JOB 3: AWX Deploy Trigger
  # Only runs on merge to main (not on PRs or feature branches)
  # ════════════════════════════════════════════════════════════
  awx-deploy:
    name: "Trigger AWX backup and validation"
    runs-on: ubuntu-latest
    timeout-minutes: 5
    needs: [lint, syntax-check]
    # Only trigger AWX on a push to main (not on pull_request events)
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - name: "Trigger AWX backup job"
        id: backup_job
        run: |
          RESPONSE=$(curl -sf -X POST \
            -H "Authorization: Bearer ${{ secrets.AWX_TOKEN }}" \
            -H "Content-Type: application/json" \
            "${{ secrets.AWX_URL }}/api/v2/job_templates/${{ secrets.AWX_JT_BACKUP }}/launch/")

          JOB_ID=$(echo "$RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
          echo "backup_job_id=${JOB_ID}" >> "$GITHUB_OUTPUT"
          echo "Backup job launched: ID ${JOB_ID}"

      - name: "Trigger AWX validation job"
        id: validate_job
        run: |
          RESPONSE=$(curl -sf -X POST \
            -H "Authorization: Bearer ${{ secrets.AWX_TOKEN }}" \
            -H "Content-Type: application/json" \
            "${{ secrets.AWX_URL }}/api/v2/job_templates/${{ secrets.AWX_JT_VALIDATE }}/launch/")

          JOB_ID=$(echo "$RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
          echo "validate_job_id=${JOB_ID}" >> "$GITHUB_OUTPUT"
          echo "Validate job launched: ID ${JOB_ID}"

      - name: "Notify Slack — deploy triggered"
        uses: slackapi/slack-github-action@v2
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            {
              "text": "✅ Merge to main by *${{ github.actor }}*\nBackup job: `${{ steps.backup_job.outputs.backup_job_id }}`\nValidate job: `${{ steps.validate_job.outputs.validate_job_id }}`\n<${{ secrets.AWX_URL }}|View in AWX>"
            }
EOF
```

### Branch Protection Rules

For the CI to actually block broken PRs from merging, branch protection must be configured:

```
GitHub repo → Settings → Branches → Add rule

Branch name pattern: main

Enable:
  ✓ Require a pull request before merging
      Required approvals: 1
      ✓ Dismiss stale pull request approvals when new commits are pushed

  ✓ Require status checks to pass before merging
      ✓ Require branches to be up to date before merging
      Status checks that are required:
        Lint (yamllint + ansible-lint)      ← add these after the first CI run
        Syntax check (all playbooks)         ← appears in the dropdown once run

  ✓ Require conversation resolution before merging

  ✓ Do not allow bypassing the above settings
```

---

## 32.4 — Molecule for Network Role Testing (Brief)

Molecule is a testing framework for Ansible roles. It manages the lifecycle of a test environment — create, converge (run the role), verify, destroy — and works with multiple drivers (Docker, Podman, EC2, and community drivers including one for Containerlab).

### What Molecule Provides

```
Without Molecule:
  Test a role → SSH to a real device → run the role → manually check output
  Reset for next test → manually restore device config
  Result: slow, manual, leaves state on real devices

With Molecule:
  Test a role → Molecule creates a test instance (container or VM)
               → runs the role → runs automated verifier (Ansible assertions)
               → destroys the instance
  Result: fast, automated, repeatable, leaves nothing behind
```

### The Molecule Directory Structure

```bash
# Standard Molecule layout within a role
roles/network_validate/
├── molecule/
│   └── default/              ← test scenario named 'default'
│       ├── molecule.yml      ← driver config, platforms, test sequence
│       ├── converge.yml      ← playbook that runs the role being tested
│       ├── verify.yml        ← playbook that asserts the expected outcome
│       └── requirements.yml  ← collections needed for this test scenario
├── defaults/main.yml
├── tasks/main.yml
└── ...

# Install Molecule
pip install molecule molecule-plugins --break-system-packages

# Initialize a new scenario for an existing role
cd roles/network_validate
molecule init scenario default
```

### A Simple Molecule Scenario

```bash
# molecule/default/molecule.yml — using Docker (localhost, no real devices)
cat > roles/network_validate/molecule/default/molecule.yml << 'EOF'
---
dependency:
  name: galaxy
  options:
    requirements-file: molecule/default/requirements.yml

driver:
  name: docker

platforms:
  - name: test-control-node
    image: quay.io/ansible/creator-ee:latest
    # creator-ee has ansible and many collections pre-installed
    pre_build_image: true

provisioner:
  name: ansible
  inventory:
    hosts:
      all:
        hosts:
          test-control-node:
            ansible_connection: docker

verifier:
  name: ansible

lint: |
  set -e
  yamllint .
  ansible-lint
EOF

# molecule/default/converge.yml — run the role in test mode
cat > roles/network_validate/molecule/default/converge.yml << 'EOF'
---
- name: "Converge"
  hosts: all
  gather_facts: false

  vars:
    # Minimal variables needed for the role to run in test mode
    validate_interfaces: false    # Skip checks that need real devices
    validate_bgp: false
    validate_ospf: false
    validate_vlans: false
    validate_routing: false
    validate_compliance: true     # Only test compliance — works without devices
    compliance:
      required:
        - pattern: "service password-encryption"
          description: "Password encryption"
          severity: critical
      prohibited:
        - pattern: "transport input telnet"
          description: "No telnet"
          severity: critical

  roles:
    - role: network_validate
EOF

# molecule/default/verify.yml — assert the role behaved correctly
cat > roles/network_validate/molecule/default/verify.yml << 'EOF'
---
- name: "Verify"
  hosts: all
  gather_facts: false

  tasks:
    - name: "Verify compliance check produced results"
      ansible.builtin.assert:
        that:
          - _validate_results is defined
          - _validate_results | length > 0
        fail_msg: "Validation role produced no results"
        success_msg: "PASS: Validation role produced {{ _validate_results | length }} results"
EOF
```

### The Containerlab Driver Limitation

```
The Molecule Containerlab driver exists but has real constraints:
  - Requires Containerlab installed on the CI runner (not available on GitHub Actions hosted runners)
  - Requires privileged containers (security concern on shared CI)
  - Startup time for network device containers is 1-3 minutes per device
  - Total test time for a 4-device topology: 10-20 minutes

Practical approaches:
  Option 1: Use Docker/localhost for role logic tests (what Molecule does well)
            Run platform-specific tests manually in Containerlab before PR
  Option 2: Self-hosted GitHub Actions runner on the control node
            (has Containerlab + Docker — can run full topology tests)
  Option 3: Molecule for non-device roles (compliance, reporting, backup)
            Manual/AWX for device-specific role testing

For this lab: use Molecule for the validation and compliance roles
(they can be tested without real devices), and rely on the Containerlab
lab environment for full integration testing of IOS/NX-OS/Junos/PAN-OS roles.
```

---

## 32.5 — Putting It Together: Developer Workflow

With all three layers in place, the developer workflow is:

```bash
# 1. Create a feature branch
git checkout -b feature/add-ospf-authentication

# 2. Make changes
vim roles/network_validate/tasks/checks/compliance.yml

# 3. Stage changes
git add roles/network_validate/tasks/checks/compliance.yml

# 4. Commit — pre-commit hooks run automatically
git commit -m "feat: add OSPF authentication to compliance checks"

# If pre-commit fails:
# Trim Trailing Whitespace.......................Failed
# - hook id: trailing-whitespace
# - exit code: 1
# - files were modified by this hook
#
# Pre-commit fixed the trailing whitespace automatically (for this hook).
# Re-stage and re-commit:
git add roles/network_validate/tasks/checks/compliance.yml
git commit -m "feat: add OSPF authentication to compliance checks"
# This time pre-commit passes — commit accepted

# 5. Push to GitHub
git push origin feature/add-ospf-authentication

# GitHub Actions CI triggers:
# → Lint job starts (~2 min)
# → Syntax-check job starts after lint passes (~3 min)
# → Slack: no notification if both pass (only notifies on failure)

# 6. Open a Pull Request on GitHub
# PR shows: ✅ Lint passed  ✅ Syntax check passed
# Team member reviews and approves

# 7. Merge PR to main
# GitHub Actions deploy workflow triggers:
# → AWX backup job launched
# → AWX validate job launched
# → Slack: "✅ Merge to main by alice | Backup job: 47 | Validate job: 48"

# 8. Check AWX
# Both jobs show green in AWX job history
# Validation report in reports/validation/ shows all checks passed
```

---


Testing and CI/CD are complete — pre-commit catches problems before they leave the developer's machine, GitHub Actions enforces lint and syntax standards on every PR and blocks merge when they fail, and AWX automatically runs backup and validation jobs when clean code lands on main. Every layer of the network automation pipeline is now automated. The capstone brings it all together.

