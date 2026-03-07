---
draft: false
title: '22 - Validation'
description: "Part 22 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 22
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 22: Validating Configurations & Playbooks

> *Deployment without validation is hope-based operations. A playbook that runs without errors doesn't mean the network is in the correct state — it means Ansible didn't encounter any Python exceptions. The BGP session might still be down. The VLAN might exist but be in err-disabled state. The ACL might have been applied to the wrong interface. Validation is the discipline of proving the network is in the state you intended, and it belongs in the pipeline before changes are made and after they land.*

---

## 22.1 — The Full Validation Pipeline

Every change in the lab follows this pipeline:

```
Stage 1 — Pre-flight (control node, no device contact)
  ├── yamllint         → YAML syntax and style
  ├── ansible --syntax-check → Playbook structure
  └── ansible-lint     → Best practices and common errors

Stage 2 — Dry run (device contact, no changes)
  ├── ansible-playbook --check  → What would change
  └── ansible-playbook --diff   → Exact config differences

Stage 3 — Deployment
  └── ansible-playbook          → Apply changes

Stage 4 — Post-deployment verification
  ├── Interface state checks
  ├── Protocol neighbor checks (BGP, OSPF)
  ├── VLAN and routing table checks
  └── Structured pass/fail report
```

Stages 1 and 2 catch problems before they touch the network. Stage 4 proves the network is behaving correctly after changes land. This part builds all four stages.

---

## 22.2 — Stage 1: Pre-flight Checks on the Control Node

### YAML Linting with `yamllint`

`yamllint` checks YAML syntax and style — indentation, trailing whitespace, line length, and document structure. It catches problems that look valid to the eye but cause parse errors at runtime.

```bash
# Install
pip install yamllint --break-system-packages

# Run against a single file
yamllint playbooks/deploy/deploy_ios_ccna.yml

# Run against the entire project
yamllint ~/projects/ansible-network/

# With a specific config file
yamllint -c .yamllint ~/projects/ansible-network/
```

Create a project-level `.yamllint` config that fits network automation files (which tend to have long lines for commands and descriptions):

```bash
cat > ~/projects/ansible-network/.yamllint << 'EOF'
---
extends: default

rules:
  line-length:
    max: 160           # Network commands and descriptions are often long
    level: warning     # Warn, don't fail

  comments:
    min-spaces-from-content: 1   # Allow single-space comments

  truthy:
    allowed-values: ['true', 'false', 'yes', 'no']
    check-keys: false  # Don't flag YAML keys that look like booleans

  braces:
    min-spaces-inside: 0
    max-spaces-inside: 1

  brackets:
    min-spaces-inside: 0
    max-spaces-inside: 1
EOF
```

**Common `yamllint` errors and fixes:**

```yaml
# ❌ Trailing whitespace (invisible — yamllint catches it)
ansible_host: 172.16.0.11   
# ✅ Fix: remove trailing spaces (most editors can do this on save)

# ❌ Wrong indentation (2 spaces expected, 4 used)
tasks:
    - name: "task"        # 4-space indent
# ✅ Fix:
tasks:
  - name: "task"          # 2-space indent

# ❌ Missing document start marker
- name: "My play"
# ✅ Fix:
---
- name: "My play"

# ❌ Duplicate key
vars:
  timeout: 30
  timeout: 60             # Duplicate — second value silently wins
# ✅ Fix: remove the duplicate
```

### Ansible Syntax Check

```bash
# Check playbook structure without running
ansible-playbook playbooks/deploy/deploy_ios_ccna.yml --syntax-check

# Check all playbooks at once
find playbooks/ -name "*.yml" -exec \
  ansible-playbook {} --syntax-check \;

# Expected output on success:
# playbook: playbooks/deploy/deploy_ios_ccna.yml

# Example failure output:
# ERROR! no action detected in task. This often indicates a misspelled module name.
# The error appears to be in 'playbooks/deploy/deploy_ios_ccna.yml': line 47
```

`--syntax-check` catches: missing required keys, invalid module names, malformed variable references, broken `when:` conditions, and structural errors in `block/rescue/always`. It does not run any tasks or connect to any device.

### `ansible-lint` — Best Practices Enforcement

`ansible-lint` enforces a rule set that goes beyond syntax — it catches patterns that are syntactically valid but operationally problematic. The approach here is to see violations appear as the playbooks are built and fix them on the spot rather than treating them as a separate audit step.

```bash
# Install
pip install ansible-lint --break-system-packages

# Run against a playbook
ansible-lint playbooks/deploy/deploy_ios_ccna.yml

# Run against the entire project
cd ~/projects/ansible-network
ansible-lint

# Run with a specific profile
ansible-lint --profile production playbooks/deploy/deploy_ios_ccna.yml
```

#### Common Violations and Fixes

**`yaml[truthy]` — Use explicit boolean values**

```yaml
# ❌ Violation: implicit truthy
shutdown: yes
enabled: no

# ✅ Fix: explicit boolean
shutdown: true
enabled: false
```

**`name[missing]` — All tasks must have names**

```yaml
# ❌ Violation: unnamed task
- cisco.ios.ios_config:
    lines:
      - "ntp server 8.8.8.8"

# ✅ Fix: add descriptive name
- name: "NTP | Configure primary NTP server"
  cisco.ios.ios_config:
    lines:
      - "ntp server 8.8.8.8"
```

**`name[casing]` — Task names should start with uppercase or a tag prefix**

```yaml
# ❌ Violation: lowercase start
- name: "configure ntp server"

# ✅ Fix: capitalize or use tag|description pattern
- name: "NTP | Configure primary NTP server"
- name: "Configure NTP server"
```

**`fqcn[action]` — Use fully qualified collection names**

```yaml
# ❌ Violation: short module name
- name: "Configure NTP"
  ios_config:
    lines:
      - "ntp server 8.8.8.8"

# ✅ Fix: fully qualified name
- name: "NTP | Configure NTP server"
  cisco.ios.ios_config:
    lines:
      - "ntp server 8.8.8.8"
```

**`no-changed-when` — Commands that don't change state should say so**

```yaml
# ❌ Violation: show command without changed_when
- name: "Verify | Check OSPF neighbors"
  cisco.ios.ios_command:
    commands: [show ip ospf neighbor]
  register: ospf_output

# ✅ Fix: add changed_when: false for read-only commands
- name: "Verify | Check OSPF neighbors"
  cisco.ios.ios_command:
    commands: [show ip ospf neighbor]
  register: ospf_output
  changed_when: false
```

**`risky-shell-option` — Shell tasks need explicit options**

```yaml
# ❌ Violation: shell without set -o pipefail
- name: "Local | Get timestamp"
  ansible.builtin.shell: date +%Y%m%d | tr -d '\n'
  delegate_to: localhost

# ✅ Fix: use command module when no pipe needed, or add pipefail
- name: "Local | Get timestamp"
  ansible.builtin.command: date +%Y%m%d_%H%M%S
  delegate_to: localhost
  changed_when: false
```

**`var-naming[no-role-prefix]` — Role variables should have a role prefix**

```yaml
# ❌ Violation (in a role): generic variable name
vars:
  timeout: 30

# ✅ Fix: prefix with role name
vars:
  ios_base_timeout: 30
```

#### `.ansible-lint` Configuration File

```bash
cat > ~/projects/ansible-network/.ansible-lint << 'EOF'
---
profile: moderate   # min | basic | moderate | safety | shared | production

# Rules to skip (with justification)
skip_list:
  - yaml[line-length]      # Managed by yamllint with higher limit
  - no-free-form           # ios_command free-form syntax is intentional

# Paths to exclude
exclude_paths:
  - .git/
  - rendered/             # Generated files — not linted
  - backups/              # Backup configs — not linted

# Treat these as warnings instead of errors
warn_list:
  - experimental

# Require changed_when on all command/shell tasks
task_name_prefix: ""    # No forced prefix — we use our own tag|description pattern
EOF
```

---

## 22.3 — Stage 2: Dry Run with `--check` and `--diff`

### `--check` Mode

`--check` connects to devices, reads current state, and reports what would change — without making any changes. It's a prediction, not a guarantee (some modules handle check mode better than others), but it's the best preview available.

```bash
# Preview what deploy_ios_ccna.yml would change on wan-r1
ansible-playbook playbooks/deploy/deploy_ios_ccna.yml \
  --check \
  --limit wan-r1

# Preview with verbose output to see exactly which tasks would change
ansible-playbook playbooks/deploy/deploy_ios_ccna.yml \
  --check \
  --limit wan-r1 \
  -v
```

Expected output for a task that would make a change:
```
TASK [Interfaces | Configure L3 interface IP addresses]
changed: [wan-r1] => (item=GigabitEthernet1 — 10.10.10.1/30)
```

Expected output for a task that's already correct:
```
TASK [Interfaces | Configure L3 interface IP addresses]
ok: [wan-r1] => (item=GigabitEthernet1 — 10.10.10.1/30)
```

### `--diff` Mode

`--diff` shows the exact lines that would be added or removed. Combined with `--check`, it gives a complete picture of what the playbook would do:

```bash
# Show diffs without making changes
ansible-playbook playbooks/deploy/deploy_ios_ccna.yml \
  --check \
  --diff \
  --limit wan-r1

# Diff output format (for ios_config tasks):
# --- before
# +++ after
# @@ -0,0 +1 @@
# +ntp server 216.239.35.0
# +ntp server 216.239.35.4
```

### The Pre-Change Checklist Script

Combining stages 1 and 2 into a single pre-change script:

```bash
cat > ~/projects/ansible-network/scripts/pre_change_check.sh << 'EOF'
#!/bin/bash
# pre_change_check.sh — Run all pre-change validation checks
# Usage: ./scripts/pre_change_check.sh [playbook] [--limit host]
# Example: ./scripts/pre_change_check.sh playbooks/deploy/deploy_ios_ccna.yml --limit wan-r1

set -e    # Exit on any failure

PLAYBOOK="${1:-playbooks/deploy/deploy_ios_ccna.yml}"
EXTRA_ARGS="${@:2}"    # All arguments after the playbook

echo "════════════════════════════════════════════════════"
echo " Pre-Change Validation Pipeline"
echo " Playbook: ${PLAYBOOK}"
echo " Args: ${EXTRA_ARGS}"
echo "════════════════════════════════════════════════════"

# Stage 1a: YAML lint
echo ""
echo "── Stage 1a: YAML lint ──────────────────────────────"
yamllint "${PLAYBOOK}"
echo "  ✓ YAML syntax clean"

# Stage 1b: Ansible syntax check
echo ""
echo "── Stage 1b: Ansible syntax check ──────────────────"
ansible-playbook "${PLAYBOOK}" --syntax-check
echo "  ✓ Ansible syntax clean"

# Stage 1c: ansible-lint
echo ""
echo "── Stage 1c: ansible-lint ───────────────────────────"
ansible-lint "${PLAYBOOK}"
echo "  ✓ ansible-lint clean"

# Stage 2: Dry run with diff
echo ""
echo "── Stage 2: Dry run (--check --diff) ────────────────"
ansible-playbook "${PLAYBOOK}" --check --diff ${EXTRA_ARGS}

echo ""
echo "════════════════════════════════════════════════════"
echo " All pre-change checks passed."
echo " Review the diff output above, then run:"
echo " ansible-playbook ${PLAYBOOK} ${EXTRA_ARGS}"
echo "════════════════════════════════════════════════════"
EOF
chmod +x ~/projects/ansible-network/scripts/pre_change_check.sh
```

---

## 22.4 — Stage 4: The Validation Role

A reusable validation role that runs after deployment, checks network state against expected values from the data model, and produces a structured pass/fail report. This role is platform-aware — shared logic for common checks, platform-specific tasks where the commands differ.

### Role Structure

```bash
mkdir -p ~/projects/ansible-network/roles/network_validate/{tasks,templates,defaults,vars}

# Create the role skeleton
cat > ~/projects/ansible-network/roles/network_validate/defaults/main.yml << 'EOF'
---
# network_validate role defaults
# Override these in group_vars or host_vars as needed

validate_interfaces: true      # Check interface up/up state
validate_bgp: true             # Check BGP neighbor state
validate_ospf: true            # Check OSPF neighbor state
validate_vlans: true           # Check VLAN existence and active state
validate_routing: true         # Check routing table for expected prefixes

# Report settings
validate_report_dir: "reports/validation"
validate_report_format: text   # text | json
validate_fail_on_error: true   # Fail the play if any check fails
                               # Set false to collect all results before failing
EOF
```

### Role Variables

```bash
cat > ~/projects/ansible-network/roles/network_validate/vars/main.yml << 'EOF'
---
# Internal role constants — not overridable
_validate_results: []          # Accumulated results — populated during role run
_validate_timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"
EOF
```

### Main Task Entry Point

```bash
cat > ~/projects/ansible-network/roles/network_validate/tasks/main.yml << 'EOF'
---
# network_validate/tasks/main.yml
# Orchestrates all validation checks based on platform and enabled checks

- name: "Validate | Initialize results accumulator"
  ansible.builtin.set_fact:
    _validate_results: []
    _validate_start_time: "{{ lookup('pipe', 'date') }}"

- name: "Validate | Create report directory"
  ansible.builtin.file:
    path: "{{ validate_report_dir }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: false

# ── Platform detection ─────────────────────────────────────────
- name: "Validate | Detect platform"
  ansible.builtin.set_fact:
    _validate_platform: >-
      {{ 'ios' if ansible_network_os | default('') in
         ['cisco.ios.ios', 'ios'] else
         'nxos' if ansible_network_os | default('') in
         ['cisco.nxos.nxos', 'nxos'] else
         'junos' if ansible_network_os | default('') in
         ['junipernetworks.junos.junos', 'junos'] else
         'unknown' }}

# ── Include check modules ──────────────────────────────────────
- name: "Validate | Run interface checks"
  ansible.builtin.include_tasks: "checks/interfaces.yml"
  when: validate_interfaces | bool

- name: "Validate | Run BGP checks"
  ansible.builtin.include_tasks: "checks/bgp.yml"
  when: validate_bgp | bool and bgp is defined

- name: "Validate | Run OSPF checks"
  ansible.builtin.include_tasks: "checks/ospf.yml"
  when: validate_ospf | bool and (ospf is defined or ospf_advanced is defined)

- name: "Validate | Run VLAN checks"
  ansible.builtin.include_tasks: "checks/vlans.yml"
  when: validate_vlans | bool and (vlans is defined or base_vlans is defined)

- name: "Validate | Run routing table checks"
  ansible.builtin.include_tasks: "checks/routing.yml"
  when: validate_routing | bool

# ── Generate report ────────────────────────────────────────────
- name: "Validate | Generate validation report"
  ansible.builtin.include_tasks: "reporting/generate_report.yml"
EOF
```

### Interface Validation Check

```bash
mkdir -p ~/projects/ansible-network/roles/network_validate/tasks/checks

cat > ~/projects/ansible-network/roles/network_validate/tasks/checks/interfaces.yml << 'EOF'
---
# checks/interfaces.yml — Interface state validation (cross-platform)

# ── IOS / IOS-XE ───────────────────────────────────────────────
- name: "Validate | IOS: gather interface states"
  cisco.ios.ios_command:
    commands:
      - show interfaces summary
      - show ip interface brief
  register: _ios_iface_raw
  changed_when: false
  when: _validate_platform == 'ios'

- name: "Validate | IOS: gather structured interface facts"
  cisco.ios.ios_facts:
    gather_subset: [interfaces]
  register: _ios_iface_facts
  when: _validate_platform == 'ios'

- name: "Validate | IOS: check expected interfaces are up/up"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'interface_state',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'interface': item.key,
        'expected': 'up',
        'actual': _ios_iface_facts.ansible_facts.ansible_net_interfaces[item.key].operstatus
                  | default('not found'),
        'result': 'PASS' if _ios_iface_facts.ansible_facts.ansible_net_interfaces[item.key].operstatus
                            | default('') == 'up' else 'FAIL',
        'detail': 'Interface ' + item.key + ' is ' +
                  _ios_iface_facts.ansible_facts.ansible_net_interfaces[item.key].operstatus
                  | default('not found')
      }] }}
  loop: "{{ interfaces | dict2items | selectattr('value.shutdown', 'defined')
            | selectattr('value.shutdown', 'equalto', false) | list }}"
  loop_control:
    label: "{{ item.key }}"
  when: _validate_platform == 'ios' and interfaces is defined

# ── NX-OS ─────────────────────────────────────────────────────
- name: "Validate | NX-OS: gather interface facts"
  cisco.nxos.nxos_facts:
    gather_subset: [interfaces]
  register: _nxos_iface_facts
  when: _validate_platform == 'nxos'

- name: "Validate | NX-OS: check expected interfaces are up/up"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'interface_state',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'interface': item.key,
        'expected': 'up',
        'actual': _nxos_iface_facts.ansible_facts.ansible_net_interfaces[item.key].operstatus
                  | default('not found'),
        'result': 'PASS' if _nxos_iface_facts.ansible_facts.ansible_net_interfaces[item.key].operstatus
                            | default('') == 'up' else 'FAIL',
        'detail': 'Interface ' + item.key + ' is ' +
                  _nxos_iface_facts.ansible_facts.ansible_net_interfaces[item.key].operstatus
                  | default('not found')
      }] }}
  loop: "{{ interfaces | default({}) | dict2items }}"
  loop_control:
    label: "{{ item.key }}"
  when: _validate_platform == 'nxos' and interfaces is defined

# ── Junos ─────────────────────────────────────────────────────
- name: "Validate | Junos: get interface state"
  junipernetworks.junos.junos_command:
    commands:
      - show interfaces terse
    display: text
  register: _junos_iface_raw
  changed_when: false
  when: _validate_platform == 'junos'

- name: "Validate | Junos: check expected interfaces are up"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'interface_state',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'interface': item.key,
        'expected': 'up',
        'actual': 'up' if item.key in _junos_iface_raw.stdout[0] and
                  'up' in (_junos_iface_raw.stdout[0]
                           | regex_search(item.key + r'\s+up', multiline=True) | default(''))
                  else 'down/not found',
        'result': 'PASS' if item.key in _junos_iface_raw.stdout[0] and
                  'up' in (_junos_iface_raw.stdout[0]
                           | regex_search(item.key + r'\s+up', multiline=True) | default(''))
                  else 'FAIL',
        'detail': 'Interface ' + item.key + ' state check'
      }] }}
  loop: "{{ interfaces | dict2items | selectattr('value.shutdown', 'defined')
            | selectattr('value.shutdown', 'equalto', false) | list }}"
  loop_control:
    label: "{{ item.key }}"
  when: _validate_platform == 'junos' and interfaces is defined
EOF
```

### BGP Validation Check

```bash
cat > ~/projects/ansible-network/roles/network_validate/tasks/checks/bgp.yml << 'EOF'
---
# checks/bgp.yml — BGP neighbor state validation

# ── IOS ────────────────────────────────────────────────────────
- name: "Validate | IOS: get BGP summary"
  cisco.ios.ios_command:
    commands:
      - show ip bgp summary
  register: _ios_bgp_raw
  changed_when: false
  when: _validate_platform == 'ios'

- name: "Validate | IOS: check each BGP neighbor is Established"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'bgp_neighbor',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'neighbor': item.ip,
        'expected': 'Established',
        'actual': 'Established' if item.ip in _ios_bgp_raw.stdout[0] and
                  _ios_bgp_raw.stdout[0] | regex_search(item.ip + r'.*\d+$', multiline=True)
                  | default('') != '' else 'Not established',
        'result': 'PASS' if item.ip in _ios_bgp_raw.stdout[0] and
                  _ios_bgp_raw.stdout[0] | regex_search(item.ip + r'\s+\d+\s+\d+\s+\d+.*\d+$',
                  multiline=True) | default('') != '' else 'FAIL',
        'detail': 'BGP neighbor ' + item.ip + ' (' + item.description | default('') + ')'
      }] }}
  loop: "{{ bgp.neighbors | default(bgp_advanced.neighbors | default([])) }}"
  loop_control:
    label: "BGP neighbor {{ item.ip }}"
  when: _validate_platform == 'ios'

# ── NX-OS ─────────────────────────────────────────────────────
- name: "Validate | NX-OS: get BGP summary"
  cisco.nxos.nxos_command:
    commands:
      - show bgp ipv4 unicast summary
  register: _nxos_bgp_raw
  changed_when: false
  when: _validate_platform == 'nxos'

- name: "Validate | NX-OS: check each BGP neighbor is Established"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'bgp_neighbor',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'neighbor': item.ip,
        'expected': 'Established',
        'actual': 'Established' if item.ip in _nxos_bgp_raw.stdout[0] and
                  _nxos_bgp_raw.stdout[0] | regex_search(item.ip + r'.*\s+\d+$', multiline=True)
                  | default('') != '' else 'Not established',
        'result': 'PASS' if item.ip in _nxos_bgp_raw.stdout[0] and
                  _nxos_bgp_raw.stdout[0] | regex_search(item.ip + r'\s+\d+\s+\d+.*\s+\d+$',
                  multiline=True) | default('') != '' else 'FAIL',
        'detail': 'BGP neighbor ' + item.ip + ' (' + item.description | default('') + ')'
      }] }}
  loop: "{{ bgp.neighbors | default([]) }}"
  loop_control:
    label: "BGP neighbor {{ item.ip }}"
  when: _validate_platform == 'nxos'

# ── Junos ─────────────────────────────────────────────────────
- name: "Validate | Junos: get BGP summary"
  junipernetworks.junos.junos_command:
    commands:
      - show bgp neighbor
    display: text
  register: _junos_bgp_raw
  changed_when: false
  when: _validate_platform == 'junos'

- name: "Validate | Junos: check each BGP neighbor is Established"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'bgp_neighbor',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'neighbor': item.ip,
        'expected': 'Established',
        'actual': 'Established' if 'Peer: ' + item.ip in _junos_bgp_raw.stdout[0] and
                  'Established' in (_junos_bgp_raw.stdout[0]
                  | regex_search('Peer: ' + item.ip + r'.*\n.*Type', multiline=True) | default(''))
                  else 'Not established',
        'result': 'PASS' if 'Peer: ' + item.ip in _junos_bgp_raw.stdout[0] and
                  'Established' in _junos_bgp_raw.stdout[0] else 'FAIL',
        'detail': 'BGP neighbor ' + item.ip + ' (' + item.description | default('') + ')'
      }] }}
  loop: "{{ bgp.groups | default([]) | map(attribute='neighbors') | flatten }}"
  loop_control:
    label: "BGP neighbor {{ item.ip }}"
  when: _validate_platform == 'junos'
EOF
```

### OSPF Validation Check

```bash
cat > ~/projects/ansible-network/roles/network_validate/tasks/checks/ospf.yml << 'EOF'
---
# checks/ospf.yml — OSPF neighbor state validation

- name: "Validate | IOS: get OSPF neighbors"
  cisco.ios.ios_command:
    commands:
      - show ip ospf neighbor
  register: _ios_ospf_raw
  changed_when: false
  when: _validate_platform == 'ios'

- name: "Validate | IOS: check OSPF neighbors are FULL"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'ospf_neighbor',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'interface': item.name,
        'expected': 'FULL',
        'actual': 'FULL' if 'FULL' in _ios_ospf_raw.stdout[0] else 'No FULL adjacency',
        'result': 'PASS' if 'FULL' in _ios_ospf_raw.stdout[0] else 'FAIL',
        'detail': 'OSPF on ' + item.name + ' (area ' +
                  (ospf.areas | selectattr('interfaces', 'defined')
                   | map(attribute='area_id') | first | string) + ')'
      }] }}
  loop: "{{ (ospf.areas | default([]) + ospf_advanced.areas | default([]))
            | map(attribute='interfaces') | flatten
            | selectattr('passive', 'undefined') | list
            + (ospf.areas | default([]) + ospf_advanced.areas | default([]))
            | map(attribute='interfaces') | flatten
            | selectattr('passive', 'defined')
            | selectattr('passive', 'equalto', false) | list }}"
  loop_control:
    label: "OSPF {{ item.name }}"
  when: _validate_platform == 'ios'

- name: "Validate | NX-OS: get OSPF neighbors"
  cisco.nxos.nxos_command:
    commands:
      - "show ip ospf neighbors"
  register: _nxos_ospf_raw
  changed_when: false
  when: _validate_platform == 'nxos'

- name: "Validate | NX-OS: check OSPF neighbors are FULL"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'ospf_neighbor',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'interface': item.name,
        'expected': 'FULL',
        'actual': 'FULL' if 'FULL' in _nxos_ospf_raw.stdout[0] else 'No FULL adjacency',
        'result': 'PASS' if 'FULL' in _nxos_ospf_raw.stdout[0] else 'FAIL',
        'detail': 'NX-OS OSPF on ' + item.name
      }] }}
  loop: "{{ ospf.areas | default([]) | map(attribute='interfaces') | flatten
            | selectattr('passive', 'undefined') | list }}"
  loop_control:
    label: "OSPF {{ item.name }}"
  when: _validate_platform == 'nxos'

- name: "Validate | Junos: get OSPF neighbors"
  junipernetworks.junos.junos_command:
    commands:
      - show ospf neighbor
    display: text
  register: _junos_ospf_raw
  changed_when: false
  when: _validate_platform == 'junos'

- name: "Validate | Junos: check OSPF neighbors are Full"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'ospf_neighbor',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'interface': item.name,
        'expected': 'Full',
        'actual': 'Full' if 'Full' in _junos_ospf_raw.stdout[0] else 'No Full adjacency',
        'result': 'PASS' if 'Full' in _junos_ospf_raw.stdout[0] else 'FAIL',
        'detail': 'Junos OSPF on ' + item.name
      }] }}
  loop: "{{ ospf.areas | default([]) | map(attribute='interfaces') | flatten
            | selectattr('passive', 'undefined') | list }}"
  loop_control:
    label: "OSPF {{ item.name }}"
  when: _validate_platform == 'junos'
EOF
```

### VLAN Validation Check

```bash
cat > ~/projects/ansible-network/roles/network_validate/tasks/checks/vlans.yml << 'EOF'
---
# checks/vlans.yml — VLAN existence and active state validation

- name: "Validate | IOS: get VLAN database"
  cisco.ios.ios_command:
    commands:
      - show vlan brief
  register: _ios_vlan_raw
  changed_when: false
  when: _validate_platform == 'ios'

- name: "Validate | IOS: check each expected VLAN exists and is active"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'vlan_state',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'vlan_id': item.id,
        'vlan_name': item.name,
        'expected': 'active',
        'actual': 'active' if (item.id | string) in _ios_vlan_raw.stdout[0] and
                  'active' in (_ios_vlan_raw.stdout[0]
                  | regex_search(item.id | string + r'\s+\S+\s+active', multiline=True) | default(''))
                  else 'not found or inactive',
        'result': 'PASS' if (item.id | string) in _ios_vlan_raw.stdout[0] and
                  'active' in (_ios_vlan_raw.stdout[0]
                  | regex_search(item.id | string + r'.*active', multiline=True) | default(''))
                  else 'FAIL',
        'detail': 'VLAN ' + (item.id | string) + ' (' + item.name + ')'
      }] }}
  loop: "{{ vlans | default([]) }}"
  loop_control:
    label: "VLAN {{ item.id }}"
  when: _validate_platform == 'ios'

- name: "Validate | NX-OS: get VLAN database"
  cisco.nxos.nxos_command:
    commands:
      - show vlan brief
  register: _nxos_vlan_raw
  changed_when: false
  when: _validate_platform == 'nxos'

- name: "Validate | NX-OS: check each expected VLAN exists and is active"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'vlan_state',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'vlan_id': item.id,
        'vlan_name': item.name,
        'expected': 'active',
        'actual': 'active' if (item.id | string) in _nxos_vlan_raw.stdout[0] and
                  'active' in (_nxos_vlan_raw.stdout[0]
                  | regex_search(item.id | string + r'.*active', multiline=True) | default(''))
                  else 'not found or inactive',
        'result': 'PASS' if (item.id | string) in _nxos_vlan_raw.stdout[0] and
                  'active' in (_nxos_vlan_raw.stdout[0]
                  | regex_search(item.id | string + r'.*active', multiline=True) | default(''))
                  else 'FAIL',
        'detail': 'VLAN ' + (item.id | string) + ' (' + item.name + ')'
      }] }}
  loop: "{{ base_vlans | default([]) + local_vlans | default([]) }}"
  loop_control:
    label: "VLAN {{ item.id }}"
  when: _validate_platform == 'nxos'
EOF
```

### Routing Table Check

```bash
cat > ~/projects/ansible-network/roles/network_validate/tasks/checks/routing.yml << 'EOF'
---
# checks/routing.yml — Routing table verification

- name: "Validate | IOS: get routing table"
  cisco.ios.ios_command:
    commands:
      - show ip route
  register: _ios_route_raw
  changed_when: false
  when: _validate_platform == 'ios'

- name: "Validate | IOS: check loopback networks are in routing table"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'routing_table',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'prefix': item.value.ip + '/' + (item.value.prefix | string),
        'expected': 'present',
        'actual': 'present' if item.value.ip in _ios_route_raw.stdout[0] else 'missing',
        'result': 'PASS' if item.value.ip in _ios_route_raw.stdout[0] else 'FAIL',
        'detail': 'Loopback ' + item.key + ' (' + item.value.ip + '/' +
                  (item.value.prefix | string) + ') in routing table'
      }] }}
  loop: "{{ loopbacks | default({}) | dict2items }}"
  loop_control:
    label: "{{ item.key }} — {{ item.value.ip }}"
  when: _validate_platform == 'ios' and loopbacks is defined

- name: "Validate | IOS: check default route is present (if static routes defined)"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'routing_table',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'prefix': '0.0.0.0/0',
        'expected': 'present',
        'actual': 'present' if '0.0.0.0' in _ios_route_raw.stdout[0] else 'missing',
        'result': 'PASS' if '0.0.0.0' in _ios_route_raw.stdout[0] else 'FAIL',
        'detail': 'Default route 0.0.0.0/0 in routing table'
      }] }}
  when:
    - _validate_platform == 'ios'
    - static_routes is defined
    - static_routes | selectattr('destination', 'equalto', '0.0.0.0') | list | length > 0
EOF
```

### Report Generation

```bash
mkdir -p ~/projects/ansible-network/roles/network_validate/tasks/reporting
mkdir -p ~/projects/ansible-network/roles/network_validate/templates

cat > ~/projects/ansible-network/roles/network_validate/tasks/reporting/generate_report.yml << 'EOF'
---
# reporting/generate_report.yml — Structured pass/fail report generation

- name: "Report | Count results by status"
  ansible.builtin.set_fact:
    _validate_pass_count: "{{ _validate_results | selectattr('result', 'equalto', 'PASS') | list | length }}"
    _validate_fail_count: "{{ _validate_results | selectattr('result', 'equalto', 'FAIL') | list | length }}"
    _validate_total_count: "{{ _validate_results | length }}"

- name: "Report | Display validation summary to console"
  ansible.builtin.debug:
    msg: |
      ════════════════════════════════════════════════════════════
       VALIDATION REPORT: {{ inventory_hostname }}
       Platform: {{ _validate_platform | upper }}
       Time: {{ _validate_start_time }}
      ════════════════════════════════════════════════════════════
       Total checks:  {{ _validate_total_count }}
       PASSED:        {{ _validate_pass_count }}
       FAILED:        {{ _validate_fail_count }}
      ════════════════════════════════════════════════════════════
      {% for result in _validate_results %}
       [{{ result.result }}] {{ result.check | upper }}: {{ result.detail }}
      {% endfor %}
      ════════════════════════════════════════════════════════════
      {% if _validate_fail_count | int > 0 %}
       ⚠ VALIDATION FAILED — {{ _validate_fail_count }} check(s) did not pass
      {% else %}
       ✓ ALL CHECKS PASSED
      {% endif %}
      ════════════════════════════════════════════════════════════

- name: "Report | Write JSON report to file"
  ansible.builtin.copy:
    content: "{{ _validate_report_data | to_nice_json }}"
    dest: "{{ validate_report_dir }}/{{ inventory_hostname }}_{{ _validate_timestamp }}.json"
    mode: '0644'
  vars:
    _validate_report_data:
      device: "{{ inventory_hostname }}"
      platform: "{{ _validate_platform }}"
      timestamp: "{{ _validate_start_time }}"
      summary:
        total: "{{ _validate_total_count | int }}"
        passed: "{{ _validate_pass_count | int }}"
        failed: "{{ _validate_fail_count | int }}"
      results: "{{ _validate_results }}"
  delegate_to: localhost

- name: "Report | Write text report to file"
  ansible.builtin.template:
    src: validation_report.txt.j2
    dest: "{{ validate_report_dir }}/{{ inventory_hostname }}_{{ _validate_timestamp }}.txt"
    mode: '0644'
  delegate_to: localhost

- name: "Report | Assert all checks passed (if validate_fail_on_error)"
  ansible.builtin.assert:
    that:
      - _validate_fail_count | int == 0
    fail_msg: >
      Validation FAILED on {{ inventory_hostname }}:
      {{ _validate_fail_count }} of {{ _validate_total_count }} checks failed.
      Failed checks:
      {% for r in _validate_results | selectattr('result', 'equalto', 'FAIL') | list %}
        - {{ r.check }}: {{ r.detail }} (expected={{ r.expected }}, actual={{ r.actual }})
      {% endfor %}
      See full report: {{ validate_report_dir }}/{{ inventory_hostname }}_{{ _validate_timestamp }}.json
    success_msg: "All {{ _validate_total_count }} validation checks passed on {{ inventory_hostname }}"
  when: validate_fail_on_error | bool
EOF
```

### Report Template

```bash
cat > ~/projects/ansible-network/roles/network_validate/templates/validation_report.txt.j2 << 'EOF'
Network Validation Report
=========================
Device:    {{ inventory_hostname }}
Platform:  {{ _validate_platform | upper }}
Timestamp: {{ _validate_start_time }}
Report:    {{ validate_report_dir }}/{{ inventory_hostname }}_{{ _validate_timestamp }}.txt

Summary
-------
Total checks: {{ _validate_total_count }}
Passed:       {{ _validate_pass_count }}
Failed:       {{ _validate_fail_count }}

Results
-------
{% for result in _validate_results %}
[{{ '%-4s' | format(result.result) }}] {{ '%-20s' | format(result.check) }} {{ result.detail }}
         Expected: {{ result.expected }}  |  Actual: {{ result.actual }}
{% endfor %}

{% if _validate_fail_count | int > 0 %}
FAILED CHECKS
-------------
{% for result in _validate_results | selectattr('result', 'equalto', 'FAIL') | list %}
  - [{{ result.check }}] {{ result.detail }}
    Expected: {{ result.expected }}
    Actual:   {{ result.actual }}
{% endfor %}
{% endif %}
EOF
```

---

## 22.5 — Using the Validation Role

### Standalone Validation Playbook

```bash
cat > ~/projects/ansible-network/playbooks/validate/validate_network.yml << 'EOF'
---
# validate_network.yml — Run validation role against all devices
# Can run standalone after deployment or as part of the pipeline

- name: "Validate | IOS devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  roles:
    - role: network_validate
      vars:
        validate_interfaces: true
        validate_bgp: true
        validate_ospf: true
        validate_vlans: true
        validate_routing: true
        validate_fail_on_error: true

- name: "Validate | NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  roles:
    - role: network_validate
      vars:
        validate_interfaces: true
        validate_bgp: true
        validate_ospf: true
        validate_vlans: true
        validate_fail_on_error: true

- name: "Validate | Junos devices"
  hosts: junos_devices
  gather_facts: false
  connection: ansible.netcommon.netconf

  roles:
    - role: network_validate
      vars:
        validate_interfaces: true
        validate_bgp: true
        validate_ospf: true
        validate_fail_on_error: true

- name: "Validate | Generate consolidated report"
  hosts: localhost
  gather_facts: false

  tasks:
    - name: "Report | List all generated validation reports"
      ansible.builtin.find:
        paths: "reports/validation"
        patterns: "*.json"
      register: report_files

    - name: "Report | Display report file locations"
      ansible.builtin.debug:
        msg: "Reports generated in reports/validation/"
        verbosity: 0
EOF
```

### Integrate Into the Deployment Pipeline

```bash
cat > ~/projects/ansible-network/playbooks/deploy/pipeline_ios.yml << 'EOF'
---
# pipeline_ios.yml — Complete deploy + validate pipeline for IOS devices
# Usage: ansible-playbook playbooks/deploy/pipeline_ios.yml --limit wan-r1

- name: "Pipeline | Import CCNA configuration deployment"
  ansible.builtin.import_playbook: deploy_ios_ccna.yml

- name: "Pipeline | Import CCNP configuration deployment"
  ansible.builtin.import_playbook: deploy_ios_ccnp.yml

- name: "Pipeline | Post-deployment validation"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  roles:
    - role: network_validate
      vars:
        validate_fail_on_error: true
        validate_report_dir: "reports/validation/{{ lookup('pipe', 'date +%Y%m%d') }}"
EOF
```

### Running the Full Pipeline

```bash
# Complete workflow with all validation stages
cd ~/projects/ansible-network

# Stage 1: Pre-flight
./scripts/pre_change_check.sh playbooks/deploy/deploy_ios_ccna.yml --limit wan-r1

# Stage 3+4: Deploy then validate
ansible-playbook playbooks/deploy/pipeline_ios.yml --limit wan-r1

# Validation only (no deployment) — for health checks and scheduled runs
ansible-playbook playbooks/validate/validate_network.yml

# View a report
cat reports/validation/wan-r1_20240316_140000.txt

# Query failed checks across all devices using jq
jq '.results[] | select(.result == "FAIL")' reports/validation/*.json
```

---

## 22.6 — `failed_when` and `assert` Patterns

Quick reference for the two validation primitives used throughout the role.

### `failed_when` — Task-Level Failure Control

```yaml
# Fail when output contains an error string
- name: "Verify | Check config was accepted"
  cisco.ios.ios_command:
    commands: [show logging | include Error]
  register: log_check
  changed_when: false
  failed_when: "'%Error' in log_check.stdout[0]"

# Fail when a count is below threshold
- name: "Verify | Check BGP prefix count"
  cisco.ios.ios_command:
    commands: [show ip bgp summary]
  register: bgp_check
  changed_when: false
  failed_when: >
    bgp_check.stdout[0] | regex_findall('Established') | length < 2

# Multiple conditions (AND)
- name: "Verify | Interface is up with correct IP"
  cisco.ios.ios_facts:
    gather_subset: [interfaces]
  register: iface_facts
  failed_when:
    - iface_facts.ansible_facts.ansible_net_interfaces['GigabitEthernet1'].operstatus != 'up'
    - "'10.10.10.1' not in iface_facts.ansible_facts.ansible_net_interfaces['GigabitEthernet1'].ipv4 | map(attribute='address') | list"
```

### `assert` — Structured Pass/Fail with Messages

```yaml
# Basic assert with clear messages
- name: "Verify | Assert BGP is established"
  ansible.builtin.assert:
    that:
      - "'Established' in bgp_summary.stdout[0]"
    fail_msg: |
      BGP not established on {{ inventory_hostname }}.
      Expected: at least one neighbor in Established state
      Output: {{ bgp_summary.stdout[0][:200] }}
    success_msg: "PASS: BGP established on {{ inventory_hostname }}"

# Assert with loop — one check per expected neighbor
- name: "Verify | Assert each neighbor is established"
  ansible.builtin.assert:
    that:
      - "item.ip in bgp_summary.stdout[0]"
    fail_msg: "BGP neighbor {{ item.ip }} ({{ item.description }}) not found in BGP table"
    success_msg: "PASS: BGP neighbor {{ item.ip }} present"
  loop: "{{ bgp.neighbors | default([]) }}"
  loop_control:
    label: "{{ item.ip }}"

# Soft assert — collect all failures, report at end
# Use ignore_errors: true with a results accumulator
- name: "Verify | Check all VLANs (collect, don't stop)"
  ansible.builtin.assert:
    that:
      - "(item.id | string) in vlan_output.stdout[0]"
    fail_msg: "VLAN {{ item.id }} missing"
    success_msg: "PASS: VLAN {{ item.id }} present"
  loop: "{{ vlans | default([]) }}"
  loop_control:
    label: "VLAN {{ item.id }}"
  ignore_errors: true    # Collect all failures rather than stopping on first
  register: vlan_assert_results

- name: "Verify | Fail if any VLAN check failed"
  ansible.builtin.fail:
    msg: "{{ vlan_assert_results.results | selectattr('failed', 'equalto', true) | map(attribute='msg') | list | join('\n') }}"
  when: vlan_assert_results is failed
```

---


The full validation pipeline is in place — pre-deployment static analysis, dry-run previewing, post-deployment state verification, and a reusable role that produces structured reports across IOS, NX-OS, and Junos. Part 23 builds on this foundation to create automated compliance checking — verifying that every device meets defined standards and flagging drift.


