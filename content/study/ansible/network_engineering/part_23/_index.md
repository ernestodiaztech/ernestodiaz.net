---
draft: true
title: '23 - Maintenance'
weight: 23
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.

## Part 23: Keeping Packages & Collections Up to Date

> *The control node running these playbooks has three independent layers of software that each need maintenance: Ubuntu system packages managed by apt, Python packages managed by pip inside a virtualenv, and Ansible collections managed by ansible-galaxy. Each layer has its own update cadence, its own failure modes when things go out of sync, and its own strategy for balancing stability against security. This part treats each domain separately, builds a version pinning strategy that survives real-world upgrade cycles, and creates two maintenance playbooks — one that safely reports what's outdated, and one that upgrades with appropriate gates.*

---

## 23.1 — The Three Update Domains

```
Ubuntu Control Node
├── Domain 1: OS packages (apt)
│     ├── linux kernel, openssl, libssl, curl, git, python3
│     ├── Update trigger: security advisories, Ubuntu LTS updates
│     └── Risk: kernel updates require reboot; openssl updates may break venv
│
├── Domain 2: Python packages (pip inside virtualenv)
│     ├── ansible-core, ansible, ncclient, paramiko, netmiko
│     ├── pan-python, pan-os-python, panos, napalm, jinja2, yamllint
│     ├── Update trigger: CVEs, new collection requirements, feature needs
│     └── Risk: ansible-core minor versions can break playbook syntax
│
└── Domain 3: Ansible collections (ansible-galaxy)
      ├── cisco.ios, cisco.nxos, junipernetworks.junos
      ├── paloaltonetworks.panos, ansible.netcommon, ansible.utils
      ├── Update trigger: new module support, bug fixes, NOS version compatibility
      └── Risk: module parameter renames between collection versions
```

These three domains are independent — updating one does not update the others. An `apt upgrade` that updates `libssl` doesn't update the Python `cryptography` package in the virtualenv that wraps it. An `ansible-galaxy collection install --upgrade` doesn't update `ansible-core`. Each domain is managed separately.

---

## 23.2 — Domain 1: OS Packages with apt

### Standard Update Commands

```bash
# Update package index (fetch current package lists from Ubuntu mirrors)
sudo apt update

# Show what would be upgraded without upgrading
apt list --upgradable 2>/dev/null | grep -v "Listing..."

# Upgrade all upgradable packages (non-interactive)
sudo apt upgrade -y

# Full upgrade — also handles packages that require removing others
# Use when apt upgrade says "kept back" packages
sudo apt full-upgrade -y

# Remove packages no longer needed
sudo apt autoremove -y

# Clean downloaded package files
sudo apt clean
```

### Security-Only Updates

For control nodes in production environments where stability matters more than feature updates, security-only patches are the right approach:

```bash
# Install only security updates
sudo apt install unattended-upgrades
sudo unattended-upgrade --dry-run -d    # Preview what would be patched
sudo unattended-upgrade -d              # Apply security patches

# Check which packages have pending security updates
sudo apt-get --just-print upgrade 2>&1 | grep -i security
```

### Critical OS Packages for the Ansible Control Node

These packages underpin the Python virtualenv and SSH connectivity — their versions matter:

```bash
# Check versions of critical dependencies
python3 --version          # Should be 3.10+ for modern collection support
openssl version            # Affects paramiko and ncclient TLS negotiation
git --version              # Required for ansible-galaxy and Git integration
ssh -V                     # OpenSSH version affects device connectivity

# Show which OS packages would affect the virtualenv if upgraded
apt show libssl-dev openssl libpython3-dev 2>/dev/null | grep Version
```

### When OS Updates Require a Virtualenv Rebuild

An `apt upgrade` that updates `python3` or `libssl` can invalidate symlinks inside the virtualenv:

```bash
# Test if virtualenv is still intact after apt upgrade
source ~/ansible-venv/bin/activate
python3 -c "import ansible; print(ansible.__version__)"
python3 -c "import paramiko; print(paramiko.__version__)"
python3 -c "import ncclient; print(ncclient.__version__)"

# If any import fails after an OS upgrade, rebuild from requirements.txt:
deactivate
rm -rf ~/ansible-venv
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
pip install -r ~/projects/ansible-network/requirements.txt
```

This is exactly why `requirements.txt` with pinned versions is essential — the rebuild produces an identical environment rather than whatever the latest versions happen to be on that day.

---

## 23.3 — Domain 2: Python Packages with pip

### Checking What's Outdated

```bash
source ~/ansible-venv/bin/activate

# List all outdated packages with current and latest versions
pip list --outdated

# Example output:
# Package          Version    Latest    Type
# ---------------- ---------- --------- -----
# ansible-core     2.16.3     2.17.1    wheel
# ansible          9.3.0      10.1.0    wheel
# paramiko         3.4.0      3.5.0     wheel
# ncclient         0.6.13     0.6.15    wheel
# cryptography     42.0.2     43.0.1    wheel

# JSON output for scripting
pip list --outdated --format=json | python3 -m json.tool

# Check a specific package
pip show ansible-core | grep -E "Name|Version"
pip index versions ansible-core    # Show all available versions
```

### Updating Ansible

```bash
# Update ansible-core only (the engine)
pip install --upgrade ansible-core

# Update the ansible package (includes collections bundled with ansible)
# Note: 'ansible' package depends on ansible-core — upgrading ansible upgrades both
pip install --upgrade ansible

# Update to a specific version (safer for production)
pip install "ansible-core==2.17.1"
pip install "ansible==10.1.0"

# After any ansible-core upgrade — verify syntax on all playbooks
find ~/projects/ansible-network/playbooks -name "*.yml" \
  -exec ansible-playbook {} --syntax-check \; 2>&1 | grep -E "ERROR|playbook:"
```

### The `pip install --upgrade` Risk

Not all package upgrades are safe. Breaking changes in `ansible-core` minor versions (2.16 → 2.17) have historically included:

- Deprecated module parameters becoming errors
- Changed behavior of Jinja2 filter handling
- New `changed_when` requirements for command tasks
- FQCN enforcement becoming mandatory

The safe upgrade process is: upgrade in a test environment, run `ansible-lint` and `--syntax-check`, run playbooks in `--check` mode, only then upgrade the production control node.

---

## 23.4 — Version Pinning Strategy

### Why Pin Versions

An unpinned `requirements.txt` with `ansible` and `paramiko` will install different versions depending on when the virtualenv is built. A playbook that works today on ansible-core 2.16.3 may fail next month when ansible-core 2.17 introduces a breaking change — or more insidiously, produce different results without obvious errors.

Pinned versions solve the reproducibility problem: the same `requirements.txt` produces the same environment regardless of when it's run.

### The `requirements.txt` File

```bash
cat > ~/projects/ansible-network/requirements.txt << 'EOF'
# =============================================================
# requirements.txt — Python dependencies for ansible-network
# Pin ALL versions. Update intentionally, not accidentally.
#
# Update process:
#   1. Copy this file to requirements.test.txt
#   2. Relax pins in requirements.test.txt (remove ==, use >=)
#   3. pip install -r requirements.test.txt in a TEST venv
#   4. Run full validation pipeline against the test venv
#   5. pip freeze > requirements.txt.new to capture exact new versions
#   6. Review diff, update this file, commit
#
# Last reviewed: 2025-01-15
# Next review due: 2025-04-15 (quarterly)
# =============================================================

# ── Ansible core ──────────────────────────────────────────────────
ansible-core==2.16.13
ansible==9.13.0

# ── Network vendor libraries ──────────────────────────────────────
paramiko==3.4.0              # SSH library (used by network_cli)
ncclient==0.6.15             # NETCONF library (used by Junos)
netmiko==4.3.0               # High-level SSH for network devices

# ── Palo Alto ─────────────────────────────────────────────────────
pan-python==0.17.0
pan-os-python==1.11.1

# ── Utility libraries ─────────────────────────────────────────────
jinja2==3.1.4                # Template engine (ansible dependency)
cryptography==42.0.5         # TLS/crypto (paramiko/ncclient dependency)
cffi==1.16.0                 # C foreign function interface (cryptography dep)
pycparser==2.22              # C parser (cffi dependency)

# ── Validation and linting tools ──────────────────────────────────
ansible-lint==24.2.3
yamllint==1.35.1
pip-audit==2.7.3

# ── Ansible utility modules ───────────────────────────────────────
ansible-pylibssh==1.1.0      # libssh-based SSH (faster than paramiko on some platforms)
resolvelib==1.0.1            # Dependency resolver (ansible-galaxy requirement)

# ── Parsing and output ────────────────────────────────────────────
textfsm==1.1.3               # TextFSM for structured output parsing
ntc-templates==5.1.0         # TextFSM templates for show command parsing
rich==13.7.1                 # Terminal formatting for reports
EOF
```

### The `collections/requirements.yml` File

```bash
mkdir -p ~/projects/ansible-network/collections

cat > ~/projects/ansible-network/collections/requirements.yml << 'EOF'
---
# =============================================================
# collections/requirements.yml — Ansible collection dependencies
# Pin ALL versions. Update intentionally, not accidentally.
#
# Install: ansible-galaxy collection install -r collections/requirements.yml
# Upgrade: ansible-galaxy collection install -r collections/requirements.yml --upgrade
#
# Collection storage: ~/.ansible/collections (default)
# Or project-local: set collections_paths in ansible.cfg
#
# Last reviewed: 2025-01-15
# Next review due: 2025-04-15 (quarterly)
# =============================================================

collections:

  # ── Core Ansible collections ───────────────────────────────────
  - name: ansible.netcommon
    version: "==6.1.3"       # Cross-platform network connection plugins

  - name: ansible.utils
    version: "==4.1.0"       # Network utility filters (ipaddr, etc.)

  # ── Cisco collections ──────────────────────────────────────────
  - name: cisco.ios
    version: "==9.0.3"       # IOS/IOS-XE resource modules

  - name: cisco.nxos
    version: "==9.2.1"       # NX-OS resource modules

  - name: cisco.iosxr
    version: "==10.2.2"      # IOS-XR (included for completeness)

  # ── Juniper collection ─────────────────────────────────────────
  - name: junipernetworks.junos
    version: "==8.0.0"       # Junos NETCONF modules

  # ── Palo Alto collection ───────────────────────────────────────
  - name: paloaltonetworks.panos
    version: "==2.20.0"      # PAN-OS XML API modules

  # ── General purpose collections ───────────────────────────────
  - name: community.general
    version: "==9.4.0"       # General modules including mail, slack, etc.

  - name: ansible.posix
    version: "==1.5.4"       # POSIX system modules
EOF
```

### Installing from Requirements Files

```bash
# Clean install of all Python dependencies
source ~/ansible-venv/bin/activate
pip install -r ~/projects/ansible-network/requirements.txt

# Install all Ansible collections
ansible-galaxy collection install \
  -r ~/projects/ansible-network/collections/requirements.yml

# Verify installed versions match requirements
pip list | grep -E "ansible|paramiko|ncclient|netmiko|pan"
ansible-galaxy collection list | grep -E "cisco|juniper|palo|ansible.net"
```

### The Safe Upgrade Process

Testing upgrades safely before committing new pins:

```bash
#!/bin/bash
# scripts/test_upgrade.sh — Safely test dependency upgrades in isolation
# Usage: ./scripts/test_upgrade.sh

set -e

echo "════════════════════════════════════════════════════"
echo " Safe Upgrade Test Process"
echo "════════════════════════════════════════════════════"

# Step 1: Create a clean test virtualenv (doesn't touch production venv)
echo ""
echo "── Step 1: Create test virtualenv ──────────────────"
python3 -m venv ~/ansible-venv-test
source ~/ansible-venv-test/bin/activate

# Step 2: Create relaxed requirements for testing (remove == pins)
echo ""
echo "── Step 2: Generate relaxed requirements ────────────"
sed 's/==[0-9].*//' ~/projects/ansible-network/requirements.txt \
    | sed 's/==[0-9].*//' > /tmp/requirements.test.txt

echo "Relaxed requirements:"
cat /tmp/requirements.test.txt

# Step 3: Install latest versions into test venv
echo ""
echo "── Step 3: Install latest versions ─────────────────"
pip install -r /tmp/requirements.test.txt

# Step 4: Show what changed
echo ""
echo "── Step 4: Version comparison ───────────────────────"
echo "CURRENT (production venv) | TEST (latest versions)"
source ~/ansible-venv/bin/activate
CURRENT=$(pip list --format=json)

source ~/ansible-venv-test/bin/activate
TEST=$(pip list --format=json)

# Compare key packages
for pkg in ansible-core ansible paramiko ncclient netmiko cryptography; do
    CURRENT_VER=$(echo "$CURRENT" | python3 -c "
import sys, json
data = json.load(sys.stdin)
pkg = [p for p in data if p['name'].lower() == '$pkg']
print(pkg[0]['version'] if pkg else 'not installed')
")
    TEST_VER=$(echo "$TEST" | python3 -c "
import sys, json
data = json.load(sys.stdin)
pkg = [p for p in data if p['name'].lower() == '$pkg']
print(pkg[0]['version'] if pkg else 'not installed')
")
    if [ "$CURRENT_VER" != "$TEST_VER" ]; then
        echo "  CHANGED: $pkg: $CURRENT_VER → $TEST_VER"
    else
        echo "  same:    $pkg: $CURRENT_VER"
    fi
done

# Step 5: Run validation pipeline against test venv
echo ""
echo "── Step 5: Run pre-change checks with test venv ─────"
cd ~/projects/ansible-network
ansible-playbook playbooks/deploy/deploy_ios_ccna.yml --syntax-check
ansible-lint playbooks/deploy/deploy_ios_ccna.yml
echo "  ✓ Syntax and lint checks passed with new versions"

# Step 6: Capture new pinned versions
echo ""
echo "── Step 6: Capture new pinned versions ──────────────"
pip freeze > /tmp/requirements.new.txt
echo "New pinned requirements saved to /tmp/requirements.new.txt"
echo ""
echo "Review the diff and update requirements.txt if satisfied:"
echo "  diff ~/projects/ansible-network/requirements.txt /tmp/requirements.new.txt"
echo "  cp /tmp/requirements.new.txt ~/projects/ansible-network/requirements.txt"

# Clean up test venv
deactivate
echo ""
echo "Test venv at ~/ansible-venv-test — remove when done:"
echo "  rm -rf ~/ansible-venv-test"

echo ""
echo "════════════════════════════════════════════════════"
echo " Upgrade test complete. Review diffs before committing."
echo "════════════════════════════════════════════════════"
```

```bash
chmod +x ~/projects/ansible-network/scripts/test_upgrade.sh
```

---

## 23.5 — Domain 3: Ansible Collections with ansible-galaxy

### Checking Installed Collection Versions

```bash
# List all installed collections with versions
ansible-galaxy collection list

# Example output:
# Collection                    Version
# ----------------------------- -------
# ansible.netcommon             6.1.3
# ansible.utils                 4.1.0
# cisco.ios                     9.0.3
# cisco.nxos                    9.2.1
# junipernetworks.junos         8.0.0
# paloaltonetworks.panos        2.20.0

# Check if a specific collection has an update available
ansible-galaxy collection install cisco.ios --dry-run 2>&1 | grep -E "cisco.ios|Nothing"

# List all collections with available upgrades (no built-in command — use this)
ansible-galaxy collection list | tail -n +3 | while read col ver; do
    LATEST=$(ansible-galaxy collection install "$col" --dry-run 2>&1 | \
             grep "Downloading" | grep -oP '\d+\.\d+\.\d+' | head -1)
    if [ -n "$LATEST" ] && [ "$LATEST" != "$ver" ]; then
        echo "UPDATE AVAILABLE: $col $ver → $LATEST"
    fi
done
```

### Upgrading Collections

```bash
# Upgrade a single collection
ansible-galaxy collection install cisco.ios --upgrade

# Upgrade all collections from requirements.yml
ansible-galaxy collection install \
  -r ~/projects/ansible-network/collections/requirements.yml \
  --upgrade

# Upgrade to a specific version
ansible-galaxy collection install "cisco.ios:==9.1.0"

# Force reinstall of exact pinned versions (reset to known state)
ansible-galaxy collection install \
  -r ~/projects/ansible-network/collections/requirements.yml \
  --force
```

### Collection Upgrade Risks

Collection upgrades are more likely to break playbooks than Python package upgrades. Module parameters get renamed, deprecated parameters become errors, and new required parameters appear. Always check the collection changelog before upgrading:

```bash
# Find the changelog for an installed collection
find ~/.ansible/collections -name "CHANGELOG*" -path "*/cisco/ios/*"
# Or check the collection's GitHub releases page

# Test collection upgrade without committing
# Temporarily install new version to a different path
ansible-galaxy collection install cisco.ios:==9.1.0 \
  --collections-path /tmp/test-collections

# Run playbooks pointing to the test collection path
ANSIBLE_COLLECTIONS_PATH=/tmp/test-collections:~/.ansible/collections \
  ansible-playbook playbooks/deploy/deploy_ios_ccna.yml \
  --check --limit wan-r1
```

---

## 23.6 — The Reporting Playbook

Safe to run at any time — checks all three domains and reports status without making changes.

```bash
cat > ~/projects/ansible-network/playbooks/maintenance/check_updates.yml << 'EOF'
---
# =============================================================
# check_updates.yml — Check for updates across all three domains
# Safe to run at any time — makes NO changes
# Produces a report showing what's outdated
#
# Usage: ansible-playbook playbooks/maintenance/check_updates.yml
# =============================================================

- name: "Maintenance | Check for updates across all domains"
  hosts: localhost
  gather_facts: true    # Needed for ansible_date_time and ansible_os_family

  vars:
    report_dir: "reports/maintenance"
    report_timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"
    venv_path: "{{ ansible_env.HOME }}/ansible-venv"
    project_path: "{{ ansible_env.HOME }}/projects/ansible-network"

  tasks:

    # ── Setup ──────────────────────────────────────────────────
    - name: "Setup | Create report directory"
      ansible.builtin.file:
        path: "{{ report_dir }}"
        state: directory
        mode: '0755'

    # ── Domain 1: OS packages ──────────────────────────────────
    - name: "OS | Update apt package cache"
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600    # Only update if cache is older than 1 hour
      become: true
      changed_when: false

    - name: "OS | Get list of upgradable packages"
      ansible.builtin.command:
        cmd: apt list --upgradable
      register: apt_upgradable
      changed_when: false
      become: false

    - name: "OS | Parse upgradable packages"
      ansible.builtin.set_fact:
        os_upgradable_packages: >-
          {{ apt_upgradable.stdout_lines
             | reject('match', '^Listing.*')
             | reject('equalto', '')
             | list }}

    - name: "OS | Get security-specific upgrades"
      ansible.builtin.shell:
        cmd: >
          apt-get --just-print upgrade 2>&1
          | grep "^Inst"
          | grep -i security
          | awk '{print $2}'
      register: security_upgrades
      changed_when: false
      become: false
      failed_when: false    # grep returns 1 when no matches

    - name: "OS | Display upgrade summary"
      ansible.builtin.debug:
        msg:
          - "OS packages upgradable: {{ os_upgradable_packages | length }}"
          - "Security updates pending: {{ security_upgrades.stdout_lines | length }}"
        verbosity: 0

    # ── Domain 2: Python packages ──────────────────────────────
    - name: "Python | Get outdated pip packages"
      ansible.builtin.command:
        cmd: "{{ venv_path }}/bin/pip list --outdated --format=json"
      register: pip_outdated_raw
      changed_when: false
      failed_when: false

    - name: "Python | Parse outdated packages"
      ansible.builtin.set_fact:
        pip_outdated: "{{ pip_outdated_raw.stdout | from_json }}"

    - name: "Python | Get current ansible-core version"
      ansible.builtin.command:
        cmd: "{{ venv_path }}/bin/ansible --version"
      register: ansible_version_raw
      changed_when: false

    - name: "Python | Display pip outdated summary"
      ansible.builtin.debug:
        msg:
          - "Python packages outdated: {{ pip_outdated | length }}"
          - "Outdated packages: {{ pip_outdated | map(attribute='name') | list | join(', ') }}"
        verbosity: 0

    # ── Domain 2b: pip-audit security scan ────────────────────
    - name: "Security | Run pip-audit vulnerability scan"
      ansible.builtin.command:
        cmd: "{{ venv_path }}/bin/pip-audit --format=json --output=/tmp/pip-audit-results.json"
      register: pip_audit_raw
      changed_when: false
      failed_when: pip_audit_raw.rc not in [0, 1]
      # pip-audit returns rc=1 when vulnerabilities are found (not an error)

    - name: "Security | Load pip-audit results"
      ansible.builtin.slurp:
        src: /tmp/pip-audit-results.json
      register: pip_audit_file
      failed_when: false

    - name: "Security | Parse pip-audit results"
      ansible.builtin.set_fact:
        pip_audit_results: "{{ pip_audit_file.content | b64decode | from_json }}"
      when: pip_audit_file is not failed

    - name: "Security | Summarize vulnerabilities by severity"
      ansible.builtin.set_fact:
        vuln_packages: >-
          {{ pip_audit_results.dependencies | default([])
             | selectattr('vulns', 'defined')
             | selectattr('vulns', 'ne', [])
             | list }}
      when: pip_audit_results is defined

    - name: "Security | Display vulnerability summary"
      ansible.builtin.debug:
        msg:
          - "Packages with CVEs: {{ vuln_packages | default([]) | length }}"
          - >-
            {% for pkg in vuln_packages | default([]) %}
            {{ pkg.name }} {{ pkg.version }}: {{ pkg.vulns | length }} CVE(s)
            —  {{ pkg.vulns | map(attribute='id') | list | join(', ') }}
            {% endfor %}
      when: vuln_packages is defined and vuln_packages | length > 0

    - name: "Security | Confirm no CVEs found"
      ansible.builtin.debug:
        msg: "No known CVEs in installed Python packages ✓"
      when: vuln_packages is defined and vuln_packages | length == 0

    # ── Domain 3: Ansible collections ─────────────────────────
    - name: "Collections | Get installed collection versions"
      ansible.builtin.command:
        cmd: ansible-galaxy collection list --format json
      register: collections_raw
      changed_when: false
      environment:
        PATH: "{{ venv_path }}/bin:{{ ansible_env.PATH }}"

    - name: "Collections | Parse installed collections"
      ansible.builtin.set_fact:
        installed_collections: "{{ collections_raw.stdout | from_json }}"

    - name: "Collections | Read pinned versions from requirements.yml"
      ansible.builtin.slurp:
        src: "{{ project_path }}/collections/requirements.yml"
      register: collections_req_file

    - name: "Collections | Parse requirements.yml"
      ansible.builtin.set_fact:
        collections_requirements: "{{ collections_req_file.content | b64decode | from_yaml }}"

    - name: "Collections | Check each required collection is installed at pinned version"
      ansible.builtin.set_fact:
        collection_status: >-
          {{ collection_status | default([]) + [{
            'name': item.name,
            'required': item.version | replace('==', ''),
            'installed': (installed_collections.values()
                          | map('dict2items')
                          | flatten
                          | selectattr('key', 'equalto', item.name | replace('.', '/'))
                          | map(attribute='value.version')
                          | first) | default('not installed'),
            'status': 'OK' if (installed_collections.values()
                               | map('dict2items')
                               | flatten
                               | selectattr('key', 'defined')
                               | map(attribute='value.version')
                               | first | default('')) == (item.version | replace('==', ''))
                          else 'MISMATCH'
          }] }}
      loop: "{{ collections_requirements.collections | default([]) }}"
      loop_control:
        label: "{{ item.name }}"

    - name: "Collections | Display collection status"
      ansible.builtin.debug:
        msg: >-
          Collections status:
          {% for col in collection_status | default([]) %}
          [{{ col.status }}] {{ col.name }}: required={{ col.required }} installed={{ col.installed }}
          {% endfor %}

    # ── Generate consolidated report ───────────────────────────
    - name: "Report | Write maintenance report"
      ansible.builtin.copy:
        content: |
          Network Automation Maintenance Report
          ======================================
          Generated: {{ lookup('pipe', 'date') }}
          Host: {{ inventory_hostname }}

          DOMAIN 1: OS PACKAGES (apt)
          ----------------------------
          Upgradable packages: {{ os_upgradable_packages | length }}
          Security updates pending: {{ security_upgrades.stdout_lines | length }}
          {% if security_upgrades.stdout_lines | length > 0 %}
          Security packages:
          {% for pkg in security_upgrades.stdout_lines %}
            - {{ pkg }}
          {% endfor %}
          {% endif %}
          {% if os_upgradable_packages | length > 0 %}
          All upgradable:
          {% for pkg in os_upgradable_packages %}
            - {{ pkg }}
          {% endfor %}
          {% endif %}

          DOMAIN 2: PYTHON PACKAGES (pip)
          ---------------------------------
          Outdated packages: {{ pip_outdated | length }}
          {% for pkg in pip_outdated %}
            - {{ pkg.name }}: {{ pkg.version }} → {{ pkg.latest_version }}
          {% endfor %}

          SECURITY SCAN (pip-audit)
          -------------------------
          Packages with CVEs: {{ vuln_packages | default([]) | length }}
          {% for pkg in vuln_packages | default([]) %}
            ⚠ {{ pkg.name }} {{ pkg.version }}:
          {% for vuln in pkg.vulns %}
              - {{ vuln.id }}: {{ vuln.description | default('No description') | truncate(80) }}
                Fix: upgrade to {{ vuln.fix_versions | join(' or ') | default('no fix available') }}
          {% endfor %}
          {% endfor %}
          {% if vuln_packages | default([]) | length == 0 %}
          No known CVEs found ✓
          {% endif %}

          DOMAIN 3: ANSIBLE COLLECTIONS
          --------------------------------
          {% for col in collection_status | default([]) %}
          [{{ col.status }}] {{ col.name }}: required={{ col.required }} installed={{ col.installed }}
          {% endfor %}

          RECOMMENDED ACTIONS
          -------------------
          {% if security_upgrades.stdout_lines | length > 0 %}
          ⚠ URGENT: Apply security OS updates:
            sudo apt upgrade {{ security_upgrades.stdout_lines | join(' ') }}
          {% endif %}
          {% for pkg in vuln_packages | default([]) %}
          ⚠ CVE: Upgrade {{ pkg.name }} to fix {{ pkg.vulns | map(attribute='id') | list | join(', ') }}
          {% endfor %}
          {% if pip_outdated | length > 0 %}
          • Python packages to review: {{ pip_outdated | map(attribute='name') | list | join(', ') }}
            Run: ./scripts/test_upgrade.sh before upgrading
          {% endif %}
          {% if os_upgradable_packages | length > 0 and security_upgrades.stdout_lines | length == 0 %}
          • {{ os_upgradable_packages | length }} non-security OS updates available
          {% endif %}
        dest: "{{ report_dir }}/maintenance_{{ report_timestamp }}.txt"
        mode: '0644'

    - name: "Report | Display report location"
      ansible.builtin.debug:
        msg:
          - "Maintenance report saved to: {{ report_dir }}/maintenance_{{ report_timestamp }}.txt"
          - "View with: cat {{ report_dir }}/maintenance_{{ report_timestamp }}.txt"
EOF
```

---

## 23.7 — The Upgrade Playbook

Gated behind explicit variable confirmation — can't run accidentally.

```bash
cat > ~/projects/ansible-network/playbooks/maintenance/apply_updates.yml << 'EOF'
---
# =============================================================
# apply_updates.yml — Apply updates across domains
# GATED: must pass confirmation variables on command line
# Makes changes — run check_updates.yml first and review output
#
# Usage examples:
#   OS security only:
#     ansible-playbook apply_updates.yml -e "upgrade_os_security=true"
#
#   All OS packages:
#     ansible-playbook apply_updates.yml -e "upgrade_os_all=true"
#
#   Python packages (runs test_upgrade.sh first):
#     ansible-playbook apply_updates.yml -e "upgrade_python=true"
#
#   Collections:
#     ansible-playbook apply_updates.yml -e "upgrade_collections=true"
#
#   All domains:
#     ansible-playbook apply_updates.yml \
#       -e "upgrade_os_security=true upgrade_python=true upgrade_collections=true"
# =============================================================

- name: "Maintenance | Apply updates (gated)"
  hosts: localhost
  gather_facts: true

  vars:
    # Gates — must be explicitly set true on command line
    # Defaults are all false — running the playbook without -e does nothing
    upgrade_os_security: false
    upgrade_os_all: false
    upgrade_python: false
    upgrade_collections: false

    venv_path: "{{ ansible_env.HOME }}/ansible-venv"
    project_path: "{{ ansible_env.HOME }}/projects/ansible-network"

  tasks:

    # ── Pre-flight: confirm something was actually requested ───
    - name: "Gate | Verify at least one upgrade domain was requested"
      ansible.builtin.assert:
        that:
          - upgrade_os_security | bool
            or upgrade_os_all | bool
            or upgrade_python | bool
            or upgrade_collections | bool
        fail_msg: |
          No upgrade domain specified. Use -e to set one or more of:
            upgrade_os_security=true
            upgrade_os_all=true
            upgrade_python=true
            upgrade_collections=true

    - name: "Gate | Display upgrade plan"
      ansible.builtin.debug:
        msg:
          - "═══════════════════════════════════════════════"
          - " UPGRADE PLAN"
          - "═══════════════════════════════════════════════"
          - " OS security updates: {{ upgrade_os_security }}"
          - " All OS updates:      {{ upgrade_os_all }}"
          - " Python packages:     {{ upgrade_python }}"
          - " Collections:         {{ upgrade_collections }}"
          - "═══════════════════════════════════════════════"

    - name: "Gate | Pause for final confirmation"
      ansible.builtin.pause:
        prompt: "Review the plan above. Press ENTER to proceed or Ctrl+C then A to abort"

    # ── Domain 1: OS security updates ─────────────────────────
    - name: "OS | Apply security updates only"
      ansible.builtin.apt:
        upgrade: dist
        update_cache: true
        default_release: "{{ ansible_distribution_release }}-security"
      become: true
      register: security_upgrade_result
      when: upgrade_os_security | bool and not upgrade_os_all | bool

    - name: "OS | Apply all OS updates"
      ansible.builtin.apt:
        upgrade: full
        update_cache: true
        autoremove: true
      become: true
      register: full_upgrade_result
      when: upgrade_os_all | bool

    - name: "OS | Check if reboot is required"
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required
      when: upgrade_os_security | bool or upgrade_os_all | bool

    - name: "OS | Warn if reboot is needed"
      ansible.builtin.debug:
        msg: |
          ⚠ REBOOT REQUIRED
          Kernel or critical library was updated.
          Schedule a reboot at a maintenance window:
            sudo reboot
          After reboot, verify virtualenv:
            source ~/ansible-venv/bin/activate
            python3 -c "import ansible; print(ansible.__version__)"
      when:
        - reboot_required is defined
        - reboot_required.stat.exists | default(false)

    # ── Domain 2: Python packages ──────────────────────────────
    - name: "Python | Run upgrade test script before applying"
      ansible.builtin.command:
        cmd: "{{ project_path }}/scripts/test_upgrade.sh"
      register: upgrade_test_result
      changed_when: false
      when: upgrade_python | bool

    - name: "Python | Display upgrade test results"
      ansible.builtin.debug:
        msg: "{{ upgrade_test_result.stdout_lines }}"
      when: upgrade_python | bool

    - name: "Python | Pause to review upgrade test output"
      ansible.builtin.pause:
        prompt: |
          Review the version changes above.
          The test virtualenv has been validated.
          Press ENTER to apply upgrades to production venv, or Ctrl+C then A to abort
      when: upgrade_python | bool

    - name: "Python | Apply Python package upgrades"
      ansible.builtin.pip:
        requirements: "{{ project_path }}/requirements.txt"
        virtualenv: "{{ venv_path }}"
        state: latest
        extra_args: "--upgrade"
      when: upgrade_python | bool
      register: pip_upgrade_result

    - name: "Python | Capture new pinned versions after upgrade"
      ansible.builtin.command:
        cmd: "{{ venv_path }}/bin/pip freeze"
      register: pip_freeze_new
      changed_when: false
      when: upgrade_python | bool

    - name: "Python | Save new requirements.txt"
      ansible.builtin.copy:
        content: |
          # =============================================================
          # requirements.txt — Updated {{ lookup('pipe', 'date') }}
          # Previous version backed up to requirements.txt.bak
          # =============================================================
          {{ pip_freeze_new.stdout }}
        dest: "{{ project_path }}/requirements.txt"
        backup: true    # Creates requirements.txt.bak before overwriting
        mode: '0644'
      when: upgrade_python | bool

    - name: "Python | Run syntax check after upgrade"
      ansible.builtin.command:
        cmd: "{{ venv_path }}/bin/ansible-playbook {{ item }} --syntax-check"
      loop: "{{ lookup('fileglob', project_path + '/playbooks/deploy/*.yml', wantlist=true) }}"
      loop_control:
        label: "{{ item | basename }}"
      changed_when: false
      when: upgrade_python | bool

    - name: "Python | Run ansible-lint after upgrade"
      ansible.builtin.command:
        cmd: "{{ venv_path }}/bin/ansible-lint {{ project_path }}/playbooks/deploy/"
      changed_when: false
      register: lint_result
      failed_when: lint_result.rc not in [0, 2]
      when: upgrade_python | bool

    # ── Domain 3: Collections ──────────────────────────────────
    - name: "Collections | Upgrade from requirements.yml"
      ansible.builtin.command:
        cmd: >
          {{ venv_path }}/bin/ansible-galaxy collection install
          -r {{ project_path }}/collections/requirements.yml
          --upgrade
      register: collection_upgrade_result
      changed_when: "'Installing' in collection_upgrade_result.stdout"
      when: upgrade_collections | bool

    - name: "Collections | Display upgrade output"
      ansible.builtin.debug:
        msg: "{{ collection_upgrade_result.stdout_lines }}"
      when: upgrade_collections | bool

    - name: "Collections | Update requirements.yml with new versions"
      ansible.builtin.command:
        cmd: "{{ venv_path }}/bin/ansible-galaxy collection list --format json"
      register: new_collection_versions
      changed_when: false
      when: upgrade_collections | bool

    - name: "Collections | Run syntax check with new collections"
      ansible.builtin.command:
        cmd: "{{ venv_path }}/bin/ansible-playbook {{ item }} --syntax-check"
      loop: "{{ lookup('fileglob', project_path + '/playbooks/deploy/*.yml', wantlist=true) }}"
      loop_control:
        label: "{{ item | basename }}"
      changed_when: false
      when: upgrade_collections | bool

    # ── Post-upgrade summary ───────────────────────────────────
    - name: "Summary | Display post-upgrade status"
      ansible.builtin.debug:
        msg:
          - "═══════════════════════════════════════════════"
          - " UPGRADE COMPLETE"
          - "═══════════════════════════════════════════════"
          - " OS security: {{ 'applied' if upgrade_os_security | bool else 'skipped' }}"
          - " OS all:      {{ 'applied' if upgrade_os_all | bool else 'skipped' }}"
          - " Python:      {{ 'applied' if upgrade_python | bool else 'skipped' }}"
          - " Collections: {{ 'applied' if upgrade_collections | bool else 'skipped' }}"
          - "───────────────────────────────────────────────"
          - " Next steps:"
          - "   1. git diff requirements.txt collections/requirements.yml"
          - "   2. git add -A && git commit -m 'chore: update dependencies'"
          - "   3. Run validation: ansible-playbook playbooks/validate/validate_network.yml"
          - "═══════════════════════════════════════════════"
EOF
```

---

## 23.8 — `pip-audit` Deep Dive

`pip-audit` queries the Python Packaging Advisory Database (PyPA advisory DB) and the OSV database for known CVEs in installed packages. It's the security scan layer that `pip list --outdated` doesn't provide — a package can be on the latest version and still have an unpatched CVE.

```bash
# Install
pip install pip-audit --break-system-packages

# Basic scan — all installed packages in current venv
pip-audit

# Example output with a vulnerability:
# Name          Version  ID                  Fix Versions
# ------------- -------- ------------------- -----------------------------------------------
# cryptography  38.0.4   GHSA-w7pp-m8wf-vj6r 39.0.1
# requests      2.27.1   CVE-2023-32681      2.31.0

# JSON output for programmatic processing
pip-audit --format=json | python3 -m json.tool

# Scan only the packages in requirements.txt (not full venv)
pip-audit -r requirements.txt

# Generate a full report including packages without CVEs
pip-audit --format=json --output=pip-audit-report.json

# Fix automatically — upgrades vulnerable packages in place
# Use with caution — may break pinned requirements
pip-audit --fix
```

### Understanding CVE Severity

When `pip-audit` reports a CVE, the GHSA (GitHub Security Advisory) or CVE record contains a CVSS score indicating severity:

```
CVSS Score  Severity    Typical Action
──────────  ──────────  ──────────────────────────────────────────────
9.0 – 10.0  Critical    Patch immediately — do not wait for change window
7.0 – 8.9   High        Patch within 24-72 hours
4.0 – 6.9   Medium      Patch within next maintenance window
0.1 – 3.9   Low         Patch on next scheduled review cycle
```

### Acting on pip-audit Findings

```bash
# For each CVE found:
# 1. Look up the advisory
pip-audit --format=json | python3 -c "
import sys, json
data = json.load(sys.stdin)
for dep in data.get('dependencies', []):
    for vuln in dep.get('vulns', []):
        print(f\"{dep['name']} {dep['version']}: {vuln['id']}\")
        print(f\"  Description: {vuln.get('description', 'N/A')[:120]}\")
        print(f\"  Fix versions: {vuln.get('fix_versions', ['none'])}\")
        print()
"

# 2. Check if the fix version is compatible with the rest of the stack
pip install "cryptography==41.0.0" --dry-run 2>&1 | grep -E "ERROR|Would install"

# 3. Update requirements.txt with the fixed version
# Edit the file manually to preserve structure and comments

# 4. Run test_upgrade.sh to verify compatibility
./scripts/test_upgrade.sh

# 5. Apply and verify
ansible-playbook playbooks/maintenance/apply_updates.yml -e "upgrade_python=true"
```

---

## 23.9 — Maintenance Schedule

Recommended cadence for the three update domains:

```
Daily (automated via cron):
  pip-audit --format=json --output reports/maintenance/audit_$(date +%Y%m%d).json
  # Alert on any Critical or High CVEs

Weekly:
  ansible-playbook playbooks/maintenance/check_updates.yml
  # Review report, act on security updates

Monthly:
  apt update && apt list --upgradable
  # Apply OS security updates during maintenance window
  # Review Python and collection update candidates

Quarterly:
  ./scripts/test_upgrade.sh
  # Test full upgrade of all Python packages
  # Review collection changelogs for breaking changes
  # Update pinned versions in requirements.txt and requirements.yml
  # Update "Last reviewed" and "Next review due" dates in both files
```

### Cron Setup for Automated Security Scanning

```bash
# Add to crontab: daily pip-audit scan at 3am
crontab -e

# Add this line:
0 3 * * * /bin/bash -c 'source ~/ansible-venv/bin/activate && pip-audit --format=json --output ~/projects/ansible-network/reports/maintenance/audit_$(date +\%Y\%m\%d).json 2>&1'

# Weekly maintenance report at 6am every Monday
0 6 * * 1 /bin/bash -c 'source ~/ansible-venv/bin/activate && cd ~/projects/ansible-network && ansible-playbook playbooks/maintenance/check_updates.yml >> logs/maintenance.log 2>&1'
```

---


The maintenance system is complete — separate reporting and upgrade playbooks for all three dependency domains, version pinning with a safe upgrade test process, pip-audit integrated for CVE scanning, and a maintenance schedule. Part 24 moves to the final operational topic: using Ansible for network documentation and topology discovery — automatically generating network diagrams and inventory reports from live device data.


