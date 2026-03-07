---
draft: false
title: '20 - JunOS'
description: "Part 20 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 20
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 20: Juniper vJunos Network Automation

> *Junos is architecturally different from IOS and NX-OS in one fundamental way: configuration is never applied directly to the running system. Every change goes into a candidate configuration first, and nothing takes effect until you commit. This changes how Ansible interacts with the device, how templates are structured, and how rollback works. Once that model is understood, Junos automation is actually cleaner than IOS — the hierarchy maps naturally to YAML data structures, and the commit mechanism gives built-in safety that IOS and NX-OS don't have.*

---

## 20.1 — The Junos Configuration Model

### Candidate Config vs Running Config

On IOS and NX-OS, every command takes effect immediately. `interface GigabitEthernet1 / ip address 10.0.0.1 255.255.255.0` — the IP is live the moment you press Enter. If something goes wrong mid-configuration, the device is in a partially configured state.

Junos works differently:

```
IOS/NX-OS model:
  Command entered → Takes effect immediately → Running config updated
  No undo unless you manually reverse each command

Junos model:
  Command entered → Goes into candidate config → Running config UNCHANGED
  Candidate config accumulates changes (edit, set, delete)
  'commit' → Candidate validated → Becomes new running config (atomic)
  'rollback 0' → Discard candidate, return to last committed state
```

The candidate config is a complete copy of the configuration sitting in memory. I can make 50 changes to it, review the diff with `show | compare`, and then commit all 50 atomically — or discard all 50 with a single `rollback 0`. There's no concept of "half-applied" configuration.

### How This Affects Ansible

When Ansible connects to a Junos device using the `junos` connection plugin, it opens a NETCONF session (not SSH CLI). NETCONF is an XML-based network management protocol that Junos has supported since 2004. The `junipernetworks.junos` collection uses NETCONF to:

1. Lock the candidate configuration (prevents concurrent changes)
2. Load new configuration into the candidate (merge, replace, or override)
3. Commit the candidate
4. Unlock

This means the `junos_config` module with `src:` doesn't send CLI commands line by line — it sends the entire configuration block as a structured payload, and Junos validates and commits it atomically. If any part of the configuration is invalid, the entire commit fails and nothing is applied.

```
Ansible task runs
      │
      ▼
NETCONF session opens (port 830, not 22)
      │
      ▼
Candidate config locked
      │
      ▼
New config loaded into candidate (merge/replace/override)
      │
      ▼
Commit validation (Junos checks the whole config for consistency)
      │
      ├── Valid → Commit → Running config updated → NETCONF closes
      └── Invalid → Commit fails → Candidate discarded → Nothing changed
```

### Junos Configuration Formats

Junos config can be expressed in three formats — all equivalent, all interchangeable:

**Hierarchical (native Junos format):**
```
interfaces {
    ge-0/0/0 {
        description "WAN | To FW-01";
        unit 0 {
            family inet {
                address 10.10.10.1/30;
            }
        }
    }
}
```

**Set format (one command per line):**
```
set interfaces ge-0/0/0 description "WAN | To FW-01"
set interfaces ge-0/0/0 unit 0 family inet address 10.10.10.1/30
```

**XML (NETCONF native):**
```xml
<interfaces>
  <interface>
    <name>ge-0/0/0</name>
    <description>WAN | To FW-01</description>
    <unit><name>0</name><family><inet><address><name>10.10.10.1/30</name></address></inet></family></unit>
  </interface>
</interfaces>
```

For Ansible automation with `junos_config src:`, I use **set format**. It's the most compact, easiest to generate from Jinja2 templates, and maps cleanly to a data model. The `src:` file contains set-format commands — one per line — and Junos applies them as a batch to the candidate config.

---

## 20.2 — Collection Setup

```bash
# Install the Juniper collection
ansible-galaxy collection install junipernetworks.junos

# Junos modules use NETCONF — install the Python NETCONF library
pip install ncclient --break-system-packages

# Verify
python3 -c "import ncclient; print('ncclient OK')"
```

### Group Variables for Junos Devices

```bash
cat > ~/projects/ansible-network/inventory/group_vars/junos_devices.yml << 'EOF'
---
ansible_network_os: junipernetworks.junos.junos
ansible_connection: ansible.netcommon.netconf
ansible_port: 830              # NETCONF port — not 22
# ansible_user and ansible_password come from vault

# Junos-specific settings
junos_commit_timeout: 120      # Seconds before confirmed-commit auto-rolls back
junos_config_format: set       # Use set-format for all config pushes
EOF
```

The critical difference: `ansible_connection: ansible.netcommon.netconf` and `ansible_port: 830`. Junos automation does not use the `network_cli` plugin — it uses NETCONF. This means no SSH CLI prompt detection, no `ios_command`-style show command parsing with wait_for — NETCONF operations are XML request/response.

---

## 20.3 — The Junos Data Model

The data model follows the same pattern as Parts 17-19 but maps to Junos hierarchy rather than IOS flat CLI.

```bash
cat > ~/projects/ansible-network/inventory/host_vars/vjunos-01.yml << 'EOF'
---
# =============================================================
# vjunos-01 — Juniper vJunos router
# Containerlab: vrnetlab/vr-vjunos image
# =============================================================

device_hostname: vjunos-01
device_role: wan_router
ansible_host: 172.16.0.41

# ── Interfaces ────────────────────────────────────────────────────
# Junos interface naming: ge-0/0/N (physical), lo0 (loopback)
# Units: Junos uses logical units under each interface (unit 0 is standard)
interfaces:
  ge-0/0/0:
    description: "WAN | To upstream provider"
    units:
      - id: 0
        family: inet
        address: 10.10.30.1
        prefix: 30
    shutdown: false
  ge-0/0/1:
    description: "LAN | Internal network"
    units:
      - id: 0
        family: inet
        address: 192.168.10.1
        prefix: 24
    shutdown: false
  ge-0/0/2:
    description: "Peering | To wan-r1"
    units:
      - id: 0
        family: inet
        address: 10.20.30.1
        prefix: 30
    shutdown: false
  lo0:
    description: "Loopback | Router-ID"
    units:
      - id: 0
        family: inet
        address: 10.255.1.1
        prefix: 32

# ── Routing Instances (Junos equivalent of VRFs) ──────────────────
routing_instances:
  - name: MGMT
    type: virtual-router    # virtual-router = no RD/RT, local VRF
    description: "Management routing instance"
    interfaces:
      - ge-0/0/3.0          # Management interface unit
    routing_options:
      static:
        - destination: 0.0.0.0/0
          next_hop: 172.16.99.1

# ── OSPF ──────────────────────────────────────────────────────────
ospf:
  areas:
    - id: 0.0.0.0
      interfaces:
        - name: ge-0/0/1.0
          interface_type: p2p
          passive: false
        - name: ge-0/0/2.0
          interface_type: p2p
          passive: false
        - name: lo0.0
          passive: true

# ── BGP ───────────────────────────────────────────────────────────
bgp:
  as_number: 65300
  router_id: 10.255.1.1
  groups:
    - name: UPSTREAM
      type: external
      local_as: 65300
      neighbors:
        - ip: 10.10.30.2
          peer_as: 65200
          description: "eBGP to upstream provider"
          export: EXPORT_TO_UPSTREAM
          import: IMPORT_FROM_UPSTREAM
    - name: INTERNAL
      type: internal
      local_as: 65300
      neighbors:
        - ip: 10.255.0.1      # wan-r1 loopback
          description: "iBGP to wan-r1"
          local_address: 10.255.1.1

# ── Firewall Filters (Junos ACL equivalent) ───────────────────────
firewall_filters:
  - name: PROTECT_RE
    family: inet
    terms:
      - name: ALLOW_SSH
        from:
          protocol: tcp
          destination_port: 22
          source_prefix_list: MGMT_PREFIXES
        then: accept
      - name: ALLOW_NETCONF
        from:
          protocol: tcp
          destination_port: 830
          source_prefix_list: MGMT_PREFIXES
        then: accept
      - name: ALLOW_NTP
        from:
          protocol: udp
          source_port: 123
        then: accept
      - name: ALLOW_OSPF
        from:
          protocol: ospf
        then: accept
      - name: ALLOW_BGP
        from:
          protocol: tcp
          destination_port: 179
        then: accept
      - name: DENY_ALL
        from: {}
        then:
          action: discard
          log: true
          syslog: true

  - name: INTERNET_INBOUND
    family: inet
    terms:
      - name: DENY_RFC1918
        from:
          source_address:
            - 10.0.0.0/8
            - 172.16.0.0/12
            - 192.168.0.0/16
        then: discard
      - name: ALLOW_ESTABLISHED
        from:
          tcp_established: true
        then: accept
      - name: DENY_DEFAULT
        from: {}
        then: discard

# ── Prefix Lists (referenced by firewall filters and BGP policy) ──
prefix_lists:
  - name: MGMT_PREFIXES
    prefixes:
      - 192.168.1.0/24
      - 172.16.0.0/24
  - name: ADVERTISED_PREFIXES
    prefixes:
      - 10.255.1.1/32
      - 192.168.10.0/24

# ── Policy Statements (Junos route-map equivalent) ────────────────
policy_statements:
  - name: EXPORT_TO_UPSTREAM
    terms:
      - name: ALLOW_MINE
        from:
          prefix_list: ADVERTISED_PREFIXES
          protocol: bgp
        then:
          action: accept
      - name: DENY_ALL
        from: {}
        then:
          action: reject

  - name: IMPORT_FROM_UPSTREAM
    terms:
      - name: ACCEPT_DEFAULT
        from:
          route_filter:
            - prefix: 0.0.0.0/0
              match_type: exact
        then:
          action: accept
          local_preference: 200
      - name: DENY_RFC1918
        from:
          prefix_list: DENY_RFC1918_PREFIXES
        then:
          action: reject
      - name: ACCEPT_REST
        from: {}
        then:
          action: accept

# ── Users ─────────────────────────────────────────────────────────
local_users:
  - username: ansible
    class: super-user
    auth_method: plain-text-password
    password: "{{ vault_ansible_password }}"
    ssh_public_key: "{{ lookup('file', '~/.ssh/ansible_ed25519.pub') }}"

# ── System Services ───────────────────────────────────────────────
system_services:
  ssh:
    root_login: deny
    protocol_version: v2
    max_sessions_per_connection: 32
  netconf:
    ssh: true              # NETCONF over SSH (port 830)
EOF
```

---

## 20.4 — The Junos Template Architecture

Rather than building playbooks with individual `junos_config` tasks per feature (which maps poorly to Junos hierarchy), I generate a **complete set-format configuration file** from a Jinja2 template and push it with a single `junos_config src:` call. This approach:

- Matches how Junos actually works (atomic config replacement)
- Produces reviewable output before any push
- Makes idempotency straightforward — same data model produces identical output
- Generates proper set-format hierarchy naturally from nested YAML

### The Master Set-Format Template

```bash
mkdir -p ~/projects/ansible-network/templates/junos

cat > ~/projects/ansible-network/templates/junos/vjunos_config.j2 << 'TEMPLATE'
{# =============================================================
   templates/junos/vjunos_config.j2
   Generates set-format Junos configuration from host_vars.
   Push with: junos_config src: rendered_file.set update: set
   ============================================================= #}

{# ── System Identity ─────────────────────────────────────────── #}
set system host-name {{ device_hostname }}
set system domain-name {{ domain_name }}
set system time-zone UTC

{# ── System Services ─────────────────────────────────────────── #}
{% if system_services is defined %}
{% if system_services.ssh is defined %}
set system services ssh root-login {{ system_services.ssh.root_login | default('deny') }}
set system services ssh protocol-version v2
{% if system_services.ssh.max_sessions_per_connection is defined %}
set system services ssh max-sessions-per-connection {{ system_services.ssh.max_sessions_per_connection }}
{% endif %}
{% endif %}
{% if system_services.netconf is defined and system_services.netconf.ssh %}
set system services netconf ssh
{% endif %}
{% endif %}

{# ── Local Users ──────────────────────────────────────────────── #}
{% for user in local_users | default([]) %}
set system login user {{ user.username }} class {{ user.class }}
set system login user {{ user.username }} authentication plain-text-password-value {{ user.password }}
{% if user.ssh_public_key is defined %}
set system login user {{ user.username }} authentication ssh-ed25519 "{{ user.ssh_public_key }}"
{% endif %}
{% endfor %}

{# ── NTP ──────────────────────────────────────────────────────── #}
{% for ntp in ntp_servers | default([]) %}
set system ntp server {{ ntp }}
{% endfor %}

{# ── Syslog ───────────────────────────────────────────────────── #}
{% if syslog_server is defined %}
set system syslog host {{ syslog_server }} any info
set system syslog host {{ syslog_server }} structured-data
{% endif %}
set system syslog file messages any notice
set system syslog file messages authorization info

{# ── Interfaces ───────────────────────────────────────────────── #}
{% for iface_name, iface in interfaces.items() %}
{% if iface.description is defined %}
set interfaces {{ iface_name }} description "{{ iface.description }}"
{% endif %}
{% if iface.shutdown | default(false) %}
set interfaces {{ iface_name }} disable
{% endif %}
{% for unit in iface.units | default([]) %}
{% if unit.family is defined %}
set interfaces {{ iface_name }} unit {{ unit.id }} family {{ unit.family }} address {{ unit.address }}/{{ unit.prefix }}
{% else %}
{# Loopback or family-less unit #}
set interfaces {{ iface_name }} unit {{ unit.id }} family inet address {{ unit.address }}/{{ unit.prefix }}
{% endif %}
{% endfor %}
{% endfor %}

{# ── Prefix Lists ─────────────────────────────────────────────── #}
{% for pl in prefix_lists | default([]) %}
{% for prefix in pl.prefixes %}
set policy-options prefix-list {{ pl.name }} {{ prefix }}
{% endfor %}
{% endfor %}

{# ── Policy Statements (route-map equivalent) ─────────────────── #}
{% for ps in policy_statements | default([]) %}
{% for term in ps.terms %}
set policy-options policy-statement {{ ps.name }} term {{ term.name }} then {{ term.then.action }}
{% if term.from is defined and term.from %}
{% if term.from.prefix_list is defined %}
set policy-options policy-statement {{ ps.name }} term {{ term.name }} from prefix-list {{ term.from.prefix_list }}
{% endif %}
{% if term.from.protocol is defined %}
set policy-options policy-statement {{ ps.name }} term {{ term.name }} from protocol {{ term.from.protocol }}
{% endif %}
{% if term.from.route_filter is defined %}
{% for rf in term.from.route_filter %}
set policy-options policy-statement {{ ps.name }} term {{ term.name }} from route-filter {{ rf.prefix }} {{ rf.match_type }}
{% endfor %}
{% endif %}
{% endif %}
{% if term.then.local_preference is defined %}
set policy-options policy-statement {{ ps.name }} term {{ term.name }} then local-preference {{ term.then.local_preference }}
{% endif %}
{% endfor %}
{% endfor %}

{# ── Firewall Filters ─────────────────────────────────────────── #}
{% for ff in firewall_filters | default([]) %}
{% for term in ff.terms %}
{% if term.from is defined and term.from %}
{% if term.from.protocol is defined %}
set firewall family {{ ff.family }} filter {{ ff.name }} term {{ term.name }} from protocol {{ term.from.protocol }}
{% endif %}
{% if term.from.destination_port is defined %}
set firewall family {{ ff.family }} filter {{ ff.name }} term {{ term.name }} from destination-port {{ term.from.destination_port }}
{% endif %}
{% if term.from.source_port is defined %}
set firewall family {{ ff.family }} filter {{ ff.name }} term {{ term.name }} from source-port {{ term.from.source_port }}
{% endif %}
{% if term.from.source_prefix_list is defined %}
set firewall family {{ ff.family }} filter {{ ff.name }} term {{ term.name }} from source-prefix-list {{ term.from.source_prefix_list }}
{% endif %}
{% if term.from.source_address is defined %}
{% for addr in term.from.source_address %}
set firewall family {{ ff.family }} filter {{ ff.name }} term {{ term.name }} from source-address {{ addr }}
{% endfor %}
{% endif %}
{% if term.from.tcp_established is defined and term.from.tcp_established %}
set firewall family {{ ff.family }} filter {{ ff.name }} term {{ term.name }} from tcp-established
{% endif %}
{% endif %}
{% if term.then is defined %}
{% if term.then is string %}
set firewall family {{ ff.family }} filter {{ ff.name }} term {{ term.name }} then {{ term.then }}
{% else %}
{% if term.then.action is defined %}
set firewall family {{ ff.family }} filter {{ ff.name }} term {{ term.name }} then {{ term.then.action }}
{% endif %}
{% if term.then.log | default(false) %}
set firewall family {{ ff.family }} filter {{ ff.name }} term {{ term.name }} then log
{% endif %}
{% if term.then.syslog | default(false) %}
set firewall family {{ ff.family }} filter {{ ff.name }} term {{ term.name }} then syslog
{% endif %}
{% endif %}
{% endif %}
{% endfor %}
{% endfor %}

{# ── OSPF ─────────────────────────────────────────────────────── #}
{% if ospf is defined %}
{% for area in ospf.areas %}
{% for iface in area.interfaces %}
{% if not iface.passive | default(false) %}
set protocols ospf area {{ area.id }} interface {{ iface.name }}{% if iface.interface_type is defined %} interface-type {{ iface.interface_type }}{% endif %}

{% else %}
set protocols ospf area {{ area.id }} interface {{ iface.name }} passive
{% endif %}
{% endfor %}
{% endfor %}
{% endif %}

{# ── BGP ──────────────────────────────────────────────────────── #}
{% if bgp is defined %}
set routing-options autonomous-system {{ bgp.as_number }}
set routing-options router-id {{ bgp.router_id }}
{% for group in bgp.groups | default([]) %}
set protocols bgp group {{ group.name }} type {{ group.type }}
{% if group.local_as is defined %}
set protocols bgp group {{ group.name }} local-as {{ group.local_as }}
{% endif %}
{% for neighbor in group.neighbors | default([]) %}
set protocols bgp group {{ group.name }} neighbor {{ neighbor.ip }} peer-as {{ neighbor.peer_as | default(group.local_as) }}
{% if neighbor.description is defined %}
set protocols bgp group {{ group.name }} neighbor {{ neighbor.ip }} description "{{ neighbor.description }}"
{% endif %}
{% if neighbor.export is defined %}
set protocols bgp group {{ group.name }} neighbor {{ neighbor.ip }} export {{ neighbor.export }}
{% endif %}
{% if neighbor.import is defined %}
set protocols bgp group {{ group.name }} neighbor {{ neighbor.ip }} import {{ neighbor.import }}
{% endif %}
{% if neighbor.local_address is defined %}
set protocols bgp group {{ group.name }} neighbor {{ neighbor.ip }} local-address {{ neighbor.local_address }}
{% endif %}
{% endfor %}
{% endfor %}
{% endif %}

{# ── Routing Instances ────────────────────────────────────────── #}
{% for ri in routing_instances | default([]) %}
set routing-instances {{ ri.name }} instance-type {{ ri.type }}
{% if ri.description is defined %}
set routing-instances {{ ri.name }} description "{{ ri.description }}"
{% endif %}
{% for iface in ri.interfaces | default([]) %}
set routing-instances {{ ri.name }} interface {{ iface }}
{% endfor %}
{% if ri.routing_options is defined %}
{% if ri.routing_options.static is defined %}
{% for route in ri.routing_options.static %}
set routing-instances {{ ri.name }} routing-options static route {{ route.destination }} next-hop {{ route.next_hop }}
{% endfor %}
{% endif %}
{% endif %}
{% endfor %}

{# ── Apply Firewall Filter to Loopback (protect Routing Engine) ─ #}
{% for ff in firewall_filters | default([]) %}
{% if ff.name == 'PROTECT_RE' %}
set interfaces lo0 unit 0 family inet filter input {{ ff.name }}
{% endif %}
{% endfor %}

{# End of configuration template #}
TEMPLATE
```

---

## 20.5 — The Junos Master Playbook

The playbook has three phases: render the template, review the diff, push with confirmed-commit and verify.

```bash
nano ~/projects/ansible-network/playbooks/deploy/deploy_junos.yml
```

```yaml
---
# =============================================================
# deploy_junos.yml — Juniper vJunos complete configuration
# Uses set-format template + junos_config src: for atomic push
#
# Phases:
#   1. Render: generate set-format config from template
#   2. Diff: show what would change (no commit)
#   3. Push: confirmed-commit with automatic rollback timer
#   4. Verify: confirm device is stable, commit or abort
#
# Common invocations:
#   Render only:    ansible-playbook deploy_junos.yml --tags render
#   Diff only:      ansible-playbook deploy_junos.yml --tags diff
#   Full push:      ansible-playbook deploy_junos.yml --tags push
#   Verify only:    ansible-playbook deploy_junos.yml --tags verify
#   Full workflow:  ansible-playbook deploy_junos.yml
# =============================================================

- name: "Junos | Phase 1 — Render configuration template"
  hosts: junos_devices
  gather_facts: false
  connection: local    # Template rendering happens on control node
  tags: [render, always]

  tasks:
    - name: "Render | Create output directory for rendered configs"
      ansible.builtin.file:
        path: "rendered/junos/{{ inventory_hostname }}"
        state: directory
        mode: '0755'

    - name: "Render | Generate set-format config from template"
      ansible.builtin.template:
        src: templates/junos/vjunos_config.j2
        dest: "rendered/junos/{{ inventory_hostname }}/{{ inventory_hostname }}.set"
        mode: '0644'

    - name: "Render | Display rendered config path"
      ansible.builtin.debug:
        msg: "Rendered config: rendered/junos/{{ inventory_hostname }}/{{ inventory_hostname }}.set"


- name: "Junos | Phase 2 — Configuration diff (no commit)"
  hosts: junos_devices
  gather_facts: false
  connection: ansible.netcommon.netconf
  tags: diff

  tasks:
    - name: "Diff | Load candidate config and show diff without committing"
      junipernetworks.junos.junos_config:
        src: "rendered/junos/{{ inventory_hostname }}/{{ inventory_hostname }}.set"
        src_format: set
        update: merge         # merge = add/update, don't delete unspecified config
        confirm: 0            # 0 = no confirm timer, just load and diff
        diff_format: text     # human-readable diff format
        commit: false         # ← Do NOT commit — show what would change only
      register: junos_diff
      check_mode: true        # Dry run — never modifies device

    - name: "Diff | Display configuration changes that would be applied"
      ansible.builtin.debug:
        msg: "{{ junos_diff.diff.prepared | default('No changes detected') }}"
      # junos_diff.diff.prepared contains the show | compare output
      # showing exactly what lines would be added/deleted


- name: "Junos | Phase 3 — Push with confirmed-commit"
  hosts: junos_devices
  gather_facts: false
  connection: ansible.netcommon.netconf
  tags: push

  vars:
    confirmed_commit_timeout: "{{ junos_commit_timeout | default(120) }}"
    # This is the safety net: if we don't send a final 'commit' within
    # confirmed_commit_timeout seconds, Junos automatically rolls back
    # the candidate config and restores the previous running config.
    # It's Junos' built-in "dead man's switch" for risky changes.

  tasks:
    - name: "Push | Gather pre-change facts"
      junipernetworks.junos.junos_facts:
        gather_subset:
          - default
      register: pre_change_facts

    - name: "Push | Display pre-change device state"
      ansible.builtin.debug:
        msg:
          - "Device:  {{ inventory_hostname }}"
          - "Version: {{ pre_change_facts.ansible_facts.ansible_net_version }}"
          - "Uptime:  {{ pre_change_facts.ansible_facts.ansible_net_uptime }}"
        verbosity: 1

    - name: "Push | Backup current config before changes"
      junipernetworks.junos.junos_command:
        commands:
          - show configuration
        display: text
      register: pre_change_config

    - name: "Push | Save pre-change backup"
      ansible.builtin.copy:
        content: "{{ pre_change_config.stdout[0] }}"
        dest: "backups/junos/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}.conf"
        mode: '0644'
      delegate_to: localhost

    - name: "Push | Load config with confirmed-commit ({{ confirmed_commit_timeout }}s timer)"
      junipernetworks.junos.junos_config:
        src: "rendered/junos/{{ inventory_hostname }}/{{ inventory_hostname }}.set"
        src_format: set
        update: merge
        confirm: "{{ confirmed_commit_timeout }}"
        # confirm: N means:
        #   - Config is committed to running immediately
        #   - A timer starts for N seconds
        #   - If a second 'commit' is NOT received within N seconds,
        #     Junos AUTOMATICALLY ROLLS BACK to the previous config
        #   - The verification play (Phase 4) sends the confirming commit
        comment: "Deployed by Ansible — {{ lookup('pipe', 'date') }}"
      register: push_result

    - name: "Push | Display commit result"
      ansible.builtin.debug:
        msg:
          - "Commit result: {{ 'SUCCESS' if push_result.changed else 'No changes' }}"
          - "Confirmed-commit timer: {{ confirmed_commit_timeout }} seconds"
          - "ACTION REQUIRED: Verification must complete within {{ confirmed_commit_timeout }}s"
          - "If verification fails, Junos will auto-rollback."


- name: "Junos | Phase 4 — Verify and confirm (or let rollback happen)"
  hosts: junos_devices
  gather_facts: false
  connection: ansible.netcommon.netconf
  tags: [push, verify]

  tasks:
    - name: "Verify | Check OSPF neighbor state"
      junipernetworks.junos.junos_command:
        commands:
          - show ospf neighbor
        display: text
      register: ospf_neighbors
      changed_when: false
      when: ospf is defined

    - name: "Verify | Assert OSPF neighbors are Full"
      ansible.builtin.assert:
        that:
          - "'Full' in ospf_neighbors.stdout[0]"
        fail_msg: >
          OSPF neighbors not in Full state on {{ inventory_hostname }}.
          Confirmed-commit timer will expire and Junos will auto-rollback.
          Manual intervention NOT required — rollback is automatic.
        success_msg: "PASS: OSPF neighbors Full on {{ inventory_hostname }}"
      when: ospf is defined and ospf_neighbors is not skipped

    - name: "Verify | Check BGP neighbor state"
      junipernetworks.junos.junos_command:
        commands:
          - show bgp neighbor
        display: text
      register: bgp_neighbors
      changed_when: false
      when: bgp is defined

    - name: "Verify | Assert BGP neighbors are Established"
      ansible.builtin.assert:
        that:
          - "'Established' in bgp_neighbors.stdout[0]"
        fail_msg: >
          BGP not established on {{ inventory_hostname }}.
          Junos will auto-rollback when confirmed-commit timer expires.
        success_msg: "PASS: BGP neighbors Established"
      when: bgp is defined and bgp_neighbors is not skipped

    - name: "Verify | Check device reachability via NETCONF"
      junipernetworks.junos.junos_command:
        commands:
          - show version
        display: text
      register: device_reachability
      changed_when: false

    - name: "Verify | Send confirming commit — locks in the change"
      junipernetworks.junos.junos_config:
        confirm_commit: true
        # confirm_commit: true sends the confirming 'commit' that
        # cancels the rollback timer and makes the change permanent.
        # This ONLY runs if all preceding assert tasks passed.
        # If any assert failed, the play stopped and this never runs.
        # The confirmed-commit timer will expire and Junos rolls back.
      when: device_reachability is succeeded
      register: confirm_result

    - name: "Verify | Display final status"
      ansible.builtin.debug:
        msg:
          - "═══════════════════════════════════════════════"
          - "DEPLOYMENT COMPLETE: {{ inventory_hostname }}"
          - "Config committed: PERMANENT (confirmed-commit sent)"
          - "OSPF: {{ 'PASS' if ospf is not defined or 'Full' in ospf_neighbors.stdout[0] | default('') else 'FAIL' }}"
          - "BGP:  {{ 'PASS' if bgp is not defined or 'Established' in bgp_neighbors.stdout[0] | default('') else 'FAIL' }}"
          - "═══════════════════════════════════════════════"
```

### The Confirmed-Commit Flow in Practice

```
Phase 3 runs:
  junos_config confirm: 120
    │
    ├── Config loads into candidate
    ├── Commit happens immediately
    ├── Config is LIVE on device
    └── 120-second countdown begins

Phase 4 runs (must complete within 120 seconds):
  Verification tasks run
    │
    ├── All assertions pass:
    │     junos_config confirm_commit: true
    │     ├── Confirming commit sent
    │     ├── Countdown cancelled
    │     └── Config permanently committed ✅
    │
    └── Any assertion fails:
          Play stops — confirm_commit never runs
          Countdown continues...
          At T+120s: Junos auto-rollback fires
          Previous config restored automatically ✅
          (No manual intervention needed)
```

---

## 20.6 — Backup, Diff, and Rollback

### Configuration Backup

```yaml
# playbooks/backup/backup_junos.yml
- name: "Backup | Junos configuration backup"
  hosts: junos_devices
  gather_facts: false
  connection: ansible.netcommon.netconf

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "Backup | Create backup directory"
      ansible.builtin.file:
        path: "backups/junos/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Backup | Capture running config in set format"
      junipernetworks.junos.junos_command:
        commands:
          - show configuration | display set
        display: text
      register: junos_set_config

    - name: "Backup | Save set-format config (easy diff and restore)"
      ansible.builtin.copy:
        content: "{{ junos_set_config.stdout[0] }}"
        dest: "backups/junos/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.set"
        mode: '0644'
      delegate_to: localhost

    - name: "Backup | Capture running config in hierarchical format"
      junipernetworks.junos.junos_command:
        commands:
          - show configuration
        display: text
      register: junos_hier_config

    - name: "Backup | Save hierarchical config (human-readable reference)"
      ansible.builtin.copy:
        content: "{{ junos_hier_config.stdout[0] }}"
        dest: "backups/junos/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.conf"
        mode: '0644'
      delegate_to: localhost
```

### Configuration Diff Between Two Backups

One of Junos' advantages is that the set-format backup files can be diffed meaningfully with standard Linux tools:

```bash
# Diff between two backup files
diff \
    backups/junos/vjunos-01/vjunos-01_20240315_020000.set \
    backups/junos/vjunos-01/vjunos-01_20240316_020000.set

# Lines starting with '<' were removed, lines starting with '>' were added
# Immediately clear which set commands changed
```

### Manual Rollback Using Junos Rollback Numbers

Junos keeps the last 50 committed configurations internally (rollback 0 = current, rollback 1 = previous, etc.):

```yaml
- name: "Rollback | Roll back to previous Junos configuration"
  junipernetworks.junos.junos_config:
    rollback: 1    # 0 = current, 1 = previous, 2 = two before, etc.
    # This loads rollback 1 into candidate and commits it
```

```yaml
- name: "Rollback | Roll back to specific rollback number"
  junipernetworks.junos.junos_config:
    rollback: "{{ rollback_number | default(1) }}"
  # Run with: ansible-playbook rollback_junos.yml -e "rollback_number=3"
```

---

## 20.7 — Key Module Reference

### `junos_facts` — Device Information

```yaml
junipernetworks.junos.junos_facts:
  gather_subset:
    - default      # hostname, version, model, serial
    - hardware     # memory, storage
    - interfaces   # interface state (via NETCONF)
    - routing      # routing table summary
```

Key variables: `ansible_net_hostname`, `ansible_net_version`, `ansible_net_model`, `ansible_net_serialnum`, `ansible_net_uptime`.

### `junos_command` — Operational Commands

```yaml
junipernetworks.junos.junos_command:
  commands:
    - show ospf neighbor
    - show bgp summary
    - show interfaces terse
  display: text      # text | json | xml
# Note: Junos commands use spaces, not abbreviations
# 'show ip ospf neigh' (IOS) → 'show ospf neighbor' (Junos)
# Always use full command words in Junos automation
```

```yaml
# JSON display for structured parsing
junipernetworks.junos.junos_command:
  commands:
    - show bgp summary
  display: json
register: bgp_json
# Access: bgp_json.stdout[0] | from_json
```

### `junos_config` — Configuration Management

```yaml
# Push from a rendered set-format file (primary approach)
junipernetworks.junos.junos_config:
  src: rendered/junos/vjunos-01/vjunos-01.set
  src_format: set      # set | text | xml
  update: merge        # merge | override | replace
  confirm: 120         # confirmed-commit timeout (0 = no timer)
  commit: true         # true = commit, false = load only (diff mode)
  comment: "Deployed by Ansible"

# Inline set commands (for small targeted changes)
junipernetworks.junos.junos_config:
  lines:
    - "set interfaces ge-0/0/0 description 'Updated by Ansible'"
    - "set system host-name vjunos-01"

# Send the confirming commit
junipernetworks.junos.junos_config:
  confirm_commit: true

# Rollback to a previous configuration
junipernetworks.junos.junos_config:
  rollback: 1
```

**`update` parameter values:**

| Value | Behavior |
|---|---|
| `merge` | Add/update specified config. Existing unspecified config is untouched. |
| `override` | Replace the entire configuration with the provided config. |
| `replace` | Replace only the config hierarchy specified (marked with `replace:` tags in text format). |

For the `src:` approach with set-format files, `merge` is correct for incremental updates. `override` is for full config replacement (use with extreme caution — equivalent to `replace: config` on IOS).

### `junos_interfaces` — Interface Resource Module

```yaml
junipernetworks.junos.junos_interfaces:
  config:
    - name: ge-0/0/0
      description: "WAN | To provider"
      enabled: true
      mtu: 1500
  state: merged
# Less commonly used than junos_config src: for Junos
# junos_interfaces exists but the set-format template approach
# covers interfaces more naturally and completely
```

### `junos_vlans` — VLAN Configuration

```yaml
junipernetworks.junos.junos_vlans:
  config:
    - name: SERVERS
      vlan_id: 10
  state: merged
# Applicable when vjunos is acting as a switch (EX-series emulation)
# For vJunos router, VLANs are typically in routing instances
```

---

## 20.8 — IOS/NX-OS vs Junos: Key Differences

| Topic | Junos | IOS / NX-OS |
|---|---|---|
| Connection protocol | NETCONF (port 830) | SSH CLI (port 22) |
| Config model | Candidate → commit (atomic) | Immediate effect |
| Rollback | Built-in (50 rollback levels) | External backup files |
| Confirmed-commit | Native safety feature | Not available |
| Config format | Hierarchical / set / XML | Flat CLI lines |
| VRFs | Routing instances | VRF definition / vrf context |
| ACLs | Firewall filters with terms | Access-lists |
| Route maps | Policy statements with terms | Route-maps |
| Interface units | Physical.Unit (ge-0/0/0.0) | Subinterfaces (Gi1.0) |
| OSPF interface | `area 0.0.0.0 interface ge-0/0/0.0` | `ip ospf 1 area 0` on interface |
| BGP groups | Peer groups with type (internal/external) | neighbor + address-family |
| Save config | Automatic on commit | `write memory` / `copy run start` |
| Loopback | `lo0` (always `lo0`) | `Loopback0`, `Loopback1`, etc. |

---

## 20.9 — Common Junos Automation Gotchas

### ### 🪲 Gotcha — NETCONF must be enabled on the device before Ansible can connect

```bash
# Manually enable NETCONF on a fresh vJunos device (one-time bootstrap)
ssh admin@172.16.0.41
# On device:
# set system services netconf ssh
# commit
# exit
```

The Ansible playbook assumes NETCONF is already enabled. Without it, every task fails with `connection refused on port 830`. The bootstrap is a manual one-time step — or done via a separate `ansible_connection: network_cli` play that runs first.

### ### 🪲 Gotcha — `update: override` replaces the entire config

```yaml
# ❌ This REPLACES THE ENTIRE DEVICE CONFIG with only what's in the file
junipernetworks.junos.junos_config:
  src: my_partial_config.set
  src_format: set
  update: override    # ← Destroys everything not in my_partial_config.set

# ✅ Use merge for incremental updates
junipernetworks.junos.junos_config:
  src: my_partial_config.set
  src_format: set
  update: merge       # ← Adds/updates, leaves other config untouched
```

### ### 🪲 Gotcha — Confirmed-commit plays must complete before the timer expires

If the confirmed-commit timeout is 120 seconds and the verification play takes 150 seconds (slow BGP convergence, many devices), the rollback fires before the confirming commit is sent:

```yaml
# Increase the timer for plays with slow convergence
confirm: 300    # 5 minutes for BGP to converge across many devices

# Or: verify BGP state with retries and longer intervals
junipernetworks.junos.junos_command:
  commands:
    - show bgp neighbor
  wait_for:
    - result[0] contains Established
  retries: 20    # Up to 100 seconds (20 × 5s) — still within 300s timer
  interval: 5
```

### ### 🪲 Gotcha — Junos set-format passwords are plain-text in the rendered file

The rendered `.set` file contains plain-text passwords from the data model:

```
set system login user ansible authentication plain-text-password-value MySecretPassword
```

This file should never be committed to Git. Add the rendered directory to `.gitignore`:

```bash
echo "rendered/" >> ~/projects/ansible-network/.gitignore
```

In production, use encrypted secrets from vault and restrict file permissions:

```bash
# In the template task
ansible.builtin.template:
  src: templates/junos/vjunos_config.j2
  dest: "rendered/junos/{{ inventory_hostname }}/{{ inventory_hostname }}.set"
  mode: '0600'    # Owner read/write only — not group or world readable
```

### ### 🪲 Gotcha — `junos_command` display format must match the parsing approach

```yaml
# ❌ Trying to parse text output as structured data
junipernetworks.junos.junos_command:
  commands: [show bgp summary]
  display: text
register: bgp_output
# bgp_output.stdout[0] is a string — can't access bgp_output.stdout[0]['peer-count']

# ✅ Use display: json for structured access
junipernetworks.junos.junos_command:
  commands: [show bgp summary]
  display: json
register: bgp_json
# bgp_json.stdout[0] is a JSON string — parse with | from_json filter
# Access: (bgp_json.stdout[0] | from_json)['bgp-information'][0]['peer-count'][0]['data']
# Junos JSON paths can be deeply nested — always test with -vvv first
```


---

Junos automation is now complete — the candidate config model, the hierarchy-aware set-format template, the confirmed-commit safety workflow with automatic rollback, and all major configuration domains. Part 21 moves to Palo Alto PAN-OS — a different automation model again, this time using the REST API and the paloaltonetworks.panos collection for firewall policy management.

