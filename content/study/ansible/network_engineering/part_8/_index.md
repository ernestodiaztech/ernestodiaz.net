---
draft: true
title: '8 - Structure'
weight: 8
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 8: Ansible Directory Structure & Best Practices

> *By the end of Part 7, the project directory has grown organically — files added as each part needed them. That's fine for learning, but now I step back and look at the whole structure deliberately. This part locks in the definitive project layout, expands `ansible.cfg` with every setting explained, establishes naming conventions I'll follow for the rest of the guide, and shows exactly how to separate playbooks by function so the project stays navigable as it grows.*

---

## 8.1 — The Definitive Project Structure

Here is the complete, final directory structure for the `ansible-network` project. Everything built in Parts 1–7 fits into this layout, and everything built in Parts 9–33 will slot in here too.

```
ansible-network/                        ← Project root
│
├── ansible.cfg                         ← Project-wide Ansible configuration
├── .gitignore                          ← Git exclusions (Part 4)
├── .ansible-lint                       ← ansible-lint configuration (Part 5)
├── .yamllint                           ← yamllint configuration (Part 5)
├── .pre-commit-config.yaml             ← Pre-commit hooks (Part 4)
├── README.md                           ← Project documentation
│
├── inventory/                          ← All inventory files
│   ├── hosts.yml                       ← Static inventory (Part 7)
│   ├── netbox.yml                      ← Dynamic inventory plugin (Part 26)
│   ├── group_vars/                     ← Group-level variables
│   │   ├── all.yml
│   │   ├── cisco_ios.yml
│   │   ├── cisco_nxos.yml
│   │   ├── paloalto.yml
│   │   ├── spine_switches.yml
│   │   ├── leaf_switches.yml
│   │   ├── wan.yml
│   │   └── linux_hosts.yml
│   └── host_vars/                      ← Per-device variables
│       ├── wan-r1.yml
│       ├── wan-r2.yml
│       ├── fw-01.yml
│       ├── spine-01.yml
│       ├── spine-02.yml
│       ├── leaf-01.yml
│       ├── leaf-02.yml
│       ├── host-01.yml
│       └── host-02.yml
│
├── playbooks/                          ← All playbooks, organized by function
│   ├── deploy/                         ← Configuration deployment playbooks
│   │   ├── site.yml                    ← Master playbook (runs everything)
│   │   ├── deploy_base_config.yml      ← Base config across all platforms
│   │   ├── deploy_vlans.yml            ← VLAN deployment (NX-OS)
│   │   ├── deploy_bgp.yml              ← BGP configuration
│   │   ├── deploy_ospf.yml             ← OSPF configuration
│   │   └── deploy_acls.yml             ← ACL deployment (IOS-XE)
│   ├── validate/                       ← Validation and compliance playbooks
│   │   ├── validate_connectivity.yml   ← Ping tests, neighbor checks
│   │   ├── validate_bgp.yml            ← BGP neighbor state validation
│   │   ├── validate_vlans.yml          ← VLAN existence validation
│   │   └── validate_compliance.yml     ← Full compliance check
│   ├── backup/                         ← Configuration backup playbooks
│   │   ├── backup_all.yml              ← Backup all devices
│   │   ├── backup_ios.yml              ← IOS-only backup
│   │   └── backup_nxos.yml             ← NX-OS-only backup
│   ├── rollback/                       ← Rollback and remediation playbooks
│   │   ├── rollback_config.yml         ← Restore from backup
│   │   └── rollback_bgp.yml            ← BGP-specific rollback
│   ├── report/                         ← Reporting and inventory playbooks
│   │   ├── report_facts.yml            ← Gather and report device facts
│   │   ├── report_interfaces.yml       ← Interface state report
│   │   └── report_bgp_neighbors.yml    ← BGP neighbor report
│   └── utils/                          ← Utility and maintenance playbooks
│       ├── update_packages.yml         ← Keep packages current (Part 23)
│       └── test_connectivity.yml       ← Ad-hoc connectivity test
│
├── roles/                              ← Reusable Ansible roles (Part 13)
│   ├── cisco_ios_base/                 ← IOS base configuration role
│   ├── cisco_nxos_base/                ← NX-OS base configuration role
│   ├── panos_base/                     ← PAN-OS base configuration role
│   └── common_network/                 ← Vendor-neutral common tasks
│
├── collections/                        ← Project-local Ansible collections
│   └── requirements.yml                ← Collection dependency file (Part 3)
│
├── templates/                          ← Jinja2 configuration templates (Part 11)
│   ├── ios/
│   │   ├── base_config.j2
│   │   ├── bgp.j2
│   │   └── acl.j2
│   ├── nxos/
│   │   ├── base_config.j2
│   │   └── bgp.j2
│   └── panos/
│       └── security_policy.j2
│
├── files/                              ← Static files deployed to devices
│   ├── ios/
│   │   └── banner.txt                  ← Login banner text
│   └── ssl/
│       └── ca-bundle.crt               ← CA certificates for HTTPS
│
├── vars/                               ← Shared variable files (non-inventory)
│   ├── common_vlans.yml                ← VLANs shared across the environment
│   ├── bgp_policy.yml                  ← BGP route maps and policies
│   └── acl_definitions.yml             ← Reusable ACL definitions
│
├── backups/                            ← Device configuration backups
│   ├── cisco_ios/
│   │   ├── wan-r1/
│   │   └── wan-r2/
│   ├── cisco_nxos/
│   └── paloalto/
│
├── scripts/                            ← Helper shell and Python scripts
│   ├── test-connectivity.sh            ← Quick Ansible ping test (Part 6)
│   └── setup.sh                        ← Fresh VM setup script (Part 1)
│
├── containerlab/                       ← Containerlab topology (Part 6)
│   ├── enterprise-lab.yml              ← Topology definition
│   └── configs/                        ← Device startup configs
│       ├── wan-r1-base.cfg
│       └── spine-01-base.cfg
│
└── .vscode/                            ← VS Code project settings (Part 1)
    └── settings.json
```

### What Each Top-Level Directory Is For

**`inventory/`** — The source of truth for what devices exist and how to reach them. Static YAML files now, Netbox dynamic plugin later. Nothing that isn't about device identity and connection settings belongs here.

**`playbooks/`** — Every playbook I write. Organized into subdirectories by function (deploy, validate, backup, rollback, report, utils). Never a flat pile of `.yml` files in the root.

**`roles/`** — Reusable, self-contained units of automation. A role for IOS base config can be called from any playbook. Roles are covered fully in Part 13.

**`collections/`** — Project-local Ansible collections and the `requirements.yml` file. Keeps collection versions tied to the project.

**`templates/`** — Jinja2 (`.j2`) template files. Organized by platform. Templates reference variables from inventory to generate device-specific configurations. Covered in Part 11.

**`files/`** — Static files that get deployed to devices unchanged — banner text, certificates, scripts. Unlike templates, these don't have dynamic variable substitution.

**`vars/`** — Variable files that don't belong in the inventory because they're not device-specific. Shared VLAN definitions, BGP policy parameters, ACL definitions that apply across platforms.

**`backups/`** — Generated by backup playbooks. In `.gitignore` by default — backups are large text files that change constantly and don't belong in Git. A separate backup repository or external storage is the right approach for production.

**`scripts/`** — Shell and Python helper scripts. Not Ansible playbooks — these are things like the `test-connectivity.sh` from Part 6, or a script that sets up a fresh VM.

**`containerlab/`** — Everything related to the lab environment. Topology files, startup configs. This is lab infrastructure — it stays in Git but is clearly separated from the automation content.

### ### ℹ️ Info

> The `backups/` directory is committed to `.gitignore` in this guide. In a production environment, configuration backups deserve their own dedicated storage — either a separate Git repository (versioned backups with full history), an S3 bucket, or a dedicated backup platform like Oxidized or Rancid. Keeping backups in the same repo as the automation code creates a repo that grows unboundedly and mixes infrastructure state with infrastructure code.

### ### 🏢 Real-World Scenario

> The most common structure failure I've seen in enterprise Ansible projects is the "flat playbooks directory" — 40 YAML files in a single folder with names like `new_vlan.yml`, `bgp_fix.yml`, `test2.yml`, `johns_playbook_FINAL.yml`. Nobody knows which ones are current, which ones are safe to run in production, or what half of them do. The subdirectory-by-function structure (`deploy/`, `validate/`, `backup/`, `rollback/`) makes the purpose of every playbook immediately clear from its location. When a junior engineer needs to run a backup, they look in `playbooks/backup/` — not guess from 40 filenames.

---

## 8.2 — Creating the Full Directory Structure

With the definitive layout decided, I create all directories that don't exist yet:

```bash
cd ~/projects/ansible-network

# Create all required directories
mkdir -p playbooks/{deploy,validate,backup,rollback,report,utils}
mkdir -p roles
mkdir -p templates/{ios,nxos,panos}
mkdir -p files/{ios,ssl}
mkdir -p vars
mkdir -p backups/{cisco_ios,cisco_nxos,paloalto}
mkdir -p scripts

# Verify the structure
tree . -L 3 --dirsfirst
```

### Add `backups/` to `.gitignore`

```bash
echo -e "\n# Configuration backups — stored externally\nbackups/" >> .gitignore
git add .gitignore
git commit -m "chore: exclude backups/ directory from Git"
```

### Create Placeholder Files to Track Empty Directories

Git doesn't track empty directories. I add `.gitkeep` placeholder files so the directory structure is committed:

```bash
# Add .gitkeep to all empty directories that should be tracked
find . -type d -empty -not -path "./.git/*" -not -path "./backups/*" \
    -exec touch {}/.gitkeep \;

git add .
git commit -m "chore: add directory structure with .gitkeep placeholders"
```

### ### 💡 Tip

> The `.gitkeep` convention (an empty file named `.gitkeep`) is the standard way to commit empty directories to Git. Some teams use `.keep` instead — either works. When a directory gets its first real file, the `.gitkeep` gets removed. I don't commit `.gitkeep` files into `backups/` since that whole directory is gitignored.

---

## 8.3 — The `ansible.cfg` File — Complete Reference

Part 5 created a working `ansible.cfg`. Now I expand it into the definitive version for this guide — every setting explained, lab vs. production differences called out explicitly.

```bash
nano ~/projects/ansible-network/ansible.cfg
```

```ini
# =============================================================
# ansible.cfg — Project configuration for ansible-network
# Location: project root (takes precedence over ~/.ansible.cfg)
#
# Settings marked [LAB ONLY] must be reviewed before
# promoting this configuration to a production environment.
# =============================================================


# =============================================================
[defaults]
# =============================================================

# --- Inventory ---
# Default inventory path. Using a directory allows multiple
# inventory files (static + dynamic) to be loaded simultaneously.
inventory = inventory/

# --- Remote User ---
# The SSH username Ansible uses to connect. This is the default;
# per-platform overrides live in group_vars/.
remote_user = ansible

# --- Host Key Checking ---
# [LAB ONLY] Disables SSH host key verification.
# In production: set to True and maintain a known_hosts file.
# Rationale for lab: Containerlab generates new host keys on
# every deploy — constant key conflicts without this setting.
host_key_checking = False

# --- Parallelism ---
# Number of devices Ansible operates on simultaneously.
# Default is 5. With 16GB RAM and a 9-device lab, 9 is fine.
# In production with 200+ devices, tune based on control node capacity.
forks = 9

# --- Roles Path ---
# Where Ansible looks for roles. Multiple paths separated by colon.
# Project-local roles first, then any shared/system roles.
roles_path = roles/

# --- Collections Path ---
# Where Ansible looks for collections.
# Project-local collections (./collections/) take precedence over
# user-level collections (~/.ansible/collections/).
collections_path = collections/:~/.ansible/collections

# --- Retry Files ---
# Retry files (.retry) are created when a playbook fails mid-run.
# They contain a list of failed hosts for re-running the playbook.
# Disabled here because they clutter the project directory and
# are already in .gitignore. Enable temporarily when debugging
# large playbook runs against many devices.
retry_files_enabled = False

# --- Stdout Callback ---
# Controls how playbook output is formatted.
# 'yaml' produces clean, readable output for network automation.
# Alternatives: 'default' (verbose), 'dense' (compact), 'json' (machine-parseable)
stdout_callback = yaml

# --- Other Callbacks ---
# Show task timing at the end of each playbook run.
# Useful for identifying slow tasks and optimizing playbooks.
callbacks_enabled = ansible.posix.timer, ansible.posix.profile_tasks

# --- Deprecation Warnings ---
# Set to True when troubleshooting to see all deprecation warnings.
# False in normal operation to reduce output noise.
deprecation_warnings = False

# --- System Warnings ---
# Suppress warnings about the control node Python version, etc.
system_warnings = True

# --- Python Interpreter ---
# 'auto_silent' automatically detects the best Python interpreter
# without printing a warning. Uses the virtualenv Python when
# the virtualenv is active (which it always should be for this project).
interpreter_python = auto_silent

# --- Vault ---
# Path to the vault password file.
# [SECURITY] This file must NEVER be committed to Git.
# It is listed in .gitignore. Create it manually on each machine:
#   echo "my_vault_password" > .vault_pass && chmod 600 .vault_pass
# Uncomment when Ansible Vault is configured (Part 12):
# vault_password_file = .vault_pass

# --- Fact Caching ---
# Cache gathered facts to disk to speed up subsequent playbook runs.
# Useful when gather_facts is True (Linux hosts, not network devices).
# fact_caching = jsonfile
# fact_caching_connection = /tmp/ansible_facts_cache
# fact_caching_timeout = 3600    # Cache facts for 1 hour

# --- Gathering ---
# Default: 'implicit' — gather facts for every play unless gather_facts: false.
# 'explicit' — only gather facts when explicitly set gather_facts: true.
# For network automation, 'explicit' is preferred since network plays
# always set gather_facts: false anyway. This prevents accidental
# fact gathering against network devices that can't run Python.
gathering = explicit

# --- Log Path ---
# Write Ansible output to a log file in addition to stdout.
# Useful for auditing playbook runs in production.
# [LAB ONLY] Disabled here — enable in production.
# log_path = /var/log/ansible/ansible.log

# --- Any Errors Fatal ---
# If True, stop the entire play if any host fails a task.
# Default False — Ansible continues with remaining hosts.
# Per-play override is better than a global setting here.
any_errors_fatal = False

# --- Timeout ---
# Default SSH connection timeout in seconds.
# Increase for devices on slow WAN links or slow-booting devices.
timeout = 30


# =============================================================
[inventory]
# =============================================================

# Enable specific inventory plugins.
# 'yaml' handles hosts.yml, 'auto' handles plugin detection,
# 'netbox.netbox.nb_inventory' handles Netbox dynamic inventory.
enable_plugins = yaml, ini, auto, netbox.netbox.nb_inventory

# Cache the inventory to speed up large dynamic inventory queries.
# Particularly useful with Netbox when querying 500+ devices.
# cache = True
# cache_plugin = jsonfile
# cache_timeout = 3600
# cache_connection = /tmp/ansible_inventory_cache


# =============================================================
[ssh_connection]
# =============================================================

# SSH arguments passed to the SSH client.
# -o ControlMaster=no : Disable SSH multiplexing/ControlMaster.
#   Network devices (IOS, NX-OS) don't support SSH multiplexing.
#   Leaving it enabled causes connection errors on network devices.
# -o ControlPersist=no : Related to ControlMaster — disable both.
# -o StrictHostKeyChecking=no : Mirrors host_key_checking=False above.
#   [LAB ONLY] — remove in production.
ssh_args = -o ControlMaster=no -o ControlPersist=no -o StrictHostKeyChecking=no

# Pipelining speeds up module execution on Linux/Unix hosts by
# reducing SSH round-trips. Has no effect on network devices.
# Must be False if sudoers has 'requiretty' set.
pipelining = False

# Number of times to retry an SSH connection before giving up.
# Useful for devices that occasionally reject connections during
# high CPU periods (common on older IOS devices).
retries = 3


# =============================================================
[persistent_connection]
# =============================================================
# These settings control the persistent SSH connections used by
# network_cli and netconf connection plugins. Unlike regular SSH
# connections (which open/close per task), persistent connections
# stay open for the duration of a play — critical for network
# device automation where re-authentication is slow.

# How long (seconds) to wait when establishing a new persistent connection.
# Increase for slow devices or high-latency WAN connections.
connect_timeout = 30

# How long (seconds) to wait for a CLI command to return output.
# Increase for commands that take a long time (e.g., 'show tech-support',
# NX-OS 'copy run start' on a loaded switch).
command_timeout = 60

# How long (seconds) an idle persistent connection is kept alive.
# After this period of inactivity, the connection is closed.
# A new connection is opened automatically if needed.
connect_retry_timeout = 15


# =============================================================
[colors]
# =============================================================
# Customize output colors. These are the defaults — uncomment
# and change if the default colors are hard to read in your terminal.
# highlight = white
# verbose = blue
# warn = bright purple
# error = red
# debug = dark gray
# deprecate = purple
# skip = cyan
# unreachable = red
# ok = green
# changed = yellow
# diff_add = green
# diff_remove = red
# diff_lines = cyan
```

### Lab vs. Production Configuration Differences

The three settings that **must** change before this config goes anywhere near production:

```ini
# LAB                              PRODUCTION
host_key_checking = False    →    host_key_checking = True
ssh_args = ... StrictHostKeyChecking=no  →  remove that argument
vault_password_file = .vault_pass   →   ensure this is set and .vault_pass is secured
```

And the settings that should be **added** for production:

```ini
# Add for production
log_path = /var/log/ansible/ansible.log
any_errors_fatal = False          # Review per-playbook — some should be True
fact_caching = jsonfile           # Speed up repeated runs
```

### ### 🔴 Danger

> The combination of `host_key_checking = False` and `StrictHostKeyChecking=no` in SSH args means Ansible will happily connect to any device claiming to be the target IP — including a rogue device that has taken over that IP address, or a man-in-the-middle attacker. In a production environment where Ansible is running privileged configuration commands against network devices, this is a critical security gap. Always enable host key checking in production and maintain a pre-populated `known_hosts` file.

### Checking Which `ansible.cfg` Is Active

```bash
ansible --version | grep "config file"
# config file = /home/ansible/projects/ansible-network/ansible.cfg

# Confirm specific settings are being read
ansible-config dump --only-changed
```

---

## 8.4 — Naming Conventions

Consistent naming is what separates a project that's navigable after six months from one that requires an archaeology expedition. I follow these conventions throughout this guide.

### Playbook Files

```
Format:  <verb>_<noun>[_<qualifier>].yml
         <verb>: deploy, validate, backup, rollback, report, test, update
         <noun>: what is being acted on
         <qualifier>: optional scope or platform

# ✅ Good names
deploy_bgp.yml
deploy_vlans_nxos.yml
validate_ospf_neighbors.yml
backup_all_configs.yml
rollback_bgp_config.yml
report_interface_status.yml

# ❌ Bad names
bgp.yml              ← No verb — what does it do?
new_playbook.yml     ← Meaningless
fix.yml              ← Fix what?
test2.yml            ← Why does test2 exist? What happened to test1?
johns_stuff.yml      ← Never use personal names in shared projects
FINAL_bgp.yml        ← "FINAL" is a red flag that something went wrong
```

### Role Names

```
Format:  <vendor>_<platform>_<function>   (for platform-specific roles)
         <function>                        (for vendor-neutral roles)

# ✅ Good names
cisco_ios_base
cisco_ios_bgp
cisco_nxos_vpc
panos_security_policy
common_ntp           ← Vendor-neutral
common_syslog        ← Vendor-neutral

# ❌ Bad names
my_role
new_cisco_role
bgp_role_v2_WORKING
```

### Task Names (within playbooks and roles)

Every task must have a `name:` field. Task names appear in playbook output and in AWX job logs — they're how I understand what's happening during a run.

```yaml
# ✅ Good task names — descriptive, explains what AND why
- name: "Deploy | Configure OSPF process and area on WAN interfaces"
- name: "Validate | Confirm BGP neighbors are in Established state"
- name: "Backup | Save running config to {{ backup_dir }}/{{ inventory_hostname }}"
- name: "Rollback | Restore previous configuration from backup file"

# ❌ Bad task names
- name: task1
- name: configure
- name: do the thing
- name: ios_config
```

### Variable Names

```
Format:  <scope>_<object>_<attribute>
         lowercase with underscores (snake_case) always
         never camelCase, never hyphens

# ✅ Good variable names
bgp_as
bgp_router_id
bgp_neighbors
ospf_process_id
ospf_area
interface_description
vlan_id
vlan_name
ntp_servers
device_role
backup_dir

# ❌ Bad variable names
BGP_AS           ← Uppercase — valid but unconventional
bgpAS            ← camelCase — will cause confusion
bgp-as           ← Hyphens are not valid in Python variable names
x                ← Meaningless
temp_var         ← Implies it's temporary — if it's in production code, it isn't
```

### File and Directory Names

```
# Always use:
lowercase-with-hyphens    ← For directories and most files
lowercase_with_underscores ← For Python files, variable files

# Examples
playbooks/deploy/         ← directory
deploy_bgp.yml            ← playbook file
cisco_ios_base/           ← role directory
bgp_policy.yml            ← vars file

# Never use:
CamelCase/
UPPERCASE_DIRS/
spaces in names/
```

---

## 8.5 — Separating Playbooks by Function

The `playbooks/` directory is organized into subdirectories by function. Here I build out the skeleton of the playbook structure with real, runnable examples for each function type.

### The Master Playbook: `playbooks/deploy/site.yml`

The master playbook calls other playbooks in sequence. Running `site.yml` deploys the entire environment end-to-end:

```bash
nano ~/projects/ansible-network/playbooks/deploy/site.yml
```

```yaml
---
# =============================================================
# site.yml — Master deployment playbook
# Runs all deployment playbooks in the correct order.
#
# Usage:
#   Full deployment:    ansible-playbook playbooks/deploy/site.yml
#   Specific platform:  ansible-playbook playbooks/deploy/site.yml --tags ios
#   Dry run:            ansible-playbook playbooks/deploy/site.yml --check
# =============================================================

- name: "Deploy | Base configuration on all network devices"
  import_playbook: deploy_base_config.yml
  tags:
    - base
    - always

- name: "Deploy | VLAN configuration on NX-OS switches"
  import_playbook: deploy_vlans.yml
  tags:
    - vlans
    - nxos

- name: "Deploy | OSPF routing on IOS-XE routers"
  import_playbook: deploy_ospf.yml
  tags:
    - ospf
    - routing
    - ios

- name: "Deploy | BGP configuration on all routing devices"
  import_playbook: deploy_bgp.yml
  tags:
    - bgp
    - routing

- name: "Deploy | ACL configuration on IOS-XE"
  import_playbook: deploy_acls.yml
  tags:
    - acls
    - security
    - ios
```

### A Deploy Playbook: `playbooks/deploy/deploy_base_config.yml`

```bash
nano ~/projects/ansible-network/playbooks/deploy/deploy_base_config.yml
```

```yaml
---
# =============================================================
# deploy_base_config.yml — Base configuration for all devices
# Configures: hostname, domain, NTP, DNS, syslog, SNMP, banners
#
# Usage:
#   All devices:    ansible-playbook playbooks/deploy/deploy_base_config.yml
#   IOS only:       ansible-playbook playbooks/deploy/deploy_base_config.yml \
#                       --limit cisco_ios
#   Dry run:        ansible-playbook playbooks/deploy/deploy_base_config.yml \
#                       --check --diff
# =============================================================

- name: "Deploy | Base configuration on Cisco IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Deploy | Set hostname"
      cisco.ios.ios_hostname:
        config:
          hostname: "{{ device_hostname }}"
        state: merged

    - name: "Deploy | Configure NTP servers"
      cisco.ios.ios_ntp_global:
        config:
          servers:
            - server: "{{ item }}"
              prefer: "{{ true if loop.index == 1 else false }}"
        state: merged
      loop: "{{ ntp_servers }}"

    - name: "Deploy | Configure DNS servers"
      cisco.ios.ios_config:
        lines:
          - "ip name-server {{ dns_servers | join(' ') }}"

    - name: "Deploy | Configure syslog"
      cisco.ios.ios_logging_global:
        config:
          hosts:
            - hostname: "{{ syslog_server }}"
              severity: informational
        state: merged

    - name: "Deploy | Save configuration"
      cisco.ios.ios_command:
        commands:
          - write memory

- name: "Deploy | Base configuration on Cisco NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Deploy | Set hostname"
      cisco.nxos.nxos_hostname:
        config:
          hostname: "{{ device_hostname }}"
        state: merged

    - name: "Deploy | Configure NTP servers"
      cisco.nxos.nxos_ntp_global:
        config:
          servers:
            - server: "{{ item }}"
        state: merged
      loop: "{{ ntp_servers }}"

    - name: "Deploy | Configure syslog"
      cisco.nxos.nxos_logging_global:
        config:
          hosts:
            - name: "{{ syslog_server }}"
              severity: info
        state: merged

    - name: "Deploy | Save configuration"
      cisco.nxos.nxos_command:
        commands:
          - copy running-config startup-config
```

### A Validate Playbook: `playbooks/validate/validate_connectivity.yml`

```bash
nano ~/projects/ansible-network/playbooks/validate/validate_connectivity.yml
```

```yaml
---
# =============================================================
# validate_connectivity.yml — Verify basic device connectivity
# Tests: SSH reachability, device response, basic interface state
#
# Usage:
#   ansible-playbook playbooks/validate/validate_connectivity.yml
#   ansible-playbook playbooks/validate/validate_connectivity.yml \
#       --limit cisco_ios
#
# This playbook uses assert tasks — it FAILS if checks don't pass.
# Run before and after any change window to confirm state.
# =============================================================

- name: "Validate | Connectivity to all network devices"
  hosts: network_devices
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Validate | Confirm device responds to show version"
      cisco.ios.ios_command:
        commands:
          - show version
      register: show_version_output
      when: ansible_network_os == 'cisco.ios.ios'

    - name: "Validate | Assert show version contains expected platform"
      ansible.builtin.assert:
        that:
          - "'Cisco IOS' in show_version_output.stdout[0] or
             'Cisco IOS-XE' in show_version_output.stdout[0]"
        fail_msg: >
          FAIL: {{ inventory_hostname }} did not return expected IOS output.
          Got: {{ show_version_output.stdout[0][:100] }}
        success_msg: "PASS: {{ inventory_hostname }} is running Cisco IOS/IOS-XE"
      when: ansible_network_os == 'cisco.ios.ios'

    - name: "Validate | Confirm NX-OS device responds"
      cisco.nxos.nxos_command:
        commands:
          - show version
      register: nxos_version_output
      when: ansible_network_os == 'cisco.nxos.nxos'

    - name: "Validate | Assert NX-OS version output"
      ansible.builtin.assert:
        that:
          - "'Nexus' in nxos_version_output.stdout[0]"
        fail_msg: "FAIL: {{ inventory_hostname }} did not return expected NX-OS output"
        success_msg: "PASS: {{ inventory_hostname }} is running NX-OS"
      when: ansible_network_os == 'cisco.nxos.nxos'

  post_tasks:
    - name: "Validate | Print connectivity summary"
      ansible.builtin.debug:
        msg: "All connectivity checks passed for {{ inventory_hostname }}"
```

### A Backup Playbook: `playbooks/backup/backup_all.yml`

```bash
nano ~/projects/ansible-network/playbooks/backup/backup_all.yml
```

```yaml
---
# =============================================================
# backup_all.yml — Back up running configurations from all devices
#
# Usage:
#   ansible-playbook playbooks/backup/backup_all.yml
#
# Output: backups/<platform>/<hostname>/<hostname>_<timestamp>.cfg
# Note: backups/ is in .gitignore — use external storage for production.
# =============================================================

- name: "Backup | Running configurations from IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "Backup | Create backup directory for {{ inventory_hostname }}"
      ansible.builtin.file:
        path: "backups/cisco_ios/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Backup | Gather running configuration from {{ inventory_hostname }}"
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: ios_running_config

    - name: "Backup | Write configuration to file"
      ansible.builtin.copy:
        content: "{{ ios_running_config.stdout[0] }}"
        dest: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      delegate_to: localhost

    - name: "Backup | Confirm backup was written"
      ansible.builtin.debug:
        msg: "Backup saved: backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"

- name: "Backup | Running configurations from NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "Backup | Create backup directory for {{ inventory_hostname }}"
      ansible.builtin.file:
        path: "backups/cisco_nxos/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Backup | Gather running configuration from {{ inventory_hostname }}"
      cisco.nxos.nxos_command:
        commands:
          - show running-config
      register: nxos_running_config

    - name: "Backup | Write configuration to file"
      ansible.builtin.copy:
        content: "{{ nxos_running_config.stdout[0] }}"
        dest: "backups/cisco_nxos/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      delegate_to: localhost
```

### A Rollback Playbook: `playbooks/rollback/rollback_config.yml`

```bash
nano ~/projects/ansible-network/playbooks/rollback/rollback_config.yml
```

```yaml
---
# =============================================================
# rollback_config.yml — Restore a device configuration from backup
#
# Usage:
#   ansible-playbook playbooks/rollback/rollback_config.yml \
#       --limit wan-r1 \
#       -e "backup_file=backups/cisco_ios/wan-r1/wan-r1_20240101_120000.cfg"
#
# IMPORTANT: This playbook replaces the running configuration
# entirely. It should only be run during a change window with
# network operations approval. Test in lab first.
# =============================================================

- name: "Rollback | Restore IOS-XE configuration from backup"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli

  pre_tasks:
    - name: "Rollback | Verify backup file exists before proceeding"
      ansible.builtin.stat:
        path: "{{ backup_file }}"
      register: backup_stat
      delegate_to: localhost

    - name: "Rollback | Abort if backup file not found"
      ansible.builtin.fail:
        msg: >
          Backup file not found: {{ backup_file }}
          Rollback aborted. Provide a valid backup_file path with -e.
      when: not backup_stat.stat.exists

    - name: "Rollback | Take pre-rollback snapshot for comparison"
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: pre_rollback_config

    - name: "Rollback | Save pre-rollback config as safety backup"
      ansible.builtin.copy:
        content: "{{ pre_rollback_config.stdout[0] }}"
        dest: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_pre_rollback_{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}.cfg"
        mode: '0644'
      delegate_to: localhost

  tasks:
    - name: "Rollback | Push backup configuration to {{ inventory_hostname }}"
      cisco.ios.ios_config:
        src: "{{ backup_file }}"
        replace: config    # Replace entire config, not merge
      register: rollback_result

    - name: "Rollback | Save configuration after rollback"
      cisco.ios.ios_command:
        commands:
          - write memory

  post_tasks:
    - name: "Rollback | Confirm device is responsive after rollback"
      cisco.ios.ios_command:
        commands:
          - show version
      register: post_rollback_check

    - name: "Rollback | Report rollback result"
      ansible.builtin.debug:
        msg: >
          Rollback complete for {{ inventory_hostname }}.
          Changed: {{ rollback_result.changed }}
          Device is responsive: {{ post_rollback_check is succeeded }}
```

### A Report Playbook: `playbooks/report/report_facts.yml`

```bash
nano ~/projects/ansible-network/playbooks/report/report_facts.yml
```

```yaml
---
# =============================================================
# report_facts.yml — Gather and display facts from all devices
# Outputs: device hostname, platform, version, uptime
#
# Usage:
#   ansible-playbook playbooks/report/report_facts.yml
#   ansible-playbook playbooks/report/report_facts.yml \
#       --limit cisco_ios
# =============================================================

- name: "Report | Gather facts from IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Report | Gather IOS device facts"
      cisco.ios.ios_facts:
        gather_subset:
          - hardware
          - default
      register: ios_facts

    - name: "Report | Display IOS device summary"
      ansible.builtin.debug:
        msg:
          - "Hostname:  {{ ansible_net_hostname }}"
          - "Platform:  {{ ansible_net_model | default('unknown') }}"
          - "Version:   {{ ansible_net_version }}"
          - "Uptime:    {{ ansible_net_stacked_models | default('N/A') }}"
          - "Interfaces:{{ ansible_net_interfaces | length }} total"

- name: "Report | Gather facts from NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Report | Gather NX-OS device facts"
      cisco.nxos.nxos_facts:
        gather_subset:
          - hardware
          - default

    - name: "Report | Display NX-OS device summary"
      ansible.builtin.debug:
        msg:
          - "Hostname:  {{ ansible_net_hostname }}"
          - "Platform:  {{ ansible_net_platform }}"
          - "Version:   {{ ansible_net_version }}"
          - "Interfaces:{{ ansible_net_interfaces | length }} total"
```

### ### 💡 Tip

> The `pre_tasks:` and `post_tasks:` sections in the rollback playbook are a pattern worth understanding. `pre_tasks` run before roles and regular tasks — I use them for safety checks and pre-change backups. `post_tasks` run after all regular tasks — I use them for verification and reporting. The order is always: `pre_tasks` → `roles` → `tasks` → `post_tasks`. Using `pre_tasks` to abort a dangerous operation (like rollback) before it starts is much safer than discovering a missing backup file halfway through.

---

## 8.6 — When to Use `host_vars` vs `group_vars` vs Play-Level Vars

This comes up constantly. Here's the definitive decision guide:

### The Decision Tree

```
Is this variable the same for every device in the entire inventory?
    YES → group_vars/all.yml
    NO  ↓

Is this variable the same for every device in a specific platform group?
    YES → group_vars/<platform>.yml  (e.g., group_vars/cisco_ios.yml)
    NO  ↓

Is this variable the same for a specific device role or logical group?
    YES → group_vars/<group>.yml  (e.g., group_vars/spine_switches.yml)
    NO  ↓

Is this variable unique to a specific device?
    YES → host_vars/<hostname>.yml  (e.g., host_vars/wan-r1.yml)
    NO  ↓

Is this variable only relevant to a specific play (not reusable)?
    YES → vars: block inside the playbook
    NO  ↓

Is this variable shared across multiple playbooks but not device-specific?
    YES → vars/ directory file (e.g., vars/bgp_policy.yml)
```

### Concrete Examples

```yaml
# group_vars/all.yml — Same for every device
ntp_servers:
  - 216.239.35.0
  - 216.239.35.4
ansible_user: ansible

# group_vars/cisco_ios.yml — Same for every IOS device
ansible_network_os: cisco.ios.ios
ansible_become: true

# group_vars/spine_switches.yml — Same for all spines
device_role: spine
bgp_as: 65000
fabric_mtu: 9216

# host_vars/spine-01.yml — Unique to spine-01
ansible_host: 172.16.0.21
loopback0:
  ip: 10.255.1.1
bgp_router_id: 10.255.1.1

# vars: inside a playbook — relevant only to this play
- name: Deploy BGP
  hosts: cisco_ios
  vars:
    bgp_timer_keepalive: 30     # Only used in this specific playbook
    bgp_timer_holdtime: 90
  tasks: ...

# vars/bgp_policy.yml — Shared across playbooks, not device-specific
route_map_permit_local:
  - prefix: 10.0.0.0/8
  - prefix: 172.16.0.0/12
```

### ### ℹ️ Info

> The `vars/` directory (at the project root level, not inside a role) is for variables that are shared across multiple playbooks but are not device-specific — BGP route policies, VLAN standards, ACL definitions that apply environment-wide. These are loaded explicitly in a playbook with `vars_files:`. They're distinct from `group_vars/` (automatically loaded by inventory) and `host_vars/` (also automatically loaded). If I have to explicitly `vars_files: vars/bgp_policy.yml` in a playbook to use it, I'm using the `vars/` directory correctly.

---

## 8.7 — Keeping Sensitive Data Out of the Directory Root

The directory root is the most visible part of the project — it's what people see first when they clone the repo or browse GitHub. Sensitive data must never land here.

### What Lives in the Root vs. What Doesn't

```
Project root — safe for:          Project root — never put here:
✅ ansible.cfg                    ❌ passwords or tokens
✅ .gitignore                     ❌ private keys
✅ README.md                      ❌ .vault_pass (put in .gitignore)
✅ .ansible-lint                  ❌ .env files with secrets
✅ .yamllint                      ❌ unencrypted credential files
✅ requirements files             ❌ production IP addressing details
```

### The `.vault_pass` File

When Ansible Vault is configured (Part 12), I'll use a vault password file. This file contains a single line — the vault password. It never goes in Git:

```bash
# Create the vault password file
echo "my_strong_vault_password_here" > .vault_pass
chmod 600 .vault_pass    # Owner read/write only

# Confirm it's in .gitignore
grep "vault_pass" .gitignore
# .vault_pass ← should appear
```

### The `.env` Pattern for Tokens

API tokens (for Netbox, AWX, cloud providers) are stored in a `.env` file that is sourced before running Ansible:

```bash
# .env — never committed to Git
export NETBOX_TOKEN="my_netbox_token_here"
export AWX_TOKEN="my_awx_token_here"
export PANOS_API_KEY="my_panos_api_key_here"
```

```bash
# Source before running playbooks
source .env
ansible-playbook playbooks/deploy/site.yml
```

The `inventory/netbox.yml` file references these as environment variables:

```yaml
token: "{{ lookup('env', 'NETBOX_TOKEN') }}"
```

### ### 🔴 Danger

> GitHub's secret scanning feature automatically scans public repositories for API tokens, AWS credentials, and common secret patterns. It also scans the full commit history — not just the current state of the files. A token committed and then deleted is still in the history and will be flagged. If any repository that previously held secrets is ever made public (accidentally or intentionally), those secrets are compromised regardless of whether the files were subsequently deleted. Treat every secret committed to any Git repository as permanently compromised and rotate it immediately.

---

## 8.8 — The `README.md`

Every professional project has a `README.md`. Mine serves as the entry point for any engineer who clones the repository.

```bash
nano ~/projects/ansible-network/README.md
```

```markdown
# ansible-network

Network automation playbooks and roles for the enterprise lab environment.
Covers Cisco IOS-XE, Cisco NX-OS, and Palo Alto PAN-OS automation using Ansible.

## Lab Topology

```
Internet → WAN-R1/WAN-R2 → FW-01 → SPINE-01/SPINE-02 → LEAF-01/LEAF-02 → HOST-01/HOST-02
```

| Device | Role | Platform | Mgmt IP |
|---|---|---|---|
| wan-r1 | WAN Router 1 | Cisco IOS-XE | 172.16.0.11 |
| wan-r2 | WAN Router 2 | Cisco IOS-XE | 172.16.0.12 |
| fw-01 | Firewall | Palo Alto PAN-OS | 172.16.0.10 |
| spine-01 | DC Spine 1 | Cisco NX-OS | 172.16.0.21 |
| spine-02 | DC Spine 2 | Cisco NX-OS | 172.16.0.22 |
| leaf-01 | DC Leaf 1 | Cisco NX-OS | 172.16.0.23 |
| leaf-02 | DC Leaf 2 | Cisco NX-OS | 172.16.0.24 |
| host-01 | Linux Host 1 | Alpine Linux | 172.16.0.31 |
| host-02 | Linux Host 2 | Alpine Linux | 172.16.0.32 |

## Prerequisites

- Ubuntu 22.04 VM with 16GB+ RAM (on Proxmox)
- Python 3.10+
- Docker Engine (for Containerlab)
- Containerlab 0.56+
- VS Code with Remote-SSH extension

## Setup

### 1. Clone the repository

```bash
git clone git@github.com:myusername/ansible-network.git
cd ansible-network
```

### 2. Create and activate the virtual environment

```bash
python3 -m venv ~/venvs/ansible-network
source ~/venvs/ansible-network/bin/activate
```

### 3. Install Python dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

### 4. Install Ansible collections

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### 5. Create the vault password file

```bash
echo "your_vault_password" > .vault_pass
chmod 600 .vault_pass
```

### 6. Deploy the Containerlab topology

```bash
cd containerlab
sudo containerlab deploy -t enterprise-lab.yml
cd ..
```

### 7. Verify connectivity

```bash
bash scripts/test-connectivity.sh
```

## Usage

### Run full deployment

```bash
ansible-playbook playbooks/deploy/site.yml
```

### Backup all devices

```bash
ansible-playbook playbooks/backup/backup_all.yml
```

### Validate connectivity

```bash
ansible-playbook playbooks/validate/validate_connectivity.yml
```

### Target a specific platform

```bash
ansible-playbook playbooks/deploy/deploy_base_config.yml --limit cisco_ios
```

### Dry run (no changes applied)

```bash
ansible-playbook playbooks/deploy/site.yml --check --diff
```

## Project Structure

```
ansible-network/
├── ansible.cfg              # Ansible configuration
├── inventory/               # Static inventory and variables
├── playbooks/               # Organized by function
│   ├── deploy/              # Configuration deployment
│   ├── validate/            # Compliance and validation
│   ├── backup/              # Configuration backups
│   ├── rollback/            # Rollback and restore
│   └── report/              # Reporting and facts
├── roles/                   # Reusable automation roles
├── templates/               # Jinja2 configuration templates
├── vars/                    # Shared variable files
└── containerlab/            # Lab topology definition
```

## Branch Strategy

- `main` — production-ready code only, protected branch
- `feature/*` — new playbooks and capabilities
- `fix/*` — bug fixes
- `hotfix/*` — urgent production fixes

All changes enter `main` via Pull Request with at least one reviewer approval.

## Related Documentation

- [Ansible Documentation](https://docs.ansible.com)
- [Containerlab Documentation](https://containerlab.dev)
- [Cisco IOS Collection](https://github.com/ansible-collections/cisco.ios)
- [Cisco NX-OS Collection](https://github.com/ansible-collections/cisco.nxos)
- [Palo Alto Collection](https://github.com/PaloAltoNetworks/pan-os-ansible)
```

---

## 8.9 — Common Gotchas in This Section

### ### 🪲 Gotcha — `import_playbook` vs `include_playbook` in `site.yml`

```yaml
# import_playbook — static, processed at parse time
# Pros: faster, --list-tasks works correctly, tags work fully
# Cons: cannot be conditional
import_playbook: deploy_bgp.yml

# include_playbook — dynamic, processed at runtime
# Pros: can use when: conditions, can loop
# Cons: slower, --list-tasks doesn't show included content
include_playbook: deploy_bgp.yml
```

For `site.yml` as a master playbook, I always use `import_playbook`. It's processed at parse time, meaning `--check`, `--list-tasks`, and `--tags` all work correctly across the entire play. `include_playbook` is only needed when I need dynamic conditional inclusion — rare for a master playbook.

### ### 🪲 Gotcha — `delegate_to: localhost` in backup playbooks

In the backup playbook, the `ansible.builtin.file` and `ansible.builtin.copy` tasks run on `localhost` (the Ubuntu VM), not on the network device. This is because I'm creating directories and writing files on the control node, not on the device. Without `delegate_to: localhost`, Ansible would try to run these tasks on the network device — which can't run Python and would fail.

This pattern — connecting to a device to gather data, then delegating the file-writing task to localhost — is fundamental to network automation. Always ask: "Where does this task actually need to run?" If it's writing files, creating directories, or calling an API, the answer is usually localhost.

### ### 🪲 Gotcha — `vars/` directory files not auto-loaded

Unlike `group_vars/` and `host_vars/`, files in the `vars/` directory are **not** automatically loaded. I must explicitly include them in a playbook:

```yaml
- name: Deploy BGP policy
  hosts: cisco_ios
  gather_facts: false
  vars_files:
    - ../../vars/bgp_policy.yml    # Relative path from playbook location
  tasks:
    ...
```

This is by design — `vars/` files contain policy-level data that not every playbook needs. Explicit inclusion keeps playbook dependencies clear.

### ### 🪲 Gotcha — Ansible interprets `playbooks/` subdirectory paths relative to the playbook file

When I use `import_playbook: deploy_bgp.yml` inside `playbooks/deploy/site.yml`, Ansible looks for `deploy_bgp.yml` relative to `site.yml` — in the same `playbooks/deploy/` directory. This is correct and expected. But if I try to import a playbook from a different subdirectory:

```yaml
# In playbooks/deploy/site.yml
import_playbook: ../validate/validate_connectivity.yml    # ← This works
import_playbook: validate/validate_connectivity.yml       # ← This FAILS
```

Relative paths in `import_playbook` are always relative to the calling playbook's location.

---

## 8.10 — Committing the Final Structure to Git

```bash
cd ~/projects/ansible-network

git add .
git status    # Review all new files

git commit -m "chore(structure): establish definitive project directory layout

- Add full playbooks/ subdirectory structure: deploy/, validate/,
  backup/, rollback/, report/, utils/
- Add site.yml master playbook with import_playbook structure and tags
- Add deploy_base_config.yml for IOS-XE and NX-OS base configuration
- Add validate_connectivity.yml with assert-based checks
- Add backup_all.yml with timestamped backup files and delegate_to: localhost
- Add rollback_config.yml with pre-rollback safety backup and post-check
- Add report_facts.yml for device inventory reporting
- Expand ansible.cfg with full annotation of every setting
- Add README.md with setup instructions and topology table
- Add templates/, files/, vars/ directory scaffolding with .gitkeep
- Add backups/ to .gitignore (managed externally)"
```

---

The project structure is now locked in. Every playbook written from here on has a clear home, every variable has a clear owner, and every engineer who clones this repository can understand it in minutes from the README. Part 9 starts using this structure for real work — ad-hoc commands that let me interact with the lab devices directly from the command line.


