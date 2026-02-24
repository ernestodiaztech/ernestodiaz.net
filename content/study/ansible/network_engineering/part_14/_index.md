---
draft: true
title: '14 - Tags'
weight: 14
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 14: Ansible Tags

> *A site.yml that deploys base config, VLANs, routing, and ACLs to 20 devices takes ten minutes to run in full. But most of the time I only need to touch one section — push a new VLAN, update NTP, re-run validation. Tags are how I run exactly the slice of automation I need without running everything. They're also how I make playbooks self-documenting: a task tagged `ntp,base,ios` tells me exactly what it does, what phase it belongs to, and which platform it applies to.*

---

## 14.1 — What Tags Are and Why They Matter

A tag is a label attached to a task, block, or play. When I run a playbook with `--tags`, only the tagged items run. When I run with `--skip-tags`, those items are skipped. Everything else behaves normally.

Without tags, a large playbook is all-or-nothing. With tags, it becomes a menu I can order from selectively.

### The Practical Value

```bash
# Without tags — push everything to every device (10 minutes)
ansible-playbook playbooks/deploy/site.yml

# With tags — push only VLAN changes to NX-OS devices (45 seconds)
ansible-playbook playbooks/deploy/site.yml --tags vlans --limit cisco_nxos

# With tags — run only validation, no changes
ansible-playbook playbooks/deploy/site.yml --tags validate

# With tags — skip backup (I already have a fresh backup)
ansible-playbook playbooks/deploy/site.yml --skip-tags backup
```

Tags don't change what the playbook does — they filter what runs. The playbook file stays the same; the tag arguments at runtime determine the scope.

### ### 🏢 Real-World Scenario

> During a maintenance window with a hard 2-hour limit, a network team needed to push an OSPF configuration change to 40 routers. Their site.yml took 25 minutes to run fully — far too long given the change window and the time needed for validation and rollback preparation. With `--tags ospf --tags validate`, they ran only the OSPF tasks and their validation checks: 4 minutes. The same playbook, a fraction of the runtime, none of the risk of running unrelated tasks during a change window.

---

## 14.2 — Adding Tags to Tasks, Blocks, and Plays

### Tags on Individual Tasks

The most common placement. Tags go as a list under the `tags:` key:

```yaml
tasks:
  - name: "Config | Configure NTP servers"
    cisco.ios.ios_ntp_global:
      config:
        servers:
          - server: "{{ item }}"
      state: merged
    loop: "{{ ntp_servers }}"
    tags:
      - ntp
      - base
      - ios

  - name: "Config | Configure syslog"
    cisco.ios.ios_logging_global:
      config:
        hosts:
          - hostname: "{{ syslog_server }}"
      state: merged
    tags:
      - logging
      - base
      - ios

  - name: "Validate | Check BGP neighbor state"
    cisco.ios.ios_command:
      commands:
        - show ip bgp summary
    register: bgp_summary
    tags:
      - validate
      - bgp
      - ios
```

A task can carry multiple tags. `--tags ntp` runs only the NTP task. `--tags base` runs both NTP and syslog tasks (both carry the `base` tag). `--tags ios` runs all three (all carry `ios`).

### Tags on Blocks

A `block:` groups related tasks. A tag on the block applies to every task inside it — I don't need to tag each task individually:

```yaml
tasks:
  - block:
      - name: "BGP | Configure BGP process"
        cisco.ios.ios_bgp_global:
          config:
            as_number: "{{ bgp.as_number }}"
            router_id: "{{ bgp.router_id }}"
          state: merged

      - name: "BGP | Configure BGP neighbors"
        cisco.ios.ios_bgp_global:
          config:
            as_number: "{{ bgp.as_number }}"
            neighbor:
              - neighbor_address: "{{ item.ip }}"
                remote_as: "{{ item.remote_as }}"
                description: "{{ item.description }}"
          state: merged
        loop: "{{ bgp.neighbors }}"

      - name: "BGP | Configure BGP address families"
        cisco.ios.ios_bgp_address_family:
          config:
            as_number: "{{ bgp.as_number }}"
            address_family:
              - afi: ipv4
                neighbors:
                  - neighbor_address: "{{ item.ip }}"
                    activate: true
                  for item in bgp.neighbors
          state: merged

    tags:
      - bgp
      - routing
      - ios
    # All three tasks above inherit the bgp, routing, and ios tags
    # without needing tags: on each individual task
```

Block-level tagging is the right pattern when a group of tasks are always run together or skipped together. If I need to run the BGP block independently of other routing tasks, I'd split it into its own block.

### Tags on Entire Plays

A tag on a play applies to every task in that play:

```yaml
---
- name: "Deploy | IOS-XE configuration"
  hosts: cisco_ios
  gather_facts: false
  tags:
    - ios          # Every task in this play gets the ios tag
    - deploy

  tasks:
    - name: "Configure hostname"
      cisco.ios.ios_hostname:
        ...        # Inherits: ios, deploy

    - name: "Configure NTP"
      cisco.ios.ios_ntp_global:
        ...
      tags:
        - ntp      # Has tags: ios, deploy, ntp (play tags + task tags combined)

- name: "Deploy | NX-OS configuration"
  hosts: cisco_nxos
  gather_facts: false
  tags:
    - nxos         # Every task in this play gets the nxos tag
    - deploy

  tasks:
    - name: "Configure VLANs"
      cisco.nxos.nxos_vlans:
        ...        # Inherits: nxos, deploy
```

Play-level tags are the right choice when an entire play is platform-specific — everything in the IOS play is `ios`, everything in the NX-OS play is `nxos`. No need to tag every task individually.

### Tag Inheritance Summary

```
Play tags   →  inherited by all blocks and tasks in the play
Block tags  →  inherited by all tasks in the block
Task tags   →  apply only to that task

Final tag set for a task = play tags + block tags + task tags (combined)
```

---

## 14.3 — Special Tags: `always` and `never`

### `always` — Run Regardless of `--tags`

A task tagged `always` runs no matter what other tags I specify. It ignores tag filtering entirely — the only way to suppress it is with `--skip-tags always`.

```yaml
- name: "Pre-flight | Gather device facts"
  cisco.ios.ios_facts:
    gather_subset: default
  tags:
    - always    # ← Runs even when I use --tags bgp or --tags vlans
                #   Facts are needed by every other task, so they always run

- name: "Pre-flight | Verify required variables"
  ansible.builtin.assert:
    that:
      - device_hostname is defined
      - ntp_servers is defined
  tags:
    - always    # ← Safety check — should never be skipped
```

Use `always` for:
- Fact gathering tasks that other tasks depend on
- Pre-flight assertion checks that validate the environment before changes
- Tasks that set up state needed by all other tasks

### `never` — Skip Unless Explicitly Requested

A task tagged `never` is skipped in normal runs. It only runs when explicitly requested with `--tags never` or `--tags <specific_tag>` where that specific tag is also on the task.

```yaml
- name: "Dangerous | Factory reset device"
  cisco.ios.ios_config:
    lines:
      - write erase
  tags:
    - never          # ← Never runs in a normal playbook execution
    - factory_reset  # ← Only runs if someone explicitly uses --tags factory_reset

- name: "Debug | Dump all host variables"
  ansible.builtin.debug:
    var: hostvars[inventory_hostname]
  tags:
    - never    # ← Development debug task — doesn't run in normal operation
    - debug_vars
```

Use `never` for:
- Destructive operations (wipe, reset, erase) that should never run accidentally
- Development debug tasks that produce verbose output
- Emergency procedures that require explicit invocation
- Tasks that are available but deliberately opt-out of normal execution

### ### ⚠️ Warning

> `--tags always` runs ONLY tasks tagged `always` — it does not mean "run everything including always tasks." This is counterintuitive. If I want to run all tasks including `always`-tagged ones, I just run the playbook normally without any `--tags`. The `always` tag's only purpose is to make a task immune to other tag filters. `--skip-tags always` is the one command that can suppress `always`-tagged tasks.

---

## 14.4 — A Fresh Example: The Tagged Network Playbook

Rather than retrofitting existing playbooks, here is a purpose-built playbook that demonstrates every tagging pattern in context. This becomes the template I follow for all future playbooks.

```bash
nano ~/projects/ansible-network/playbooks/deploy/site_tagged.yml
```

```yaml
---
# =============================================================
# site_tagged.yml — Full network deployment with comprehensive tags
# Demonstrates: task tags, block tags, play tags, always, never
#
# Common invocations:
#   Full run:          ansible-playbook site_tagged.yml
#   Backup only:       ansible-playbook site_tagged.yml --tags backup
#   IOS only:          ansible-playbook site_tagged.yml --tags ios
#   NTP on IOS:        ansible-playbook site_tagged.yml --tags "ntp,ios"
#   Skip backup:       ansible-playbook site_tagged.yml --skip-tags backup
#   Validate only:     ansible-playbook site_tagged.yml --tags validate
#   List all tasks:    ansible-playbook site_tagged.yml --list-tasks
# =============================================================


# ── Play 1: Backup (runs first, before any changes) ─────────────────
- name: "Backup | Configuration backup — all network devices"
  hosts: network_devices
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  tags:
    - backup        # ← Entire play tagged — skip with --skip-tags backup

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "Backup | Create backup directory"
      ansible.builtin.file:
        path: "backups/{{ ansible_network_os | split('.') | first }}/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Backup | Gather running config (IOS)"
      cisco.ios.ios_command:
        commands: [show running-config]
      register: ios_config
      when: ansible_network_os == 'cisco.ios.ios'

    - name: "Backup | Gather running config (NX-OS)"
      cisco.nxos.nxos_command:
        commands: [show running-config]
      register: nxos_config
      when: ansible_network_os == 'cisco.nxos.nxos'

    - name: "Backup | Write IOS config to file"
      ansible.builtin.copy:
        content: "{{ ios_config.stdout[0] }}"
        dest: "backups/cisco/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      when: ansible_network_os == 'cisco.ios.ios'
      delegate_to: localhost

    - name: "Backup | Write NX-OS config to file"
      ansible.builtin.copy:
        content: "{{ nxos_config.stdout[0] }}"
        dest: "backups/cisco/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      when: ansible_network_os == 'cisco.nxos.nxos'
      delegate_to: localhost


# ── Play 2: IOS-XE deployment ────────────────────────────────────────
- name: "Deploy | IOS-XE configuration"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  force_handlers: true
  tags:
    - ios           # ← All tasks inherit this tag
    - deploy

  tasks:

    # always tag — runs regardless of --tags filter
    - name: "Pre-flight | Gather IOS device facts"
      cisco.ios.ios_facts:
        gather_subset: default
      tags:
        - always    # ← Cannot be skipped by tag filtering (only by --skip-tags always)

    - name: "Pre-flight | Assert required variables exist"
      ansible.builtin.assert:
        that:
          - device_hostname is defined
          - ntp_servers is defined
      tags:
        - always

    # Base configuration block
    - block:
        - name: "Base | Set hostname"
          cisco.ios.ios_hostname:
            config:
              hostname: "{{ device_hostname }}"
            state: merged
          notify: "Save IOS configuration"

        - name: "Base | Set IP domain name"
          cisco.ios.ios_config:
            lines:
              - "ip domain-name {{ domain_name }}"
          notify: "Save IOS configuration"

        - name: "Base | Enable password encryption"
          cisco.ios.ios_config:
            lines:
              - service password-encryption
          notify: "Save IOS configuration"

      tags:
        - base
        - identity
      # All tasks in this block carry: ios, deploy, base, identity

    # NTP block
    - block:
        - name: "NTP | Configure NTP servers"
          cisco.ios.ios_ntp_global:
            config:
              servers:
                - server: "{{ item }}"
                  prefer: "{{ true if loop.index == 1 else false }}"
            state: merged
          loop: "{{ ntp_servers }}"
          loop_control:
            label: "NTP: {{ item }}"
          notify: "Save IOS configuration"

        - name: "NTP | Set NTP source interface"
          cisco.ios.ios_config:
            lines:
              - "ntp source {{ ntp_source_interface }}"
          when: ntp_source_interface is defined
          notify: "Save IOS configuration"

      tags:
        - ntp
        - base

    # Logging block
    - block:
        - name: "Logging | Configure syslog"
          cisco.ios.ios_logging_global:
            config:
              hosts:
                - hostname: "{{ syslog_server }}"
                  severity: informational
            state: merged
          notify: "Save IOS configuration"

        - name: "Logging | Set logging buffer"
          cisco.ios.ios_config:
            lines:
              - "logging buffered 16384 informational"
          notify: "Save IOS configuration"

      tags:
        - logging
        - base

    # SNMP block
    - block:
        - name: "SNMP | Configure community string"
          cisco.ios.ios_config:
            lines:
              - "snmp-server community {{ snmp_community_ro }} RO"
          no_log: true
          notify: "Save IOS configuration"

        - name: "SNMP | Configure location and contact"
          cisco.ios.ios_config:
            lines:
              - "snmp-server location {{ snmp_location | default('Unknown') }}"
              - "snmp-server contact {{ snmp_contact | default('netops@lab.local') }}"
          notify: "Save IOS configuration"

      tags:
        - snmp
        - base

    # Routing block
    - block:
        - name: "Routing | Configure OSPF"
          cisco.ios.ios_ospfv2:
            config:
              processes:
                - process_id: "{{ ospf.process_id }}"
                  router_id: "{{ ospf.router_id }}"
            state: merged
          when: ospf is defined
          notify: "Save IOS configuration"

        - name: "Routing | Configure BGP"
          cisco.ios.ios_bgp_global:
            config:
              as_number: "{{ bgp.as_number }}"
              bgp:
                router_id:
                  address: "{{ bgp.router_id }}"
                log_neighbor_changes: true
            state: merged
          when: bgp is defined
          notify: "Save IOS configuration"

      tags:
        - routing
        - bgp
        - ospf

    # Validation block — read-only, never makes changes
    - block:
        - name: "Validate | Check IOS version"
          cisco.ios.ios_command:
            commands: [show version]
          register: ios_version

        - name: "Validate | Assert device responded"
          ansible.builtin.assert:
            that:
              - "'Cisco IOS' in ios_version.stdout[0] or 'IOS-XE' in ios_version.stdout[0]"
            fail_msg: "Device {{ inventory_hostname }} did not return expected IOS output"
            success_msg: "PASS: {{ inventory_hostname }} is running IOS/IOS-XE"

        - name: "Validate | Check BGP neighbors"
          cisco.ios.ios_command:
            commands: [show ip bgp summary]
          register: bgp_summary
          when: bgp is defined

        - name: "Validate | Display BGP summary"
          ansible.builtin.debug:
            msg: "{{ bgp_summary.stdout_lines[0] }}"
          when: bgp is defined and bgp_summary is not skipped

      tags:
        - validate
        - ios_validate

    # Debug task — never runs in normal operation
    - name: "Debug | Dump all variables for this host"
      ansible.builtin.debug:
        var: hostvars[inventory_hostname]
      tags:
        - never
        - debug_vars

  handlers:
    - name: "Save IOS configuration"
      cisco.ios.ios_command:
        commands: [write memory]
      listen: "Save IOS configuration"


# ── Play 3: NX-OS deployment ─────────────────────────────────────────
- name: "Deploy | NX-OS configuration"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli
  force_handlers: true
  tags:
    - nxos          # ← All tasks inherit this tag
    - deploy

  tasks:

    - name: "Pre-flight | Gather NX-OS facts"
      cisco.nxos.nxos_facts:
        gather_subset: default
      tags:
        - always

    - block:
        - name: "VLANs | Configure base VLANs"
          cisco.nxos.nxos_vlans:
            config:
              - vlan_id: "{{ item.id }}"
                name: "{{ item.name }}"
            state: merged
          loop: "{{ base_vlans }}"
          loop_control:
            label: "VLAN {{ item.id }} — {{ item.name }}"
          notify: "Save NX-OS configuration"

      tags:
        - vlans
        - base

    - block:
        - name: "Features | Enable required NX-OS features"
          cisco.nxos.nxos_feature:
            feature: "{{ item }}"
            state: enabled
          loop: "{{ nxos_features_required | default([]) }}"
          loop_control:
            label: "feature: {{ item }}"
          notify: "Save NX-OS configuration"

      tags:
        - features
        - base

    - block:
        - name: "Validate | Check VLAN state"
          cisco.nxos.nxos_command:
            commands: [show vlan brief]
          register: vlan_brief

        - name: "Validate | Display VLAN summary"
          ansible.builtin.debug:
            msg: "{{ vlan_brief.stdout_lines[0] }}"

      tags:
        - validate
        - nxos_validate

  handlers:
    - name: "Save NX-OS configuration"
      cisco.nxos.nxos_command:
        commands: [copy running-config startup-config]
      listen: "Save NX-OS configuration"


# ── Play 4: Cross-platform report ────────────────────────────────────
- name: "Report | Device inventory and status"
  hosts: network_devices
  gather_facts: false
  connection: network_cli
  tags:
    - report        # ← Skip with --skip-tags report (or run alone with --tags report)

  tasks:
    - name: "Report | Gather facts for report"
      cisco.ios.ios_facts:
        gather_subset: default
      when: ansible_network_os == 'cisco.ios.ios'

    - name: "Report | Gather NX-OS facts for report"
      cisco.nxos.nxos_facts:
        gather_subset: default
      when: ansible_network_os == 'cisco.nxos.nxos'

    - name: "Report | Display device summary"
      ansible.builtin.debug:
        msg:
          - "Device:  {{ inventory_hostname }}"
          - "Host:    {{ ansible_net_hostname | default('unknown') }}"
          - "Version: {{ ansible_net_version | default('unknown') }}"
```

---

## 14.5 — Tag Taxonomy for Network Automation

A consistent tagging taxonomy makes every playbook in the project predictable. Here is the full taxonomy I use throughout this guide — every tag, its meaning, and when to use it.

### Tier 1 — Phase Tags (What Stage of the Workflow)

| Tag | Meaning | `--tags` Use Case |
|---|---|---|
| `backup` | Configuration backup tasks | Run backup before a change window |
| `deploy` | Configuration push tasks | Run only deployment, skip validation |
| `validate` | Validation and assertion tasks | Run checks without making changes |
| `rollback` | Rollback and restore tasks | Disaster recovery — explicit invocation only |
| `report` | Reporting and fact-gathering tasks | Generate reports without touching config |

### Tier 2 — Platform Tags (Which Device Type)

| Tag | Platform | `--tags` Use Case |
|---|---|---|
| `ios` | Cisco IOS / IOS-XE | Deploy only to IOS devices |
| `nxos` | Cisco NX-OS | Deploy only to NX-OS switches |
| `junos` | Juniper JunOS | Deploy only to Juniper devices |
| `panos` | Palo Alto PAN-OS | Deploy only to firewalls |

### Tier 3 — Feature Tags (Which Configuration Domain)

| Tag | Domain | `--tags` Use Case |
|---|---|---|
| `base` | Core device settings (hostname, NTP, logging) | Apply baseline config |
| `identity` | Hostname, domain name | Change hostnames across fleet |
| `ntp` | NTP configuration | Push NTP server updates |
| `logging` | Syslog and local log buffer | Update syslog server |
| `snmp` | SNMP community and traps | Rotate SNMP communities |
| `routing` | All routing protocols | Apply routing changes |
| `ospf` | OSPF specifically | OSPF-only changes |
| `bgp` | BGP specifically | BGP-only changes |
| `vlans` | VLAN configuration | Add/remove VLANs |
| `interfaces` | Interface configuration | Interface changes |
| `acls` | Access control lists | ACL updates |
| `security` | Security-related config | Security hardening pass |
| `features` | NX-OS feature enablement | Enable platform features |

### Special Tags

| Tag | Behavior |
|---|---|
| `always` | Runs regardless of `--tags` — use for pre-flight and fact gathering |
| `never` | Skipped unless explicitly requested — use for dangerous or debug tasks |

---

## 14.6 — Realistic CLI Invocations

This is the section I come back to during actual operations. Each invocation reflects a real task I need to do.

### Change Window Operations

```bash
# Take a pre-change backup of all network devices
ansible-playbook playbooks/deploy/site_tagged.yml --tags backup

# Run full deployment (backup runs first automatically)
ansible-playbook playbooks/deploy/site_tagged.yml

# Run full deployment but skip backup (already done separately)
ansible-playbook playbooks/deploy/site_tagged.yml --skip-tags backup

# Deploy only to IOS-XE routers, nothing else
ansible-playbook playbooks/deploy/site_tagged.yml --tags ios

# Deploy only to NX-OS switches
ansible-playbook playbooks/deploy/site_tagged.yml --tags nxos

# Deploy to a single device
ansible-playbook playbooks/deploy/site_tagged.yml --limit wan-r1
```

### Targeted Configuration Updates

```bash
# Push NTP changes across all IOS devices
ansible-playbook playbooks/deploy/site_tagged.yml --tags "ntp,ios"

# Update SNMP community on all devices (both platforms)
ansible-playbook playbooks/deploy/site_tagged.yml --tags snmp

# Push VLAN changes to NX-OS only
ansible-playbook playbooks/deploy/site_tagged.yml --tags "vlans,nxos"

# Apply routing changes to IOS devices only
ansible-playbook playbooks/deploy/site_tagged.yml --tags "routing,ios"

# Deploy BGP config to a single router
ansible-playbook playbooks/deploy/site_tagged.yml --tags bgp --limit wan-r1
```

### Validation Without Changes

```bash
# Run all validation tasks, no configuration changes
ansible-playbook playbooks/deploy/site_tagged.yml --tags validate

# Validate IOS devices only
ansible-playbook playbooks/deploy/site_tagged.yml --tags "validate,ios"

# Generate report without touching devices
ansible-playbook playbooks/deploy/site_tagged.yml --tags report
```

### Planning and Dry Runs

```bash
# List all tasks that would run (no execution)
ansible-playbook playbooks/deploy/site_tagged.yml --list-tasks

# List all tasks that --tags ntp would run
ansible-playbook playbooks/deploy/site_tagged.yml --tags ntp --list-tasks

# Dry run — predict changes without applying (where --check is reliable)
ansible-playbook playbooks/deploy/site_tagged.yml --tags vlans --check --diff

# List all hosts that would be targeted
ansible-playbook playbooks/deploy/site_tagged.yml --list-hosts
```

### Development and Debugging

```bash
# Run the debug task that's tagged 'never' — explicitly requested
ansible-playbook playbooks/deploy/site_tagged.yml --tags debug_vars --limit wan-r1

# Run everything including never-tagged tasks (unusual — development only)
ansible-playbook playbooks/deploy/site_tagged.yml --tags all

# Run only pre-flight checks (tagged always)
ansible-playbook playbooks/deploy/site_tagged.yml --tags always
```

### ### 💡 Tip

> I keep a `RUNBOOK.md` or a comment block at the top of `site.yml` with the most common invocations — the 10 commands the team uses 90% of the time. Engineers who are new to the project can see immediately what `--tags backup`, `--tags validate`, and `--tags deploy` do without reading the entire playbook. The comment block at the top of `site_tagged.yml` above is the template I use for every master playbook.

---

## 14.7 — Tags with Roles

Tags attach to role calls just like tasks. A tag on a role call applies to every task inside the role:

```yaml
- name: "Deploy | Multi-role playbook"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  force_handlers: true

  roles:
    - role: cisco_ios_base
      tags:
        - base
        - ios          # Every task in cisco_ios_base gets: base, ios

    - role: cisco_ios_interfaces
      tags:
        - interfaces
        - ios          # Every task in cisco_ios_interfaces gets: interfaces, ios

    - role: cisco_ios_routing
      tags:
        - routing
        - ios
```

But this only works fully with `import_role` or the `roles:` key — **not** `include_role`. As covered in Part 13, `include_role` is dynamic and tag inheritance doesn't propagate through it the same way. For the `roles:` key and `import_role`, tags propagate completely into the role's tasks.

### Tags Defined Inside the Role

Role tasks can also have their own tags (as shown in Part 13 — each task file in `cisco_ios_base` carries its own `ntp`, `logging`, `snmp` tags). These combine with any tags on the role call:

```
Role call tag: ios, base
  + Task tag in role: ntp
  = Final task tags: ios, base, ntp
```

So `--tags ntp` runs NTP tasks inside the role, and those tasks also carry `ios` and `base` from the role call. `--tags base` runs every task in the role because they all inherit `base`. This composability is the reason for the two-tier tag approach — phase tags on the role call, feature tags inside role tasks.

---

## 14.8 — Common Gotchas

### ### 🪲 Gotcha — `--tags` matches ANY tag, not ALL tags

`--tags "ntp,ios"` does **not** mean "tasks tagged both ntp AND ios." It means "tasks tagged ntp OR tasks tagged ios." The comma is OR, not AND.

```bash
# This runs ALL ntp-tagged tasks AND ALL ios-tagged tasks
# (union of both sets)
ansible-playbook site.yml --tags "ntp,ios"

# If I want tasks tagged BOTH ntp and ios, there's no direct syntax for that.
# The solution is to design task tags so the combination is unambiguous:
# a task tagged ntp on an ios play already carries both tags through inheritance.
# Running --tags ntp against only the IOS play gives the "AND" behavior indirectly.
ansible-playbook site.yml --tags ntp --limit cisco_ios
```

### ### 🪲 Gotcha — `--tags always` doesn't mean "run all tasks"

```bash
# ❌ Wrong mental model — this runs ONLY 'always' tagged tasks
ansible-playbook site.yml --tags always

# ✅ To run ALL tasks (including always-tagged ones): just run normally
ansible-playbook site.yml

# ✅ To run a specific tag AND always-tagged tasks:
ansible-playbook site.yml --tags "ntp,always"
# (always-tagged tasks always run anyway, so this is redundant but explicit)
```

### ### 🪲 Gotcha — Tags on `include_tasks` don't propagate to tasks inside

```yaml
# ❌ This tags the include statement itself, not the tasks inside ntp.yml
- ansible.builtin.include_tasks: ntp.yml
  tags: ntp

# The tasks inside ntp.yml must have their own tags for --tags ntp to reach them.
# OR use import_tasks (static) where tags DO propagate:
- ansible.builtin.import_tasks: ntp.yml
  tags: ntp    # ← With import_tasks, this tag reaches every task in ntp.yml
```

This is why the `cisco_ios_base` role in Part 13 has tags on each `include_tasks:` call AND each included task file has its own tags internally. The `include_tasks` tags apply to the include decision itself; the internal tags apply to the tasks when they run.

### ### 🪲 Gotcha — Forgetting `--tags always` suppresses `always`-tagged tasks

```bash
# I want to run only NTP tasks, but facts are tagged always
ansible-playbook site.yml --tags ntp
# Result: always-tagged facts tasks run, then ntp tasks run ✅

# I accidentally skip facts
ansible-playbook site.yml --tags ntp --skip-tags always
# Result: facts tasks are skipped, NTP tasks fail because ansible_net_* vars don't exist ❌
```

The `--skip-tags always` invocation is almost never the right choice. The only time it's appropriate is in a dry-run scenario where I explicitly don't want fact gathering to consume time and I know the variables are already available from a previous run.

---


Tags make large playbooks practical — they're the difference between "I have to run everything" and "I run exactly what I need." From Part 15 onward, every playbook and role carries a full tag set following the taxonomy established here. Part 15 covers network resource module states — merged, replaced, overridden, deleted — the behavior that determines whether Ansible adds to, replaces, or removes network configuration.


