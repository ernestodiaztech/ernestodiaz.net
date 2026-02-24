---
draft: true
title: '11 - Jinja2'
weight: 11
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.




## Part 11: Variables, Data Models & Jinja2 Templating

> *Part 10 showed how to push a hostname change and loop over NTP servers. Those are simple one-value substitutions. Real network automation — generating full device configurations from structured data — requires a proper data model and Jinja2 templates that render that data into config text. This part builds both: a clean data model for the lab topology, and a growing Jinja2 template that turns that model into actual device configuration. This is the foundation that makes automation scale.*

---

## 11.1 — What a Data Model Is in Network Automation

A data model is a structured representation of network state — written in YAML, stored in inventory variables, and consumed by templates and playbooks to generate configuration.

Without a data model, network automation looks like this:

```yaml
# ❌ No data model — hardcoded values everywhere
- name: Configure interface
  cisco.ios.ios_config:
    lines:
      - "interface GigabitEthernet1"
      - " description WAN | To FW-01 eth1"
      - " ip address 10.10.10.1 255.255.255.252"
      - " no shutdown"
```

This works for one device. It breaks when I need to run it against 20 devices — each with different IPs and descriptions. And when the IP addressing scheme changes, I'm editing playbooks instead of data files.

With a data model:

```yaml
# ✅ Data model in host_vars/wan-r1.yml
interfaces:
  GigabitEthernet1:
    description: "WAN | To FW-01 eth1"
    ip: 10.10.10.1
    mask: 255.255.255.252
    state: up
```

```yaml
# ✅ Playbook uses the data model — no hardcoded values
- name: Configure interfaces from data model
  cisco.ios.ios_l3_interfaces:
    config:
      - name: "{{ item.key }}"
        ipv4:
          - address: "{{ item.value.ip }}/{{ item.value.mask | ansible.utils.ipaddr('prefix') }}"
    state: merged
  loop: "{{ interfaces | dict2items }}"
```

The playbook never changes. Only the data model changes when the network changes.

### The Three Properties of a Good Data Model

**1. Separation of data from logic**
Data lives in inventory (`host_vars/`, `group_vars/`). Logic lives in playbooks and templates. I never hardcode a device IP or VLAN ID in a playbook.

**2. Consistency across devices**
Every device of the same type uses the same variable structure. `wan-r1` and `wan-r2` both have an `interfaces:` dict with the same keys. Templates can loop over any IOS router and produce correct output because the structure is predictable.

**3. The inventory is the source of truth**
If a device's IP changes, I update `host_vars/wan-r1.yml`. The next playbook run generates the correct configuration automatically. I don't hunt through playbooks looking for hardcoded values.

### ### 🏢 Real-World Scenario

> The difference between a network automation project that scales and one that doesn't usually comes down to whether a data model exists. A project I worked with had 40 playbooks for 40 different devices — each playbook with the device's IP, VLANs, and routing config hardcoded directly in the task arguments. Adding a new device meant copying a playbook and editing every hardcoded value. One network re-IP project meant editing 40 playbooks. With a proper data model, it would have meant editing 40 `host_vars` files — same number of files, but at least those files contain only data, making the changes mechanical and auditable rather than requiring engineering judgment to avoid breaking the logic.

---

## 11.2 — Building the Lab Data Model

The lab topology from Part 6 already has a partial data model in `host_vars/` from Part 7. Now I complete it — making the data model comprehensive enough to drive Jinja2 templates.

### The Principle: One Struct Per Configuration Domain

I organize each device's `host_vars` file into clearly named dictionaries — one per configuration domain. Each dictionary has a consistent structure across all devices of the same type.

```
host_vars/<device>.yml
├── device identity (hostname, role, location)
├── interfaces (dict keyed by interface name)
├── loopback interfaces
├── routing (ospf, bgp)
├── vlans (for switches)
├── services (NTP, syslog, SNMP — usually in group_vars)
└── acls (for edge devices)
```

### Completing `host_vars/wan-r1.yml`

I update the existing file to use a complete, consistent data model:

```bash
cat > ~/projects/ansible-network/inventory/host_vars/wan-r1.yml << 'EOF'
---
# =============================================================
# wan-r1 — Cisco IOS-XE WAN Router 1
# Data model: interfaces, loopbacks, routing, services
# =============================================================

# --- Device Identity ---
device_hostname: wan-r1
device_role: wan_router
device_vendor: cisco
device_platform: ios-xe
device_location: HQ-DC-Rack-A1
device_serial: LAB-SERIAL-001

# --- Management ---
ansible_host: 172.16.0.11

# --- Loopback Interfaces ---
loopbacks:
  Loopback0:
    description: "Loopback0 | Router-ID and management reachability"
    ip: 10.255.0.1
    prefix: 32

# --- Data Plane Interfaces ---
interfaces:
  GigabitEthernet1:
    description: "WAN | To FW-01 eth1"
    ip: 10.10.10.1
    prefix: 30
    state: up
    shutdown: false
  GigabitEthernet2:
    description: "WAN | To WAN-R2 inter-router link"
    ip: 10.10.20.1
    prefix: 30
    state: up
    shutdown: false

# --- OSPF ---
ospf:
  process_id: 1
  router_id: 10.255.0.1
  areas:
    - area_id: 0
      interfaces:
        - name: GigabitEthernet1
          type: point-to-point
        - name: GigabitEthernet2
          type: point-to-point
        - name: Loopback0
          passive: true

# --- BGP ---
bgp:
  as_number: 65100
  router_id: 10.255.0.1
  neighbors:
    - ip: 10.10.10.2
      remote_as: 65200
      description: "eBGP to FW-01 (trust side)"
      update_source: ~     # null — no update-source needed
  address_families:
    - afi: ipv4
      networks:
        - prefix: 10.255.0.1
          mask: 255.255.255.255

# --- ACLs ---
acls:
  - name: MGMT_ACCESS
    type: standard
    entries:
      - sequence: 10
        action: permit
        source: 172.16.0.0
        wildcard: 0.0.0.255
      - sequence: 20
        action: deny
        source: any
        log: true

# --- Device-specific service overrides ---
# (Global NTP/syslog/SNMP are in group_vars/all.yml)
ntp_source_interface: Loopback0
snmp_location: "HQ-DC-Rack-A1-U12"
snmp_contact: "netops@lab.local"
EOF
```

### Completing `host_vars/spine-01.yml`

```bash
cat > ~/projects/ansible-network/inventory/host_vars/spine-01.yml << 'EOF'
---
# =============================================================
# spine-01 — Cisco NX-OS Spine Switch 1
# Data model: interfaces, loopbacks, routing, vpc
# =============================================================

device_hostname: spine-01
device_role: spine
device_vendor: cisco
device_platform: nxos
device_location: HQ-DC-Rack-B1

ansible_host: 172.16.0.21

# --- Loopback Interfaces ---
loopbacks:
  Loopback0:
    description: "Loopback0 | Router-ID and BGP next-hop"
    ip: 10.255.1.1
    prefix: 32
  Loopback1:
    description: "Loopback1 | VTEP NVE source interface"
    ip: 10.255.2.1
    prefix: 32

# --- Fabric Interfaces (routed, no switchport) ---
interfaces:
  Ethernet1/1:
    description: "Fabric | To FW-01 eth3 (uplink)"
    ip: 10.20.0.2
    prefix: 30
    mtu: 9216
    shutdown: false
  Ethernet1/2:
    description: "Fabric | To SPINE-02 (inter-spine link)"
    ip: 10.20.1.1
    prefix: 30
    mtu: 9216
    shutdown: false
  Ethernet1/3:
    description: "Fabric | To LEAF-01"
    ip: 10.20.2.1
    prefix: 30
    mtu: 9216
    shutdown: false
  Ethernet1/4:
    description: "Fabric | To LEAF-02"
    ip: 10.20.3.1
    prefix: 30
    mtu: 9216
    shutdown: false

# --- BGP (eBGP spine-leaf) ---
bgp:
  as_number: 65000
  router_id: 10.255.1.1
  neighbors:
    - ip: 10.20.0.1
      remote_as: 65200
      description: "eBGP to FW-01"
    - ip: 10.20.2.2
      remote_as: 65001
      description: "eBGP to LEAF-01"
    - ip: 10.20.3.2
      remote_as: 65002
      description: "eBGP to LEAF-02"
  address_families:
    - afi: ipv4
      networks:
        - prefix: 10.255.1.1
          mask: 255.255.255.255
        - prefix: 10.255.2.1
          mask: 255.255.255.255

# --- VPC Domain (NX-OS only) ---
vpc:
  domain_id: 1
  peer_keepalive_ip: 172.16.0.22    # spine-02 mgmt IP
  peer_keepalive_vrf: management

snmp_location: "HQ-DC-Rack-B1-U20"
snmp_contact: "netops@lab.local"
EOF
```

### Completing `host_vars/leaf-01.yml`

```bash
cat > ~/projects/ansible-network/inventory/host_vars/leaf-01.yml << 'EOF'
---
# =============================================================
# leaf-01 — Cisco NX-OS Leaf Switch 1
# Data model: interfaces, routing, vlans, vpc
# =============================================================

device_hostname: leaf-01
device_role: leaf
device_vendor: cisco
device_platform: nxos
device_location: HQ-DC-Rack-C1

ansible_host: 172.16.0.23

# --- Loopback Interfaces ---
loopbacks:
  Loopback0:
    description: "Loopback0 | Router-ID"
    ip: 10.255.1.3
    prefix: 32

# --- Fabric Uplinks (routed) ---
interfaces:
  Ethernet1/1:
    description: "Fabric | To SPINE-01"
    ip: 10.20.2.2
    prefix: 30
    mtu: 9216
    shutdown: false
  Ethernet1/2:
    description: "Fabric | To SPINE-02"
    ip: 10.20.4.2
    prefix: 30
    mtu: 9216
    shutdown: false
  Ethernet1/3:
    description: "Access | To HOST-01 (VLAN 20)"
    mode: access
    access_vlan: 20
    shutdown: false

# --- BGP ---
bgp:
  as_number: 65001
  router_id: 10.255.1.3
  neighbors:
    - ip: 10.20.2.1
      remote_as: 65000
      description: "eBGP to SPINE-01"
    - ip: 10.20.4.1
      remote_as: 65000
      description: "eBGP to SPINE-02"
  address_families:
    - afi: ipv4
      networks:
        - prefix: 10.255.1.3
          mask: 255.255.255.255

# --- VLANs (leaf-specific additions to base_vlans from group_vars) ---
local_vlans:
  - id: 20
    name: APP_SERVERS
    svi_ip: 192.168.20.1
    svi_prefix: 24
  - id: 30
    name: DB_SERVERS
    svi_ip: 192.168.30.1
    svi_prefix: 24

# --- VPC ---
vpc:
  domain_id: 10
  peer_keepalive_ip: 172.16.0.24    # leaf-02

snmp_location: "HQ-DC-Rack-C1-U18"
snmp_contact: "netops@lab.local"
EOF
```

### What Makes This a Good Data Model

Looking at these three files, the data model has consistent properties:

- Every device has `interfaces:` as a dict keyed by interface name
- Every device has `bgp:` with `as_number`, `router_id`, `neighbors` list
- Every neighbor in `bgp.neighbors` has `ip`, `remote_as`, `description`
- Loopbacks are in `loopbacks:`, separate from data plane `interfaces:`

This consistency means a single Jinja2 template can generate correct configuration for any of these devices by iterating the same structure.

---

## 11.3 — Variable Precedence: The Levels That Actually Matter

Ansible has 18 levels of variable precedence. Most of them never come up in practice. Here are the six levels I actually interact with, in order from lowest to highest priority:

```
LOWEST PRIORITY
    ↑
    1. role defaults        (roles/<role>/defaults/main.yml)
    2. group_vars/all.yml   (applies to every device)
    3. group_vars/<group>   (applies to a specific group)
    4. host_vars/<host>     (applies to a single device)
    5. vars: in a playbook  (defined in the play header)
    6. -e on the command line  (extra vars passed at runtime)
    ↑
HIGHEST PRIORITY
```

### What This Means in Practice

**Level 2 → 3 → 4 (the everyday flow):**

`group_vars/all.yml` sets the default for everyone. `group_vars/cisco_ios.yml` overrides it for IOS devices. `host_vars/wan-r1.yml` overrides it for `wan-r1` specifically.

```yaml
# group_vars/all.yml
ansible_user: ansible          # Default for all devices

# group_vars/cisco_ios.yml
ansible_become_password: "ansible123"    # IOS devices need this, Linux doesn't

# host_vars/wan-r1.yml
ansible_become_password: "r1-unique-enable-pass"   # wan-r1 has a different enable password
```

For `wan-r1`, `ansible_become_password` resolves to `r1-unique-enable-pass`.
For `wan-r2`, it resolves to `ansible123` (from `group_vars/cisco_ios.yml`).
For Linux hosts, `ansible_become_password` isn't set at all.

**Level 5 (play vars) — temporary overrides:**

```yaml
- name: Deploy BGP
  hosts: cisco_ios
  vars:
    bgp_timer_keepalive: 30    # Overrides anything in group_vars or host_vars
```

Use this sparingly. If a variable in `vars:` overrides a `host_vars` value, it applies identically to every host in the play — which defeats the purpose of per-device `host_vars`. Play vars are for truly play-specific constants, not per-device values.

**Level 6 (`-e`) — runtime overrides:**

```bash
# Override bgp_as for this specific run
ansible-playbook deploy_bgp.yml -e "bgp_as=65999"

# Override for a specific device (use with caution in production)
ansible-playbook deploy_base.yml -e "device_hostname=temp-name" -l wan-r1
```

`-e` is the highest priority and overrides everything — including `host_vars`. I use it for:
- Testing a change before committing it to `host_vars`
- Emergency overrides during an incident
- CI/CD pipelines that inject environment-specific variables at runtime

### The One Rule That Prevents 90% of Precedence Bugs

> **Data that is device-specific always goes in `host_vars/`. Data that is group-specific always goes in `group_vars/<group>/`. Data that is truly global goes in `group_vars/all.yml`. Never set the same variable in multiple places at the same level.**

When I follow this rule, precedence conflicts don't arise. The only time I need to think hard about precedence is when I'm intentionally overriding something at a higher level — and I should comment the code when I do.

### The Full 18 Levels (Reference)

For completeness — the full order from lowest to highest:

```
 1. Role defaults (roles/<role>/defaults/main.yml)
 2. Inventory file or dynamic inventory group vars
 3. Inventory group_vars/all
 4. Playbook group_vars/all
 5. Inventory group_vars/*
 6. Playbook group_vars/*
 7. Inventory host_vars/*
 8. Playbook host_vars/*
 9. Host facts (ansible_net_*)
10. Play vars
11. Play vars_prompt
12. Play vars_files
13. Role vars (roles/<role>/vars/main.yml)
14. Block vars
15. Task vars
16. include_vars
17. set_facts / registered vars
18. Role params / include_role params
19. Extra vars (-e on command line)   ← Always wins
```

In practice I only interact with levels 3–8 (group_vars, host_vars), level 10 (play vars), and level 19 (-e).

### ### ℹ️ Info

> Levels 7 and 8 both say `host_vars/*` — one is inventory host_vars and one is playbook host_vars. If I have both `inventory/host_vars/wan-r1.yml` AND `host_vars/wan-r1.yml` at the project root, the playbook-level one wins. This is almost never intentional. I keep all `host_vars` inside `inventory/` and never create a separate `host_vars/` at the project root, to avoid this confusion entirely.

---

## 11.4 — Introduction to Jinja2 Templating

Jinja2 is the templating engine Ansible uses to substitute variables into strings, evaluate conditions, and loop over data. I've already used Jinja2 throughout this guide — every `{{ variable }}` expression is Jinja2.

### The Three Jinja2 Constructs

```jinja2
{{ variable }}          ← Expression — outputs the value of a variable
{% if condition %}      ← Statement — control flow (if, for, set)
{# comment #}          ← Comment — not included in rendered output
```

### Jinja2 in Ansible — Two Contexts

**In playbook YAML files** — inline substitution within a string:
```yaml
- name: "Configure {{ device_hostname }}"
  cisco.ios.ios_hostname:
    config:
      hostname: "{{ device_hostname }}"
```

**In `.j2` template files** — full Jinja2 with control flow:
```jinja2
hostname {{ device_hostname }}
!
{# Configure each interface from the data model #}
{% for iface_name, iface in interfaces.items() %}
interface {{ iface_name }}
 description {{ iface.description }}
{% if iface.ip is defined %}
 ip address {{ iface.ip }} {{ iface.prefix | ansible.utils.ipaddr('netmask') }}
{% endif %}
 {% if iface.shutdown %}shutdown{% else %}no shutdown{% endif %}
!
{% endfor %}
```

`.j2` template files have no YAML syntax constraints — they're pure text with Jinja2 control flow. This is where I write the actual configuration text.

### The `template` Module

The `template` module reads a `.j2` file, substitutes variables from the current play's context, and writes the rendered output — either to a file on the control node or directly to a device via `ios_config src:`:

```yaml
# Render template to a file on the control node
- name: "Template | Render IOS config to file"
  ansible.builtin.template:
    src: templates/ios/device_config.j2
    dest: "/tmp/rendered/{{ inventory_hostname }}_config.txt"
  delegate_to: localhost

# Render and push directly to the device
- name: "Template | Push rendered config to device"
  cisco.ios.ios_config:
    src: "templates/ios/device_config.j2"   # ios_config can take a .j2 file directly
```

---

## 11.5 — Building the IOS-XE Device Config Template

I'll build a single growing template that covers interfaces, routing, and BGP. I start with the skeleton and add sections one at a time.

```bash
mkdir -p ~/projects/ansible-network/templates/ios
nano ~/projects/ansible-network/templates/ios/device_config.j2
```

### Version 1 — Hostname and Basic Settings

```jinja2
{# =============================================================
   templates/ios/device_config.j2
   IOS-XE device configuration template
   Variables sourced from host_vars/<device>.yml and group_vars/
   ============================================================= #}

{# ── Section 1: Basic device identity ─────────────────────── #}
hostname {{ device_hostname }}
!
ip domain-name {{ domain_name }}
!
{# Configure DNS servers from group_vars/all.yml (ntp_servers list) #}
{% for dns in dns_servers %}
ip name-server {{ dns }}
{% endfor %}
!
```

The `{% for %}` loop generates one `ip name-server` line per entry in `dns_servers`. With `dns_servers: [8.8.8.8, 8.8.4.4]` from `group_vars/all.yml`, this renders as:

```
ip name-server 8.8.8.8
ip name-server 8.8.4.4
```

### Version 2 — Add NTP Configuration

```jinja2
{# ── Section 2: NTP ────────────────────────────────────────── #}
{% for ntp in ntp_servers %}
ntp server {{ ntp }}{% if loop.first %} prefer{% endif %}

{% endfor %}
{% if ntp_source_interface is defined %}
ntp source {{ ntp_source_interface }}
{% endif %}
!
```

**Jinja2 constructs used here:**
- `loop.first` — boolean, true only on the first iteration. Used to add `prefer` to the first NTP server.
- `is defined` — tests whether a variable exists before trying to use it. Prevents errors when the variable isn't set for every device.

Rendered output for `wan-r1`:
```
ntp server 216.239.35.0 prefer
ntp server 216.239.35.4
ntp source Loopback0
```

### Version 3 — Add Loopback Interfaces

```jinja2
{# ── Section 3: Loopback Interfaces ───────────────────────── #}
{% for iface_name, iface in loopbacks.items() %}
interface {{ iface_name }}
 description {{ iface.description }}
 ip address {{ iface.ip }} {{ iface.prefix | ansible.utils.ipaddr('netmask') }}
 no shutdown
!
{% endfor %}
```

**Constructs used:**
- `loopbacks.items()` — iterates over the `loopbacks` dict, yielding `(key, value)` pairs. `iface_name` gets the key (`Loopback0`), `iface` gets the value dict.
- `ansible.utils.ipaddr('netmask')` — converts a prefix length to a dotted-decimal subnet mask. `32 | ansible.utils.ipaddr('netmask')` → `255.255.255.255`.

Rendered output for `wan-r1`:
```
interface Loopback0
 description Loopback0 | Router-ID and management reachability
 ip address 10.255.0.1 255.255.255.255
 no shutdown
```

### Version 4 — Add Data Plane Interfaces

```jinja2
{# ── Section 4: Data Plane Interfaces ─────────────────────── #}
{% for iface_name, iface in interfaces.items() %}
interface {{ iface_name }}
 description {{ iface.description | default('No description') }}
{% if iface.ip is defined %}
 ip address {{ iface.ip }} {{ iface.prefix | ansible.utils.ipaddr('netmask') }}
{% endif %}
 ip mtu {{ iface.mtu | default(1500) }}
{% if iface.shutdown | default(false) %}
 shutdown
{% else %}
 no shutdown
{% endif %}
!
{% endfor %}
```

**Constructs used:**
- `| default('No description')` — if `iface.description` is undefined, use the fallback value instead of raising an error.
- `| default(1500)` — default MTU of 1500 if not specified per-interface.
- `iface.shutdown | default(false)` — if `shutdown` key is missing from the interface dict, treat it as false (interface is up).

Rendered output for `wan-r1`:
```
interface GigabitEthernet1
 description WAN | To FW-01 eth1
 ip address 10.10.10.1 255.255.255.252
 ip mtu 1500
 no shutdown
!
interface GigabitEthernet2
 description WAN | To WAN-R2 inter-router link
 ip address 10.10.20.1 255.255.255.252
 ip mtu 1500
 no shutdown
!
```

### Version 5 — Add OSPF Configuration

```jinja2
{# ── Section 5: OSPF ───────────────────────────────────────── #}
{% if ospf is defined %}
router ospf {{ ospf.process_id }}
 router-id {{ ospf.router_id }}
{% for area in ospf.areas %}
{% for iface in area.interfaces %}
{% if iface.passive | default(false) %}
 passive-interface {{ iface.name }}
{% endif %}
{% endfor %}
{% endfor %}
!
{% for area in ospf.areas %}
{% for iface in area.interfaces %}
interface {{ iface.name }}
 ip ospf {{ ospf.process_id }} area {{ area.area_id }}
{% if iface.type is defined %}
 ip ospf network {{ iface.type }}
{% endif %}
!
{% endfor %}
{% endfor %}
{% endif %}
```

**Constructs used:**
- Nested `{% for %}` loops — outer loop over areas, inner loop over interfaces within each area.
- `{% if ospf is defined %}` wrapping the entire section — devices without OSPF (like Linux hosts or the firewall) skip this entire block.

Rendered output for `wan-r1`:
```
router ospf 1
 router-id 10.255.0.1
 passive-interface Loopback0
!
interface GigabitEthernet1
 ip ospf 1 area 0
 ip ospf network point-to-point
!
interface GigabitEthernet2
 ip ospf 1 area 0
 ip ospf network point-to-point
!
interface Loopback0
 ip ospf 1 area 0
```

### Version 6 — Add BGP Configuration

```jinja2
{# ── Section 6: BGP ────────────────────────────────────────── #}
{% if bgp is defined %}
router bgp {{ bgp.as_number }}
 bgp router-id {{ bgp.router_id }}
 bgp log-neighbor-changes
!
{# Configure each BGP neighbor #}
{% for neighbor in bgp.neighbors %}
 neighbor {{ neighbor.ip }} remote-as {{ neighbor.remote_as }}
 neighbor {{ neighbor.ip }} description {{ neighbor.description }}
{% if neighbor.update_source is defined and neighbor.update_source != None %}
 neighbor {{ neighbor.ip }} update-source {{ neighbor.update_source }}
{% endif %}
{% endfor %}
!
{# Configure address families #}
{% for af in bgp.address_families %}
 address-family {{ af.afi }} unicast
{% for neighbor in bgp.neighbors %}
  neighbor {{ neighbor.ip }} activate
{% endfor %}
{% if af.networks is defined %}
{% for network in af.networks %}
  network {{ network.prefix }} mask {{ network.mask }}
{% endfor %}
{% endif %}
 exit-address-family
!
{% endfor %}
{% endif %}
```

Rendered output for `wan-r1`:
```
router bgp 65100
 bgp router-id 10.255.0.1
 bgp log-neighbor-changes
!
 neighbor 10.10.10.2 remote-as 65200
 neighbor 10.10.10.2 description eBGP to FW-01 (trust side)
!
 address-family ipv4 unicast
  neighbor 10.10.10.2 activate
  network 10.255.0.1 mask 255.255.255.255
 exit-address-family
!
```

### Version 7 — The Complete Template

```bash
cat > ~/projects/ansible-network/templates/ios/device_config.j2 << 'TEMPLATE'
{# =============================================================
   templates/ios/device_config.j2
   Complete IOS-XE device configuration template.
   Driven entirely by host_vars/<device>.yml and group_vars/.
   Generated by Ansible — do not edit on device directly.
   ============================================================= #}

{# ── Section 1: Basic Identity ─────────────────────────────── #}
hostname {{ device_hostname }}
!
ip domain-name {{ domain_name }}
!
{% for dns in dns_servers %}
ip name-server {{ dns }}
{% endfor %}
!

{# ── Section 2: NTP ────────────────────────────────────────── #}
{% for ntp in ntp_servers %}
ntp server {{ ntp }}{% if loop.first %} prefer{% endif %}

{% endfor %}
{% if ntp_source_interface is defined %}
ntp source {{ ntp_source_interface }}
{% endif %}
!

{# ── Section 3: Loopback Interfaces ───────────────────────── #}
{% for iface_name, iface in loopbacks.items() %}
interface {{ iface_name }}
 description {{ iface.description }}
 ip address {{ iface.ip }} {{ iface.prefix | ansible.utils.ipaddr('netmask') }}
 no shutdown
!
{% endfor %}

{# ── Section 4: Data Plane Interfaces ─────────────────────── #}
{% for iface_name, iface in interfaces.items() %}
interface {{ iface_name }}
 description {{ iface.description | default('No description') }}
{% if iface.ip is defined %}
 ip address {{ iface.ip }} {{ iface.prefix | ansible.utils.ipaddr('netmask') }}
{% endif %}
 ip mtu {{ iface.mtu | default(1500) }}
{% if iface.shutdown | default(false) %}
 shutdown
{% else %}
 no shutdown
{% endif %}
!
{% endfor %}

{# ── Section 5: OSPF ───────────────────────────────────────── #}
{% if ospf is defined %}
router ospf {{ ospf.process_id }}
 router-id {{ ospf.router_id }}
{% for area in ospf.areas %}
{% for iface in area.interfaces %}
{% if iface.passive | default(false) %}
 passive-interface {{ iface.name }}
{% endif %}
{% endfor %}
{% endfor %}
!
{% for area in ospf.areas %}
{% for iface in area.interfaces %}
interface {{ iface.name }}
 ip ospf {{ ospf.process_id }} area {{ area.area_id }}
{% if iface.type is defined %}
 ip ospf network {{ iface.type }}
{% endif %}
!
{% endfor %}
{% endfor %}
{% endif %}

{# ── Section 6: BGP ────────────────────────────────────────── #}
{% if bgp is defined %}
router bgp {{ bgp.as_number }}
 bgp router-id {{ bgp.router_id }}
 bgp log-neighbor-changes
!
{% for neighbor in bgp.neighbors %}
 neighbor {{ neighbor.ip }} remote-as {{ neighbor.remote_as }}
 neighbor {{ neighbor.ip }} description {{ neighbor.description }}
{% if neighbor.update_source is defined and neighbor.update_source != None %}
 neighbor {{ neighbor.ip }} update-source {{ neighbor.update_source }}
{% endif %}
{% endfor %}
!
{% for af in bgp.address_families %}
 address-family {{ af.afi }} unicast
{% for neighbor in bgp.neighbors %}
  neighbor {{ neighbor.ip }} activate
{% endfor %}
{% if af.networks is defined %}
{% for network in af.networks %}
  network {{ network.prefix }} mask {{ network.mask }}
{% endfor %}
{% endif %}
 exit-address-family
!
{% endfor %}
{% endif %}

{# ── Section 7: SNMP ───────────────────────────────────────── #}
snmp-server community {{ snmp_community_ro }} RO
{% if snmp_location is defined %}
snmp-server location {{ snmp_location }}
{% endif %}
{% if snmp_contact is defined %}
snmp-server contact {{ snmp_contact }}
{% endif %}
!

{# ── Section 8: Logging ────────────────────────────────────── #}
logging host {{ syslog_server }}
logging trap informational
!

{# ── End of template ──────────────────────────────────────── #}
TEMPLATE
```

---

## 11.6 — Using the `template` Module to Render and Push Configs

Now I write a playbook that uses this template.

```bash
nano ~/projects/ansible-network/playbooks/deploy/deploy_ios_config.yml
```

```yaml
---
# =============================================================
# deploy_ios_config.yml
# Renders the IOS-XE device_config.j2 template per device
# and pushes it to the device.
#
# Usage:
#   Render only (no push): ansible-playbook deploy_ios_config.yml --tags render
#   Render and push:       ansible-playbook deploy_ios_config.yml
#   Single device:         ansible-playbook deploy_ios_config.yml -l wan-r1
#   Dry run (diff only):   ansible-playbook deploy_ios_config.yml --check --diff
# =============================================================

- name: "Deploy | IOS-XE device configuration from template"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  vars:
    rendered_config_dir: "/tmp/ansible-rendered/{{ inventory_hostname }}"
    rendered_config_file: "{{ rendered_config_dir }}/device_config.txt"

  tasks:

    # ── Step 1: Gather facts so template can reference ansible_net_* ──
    - name: "Pre-flight | Gather IOS device facts"
      cisco.ios.ios_facts:
        gather_subset:
          - default

    # ── Step 2: Render the template to a local file ────────────────
    - name: "Template | Create rendered config directory"
      ansible.builtin.file:
        path: "{{ rendered_config_dir }}"
        state: directory
        mode: '0755'
      delegate_to: localhost
      tags: render

    - name: "Template | Render device_config.j2 for {{ inventory_hostname }}"
      ansible.builtin.template:
        src: templates/ios/device_config.j2
        dest: "{{ rendered_config_file }}"
        mode: '0644'
      delegate_to: localhost
      tags: render

    - name: "Template | Show rendered config path"
      ansible.builtin.debug:
        msg: "Rendered config: {{ rendered_config_file }}"
      tags: render

    # ── Step 3: Show what the rendered config looks like (verbose mode) ─
    - name: "Template | Display rendered config (debug only)"
      ansible.builtin.command:
        cmd: "cat {{ rendered_config_file }}"
      register: rendered_content
      delegate_to: localhost
      changed_when: false
      tags: render

    - name: "Template | Print rendered config"
      ansible.builtin.debug:
        var: rendered_content.stdout_lines
        verbosity: 1    # Only shown with -v
      tags: render

    # ── Step 4: Push rendered config to device ──────────────────────
    - name: "Deploy | Push rendered configuration to {{ inventory_hostname }}"
      cisco.ios.ios_config:
        src: "{{ rendered_config_file }}"    # ios_config accepts a local file path
      register: push_result
      tags: push

    - name: "Deploy | Save configuration if changes were made"
      cisco.ios.ios_command:
        commands:
          - write memory
      when: push_result.changed
      tags: push

    - name: "Deploy | Report push result"
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} — changed: {{ push_result.changed }}"
      tags: push
```

### Rendering Without Pushing (Safe Preview)

```bash
# Render templates to files but don't push to devices
ansible-playbook playbooks/deploy/deploy_ios_config.yml --tags render

# Look at the rendered output for wan-r1
cat /tmp/ansible-rendered/wan-r1/device_config.txt
```

This is the workflow I follow before any production change — render first, review the output, confirm it looks correct, then push.

---

## 11.7 — Jinja2 Filters for Network Automation

Jinja2 filters transform variable values — they're applied with the `|` pipe operator. I use them constantly in templates and playbooks.

### `default()` — Fallback Values

```jinja2
{# Use 'No description' if iface.description is undefined or empty #}
{{ iface.description | default('No description') }}

{# Default boolean — treat missing 'shutdown' key as false #}
{{ iface.shutdown | default(false) }}

{# Default integer — MTU of 1500 if not specified #}
{{ iface.mtu | default(1500) }}

{# default(omit) — omit the parameter entirely if the variable is undefined #}
{# Useful in Ansible module calls, not templates #}
ip: "{{ iface.ip | default(omit) }}"
```

### `join()` — Combine List Items into a String

```jinja2
{# Join DNS servers into a space-separated string #}
ip name-server {{ dns_servers | join(' ') }}
{# Renders: ip name-server 8.8.8.8 8.8.4.4 #}

{# Join with a custom separator #}
{{ ntp_servers | join(', ') }}
{# Renders: 216.239.35.0, 216.239.35.4 #}

{# Join interface names for a passive-interface list #}
{% set passive_ifaces = ospf.areas[0].interfaces | selectattr('passive', 'equalto', true) | map(attribute='name') | list %}
{% for iface in passive_ifaces %}
 passive-interface {{ iface }}
{% endfor %}
```

### `selectattr()` — Filter a List by Attribute Value

`selectattr()` filters a list of dicts, keeping only items where a specified attribute meets a condition. This is one of the most useful filters in network automation.

```jinja2
{# From bgp.neighbors, get only eBGP neighbors (remote_as != local bgp.as_number) #}
{% set ebgp_neighbors = bgp.neighbors | selectattr('remote_as', 'ne', bgp.as_number) | list %}
{% for neighbor in ebgp_neighbors %}
 neighbor {{ neighbor.ip }} remote-as {{ neighbor.remote_as }}
{% endfor %}

{# Get only interfaces that are not shut down #}
{% set active_interfaces = interfaces | dict2items | selectattr('value.shutdown', 'equalto', false) | list %}
{% for iface in active_interfaces %}
interface {{ iface.key }}
 {# ... #}
{% endfor %}

{# Get only interfaces with an IP address defined #}
{% set routed_interfaces = interfaces | dict2items | selectattr('value.ip', 'defined') | list %}
```

**`selectattr()` test operators:**

| Test | Meaning | Example |
|---|---|---|
| `equalto` | Equal to value | `selectattr('state', 'equalto', 'up')` |
| `ne` | Not equal to value | `selectattr('remote_as', 'ne', 65000)` |
| `defined` | Attribute exists | `selectattr('value.ip', 'defined')` |
| `undefined` | Attribute doesn't exist | `selectattr('description', 'undefined')` |
| `match` | Regex match | `selectattr('name', 'match', '^Eth.*')` |

### `map()` — Extract an Attribute from a List of Dicts

`map()` transforms a list by extracting a single attribute from each item:

```jinja2
{# Extract just the IP addresses from the bgp.neighbors list #}
{% set neighbor_ips = bgp.neighbors | map(attribute='ip') | list %}
{# Result: ['10.10.10.2', '10.10.20.2'] #}

{# Extract interface names from OSPF area interfaces #}
{% set ospf_ifaces = ospf.areas[0].interfaces | map(attribute='name') | list %}
{# Result: ['GigabitEthernet1', 'GigabitEthernet2', 'Loopback0'] #}

{# Use in a template — generate passive-interface lines #}
{% for iface_name in ospf.areas[0].interfaces | selectattr('passive', 'defined') | map(attribute='name') | list %}
 passive-interface {{ iface_name }}
{% endfor %}
```

### `ansible.utils.ipaddr()` — IP Address Manipulation

The `ipaddr` filter from `ansible.utils` is purpose-built for network automation. It handles subnet math, address validation, and format conversion.

```jinja2
{# Convert prefix length to subnet mask #}
{{ 24 | ansible.utils.ipaddr('netmask') }}
{# Result: 255.255.255.0 #}

{{ 30 | ansible.utils.ipaddr('netmask') }}
{# Result: 255.255.255.252 #}

{# Extract network address from a CIDR #}
{{ '10.10.10.1/30' | ansible.utils.ipaddr('network') }}
{# Result: 10.10.10.0 #}

{# Extract broadcast address #}
{{ '192.168.20.0/24' | ansible.utils.ipaddr('broadcast') }}
{# Result: 192.168.20.255 #}

{# Get first usable host in a subnet #}
{{ '10.20.2.0/30' | ansible.utils.ipaddr('1') }}
{# Result: 10.20.2.1 #}

{# Get last usable host #}
{{ '10.20.2.0/30' | ansible.utils.ipaddr('-2') }}
{# Result: 10.20.2.2 #}

{# Validate that a value is a valid IP address #}
{{ '10.10.10.1' | ansible.utils.ipaddr }}
{# Result: 10.10.10.1 (valid), False (invalid) #}

{# Convert IP + prefix to CIDR notation #}
{{ '10.10.10.1' | ansible.utils.ipaddr('10.10.10.0/30') }}
{# Result: 10.10.10.1/30 #}
```

### `ansible.utils.ipsubnet()` — Subnet Calculations

```jinja2
{# Generate the 3rd /30 subnet from a /24 address space #}
{{ '10.20.0.0/24' | ansible.utils.ipsubnet(30, 2) }}
{# Result: 10.20.0.8/30 #}

{# Check if an IP is within a subnet #}
{{ '10.10.10.1' | ansible.utils.ipaddr('10.10.10.0/30') }}
{# Result: 10.10.10.1/30 (truthy if in subnet, False if not) #}
```

### Chaining Filters Together

Filters can be chained — the output of one becomes the input of the next:

```jinja2
{# Get the subnet mask for a prefix stored as an integer #}
{{ iface.prefix | ansible.utils.ipaddr('netmask') }}
{# Steps: prefix (30) → ipaddr('netmask') → '255.255.255.252' #}

{# Get all active BGP neighbor IPs as a comma-separated string #}
{{ bgp.neighbors | selectattr('remote_as', 'ne', bgp.as_number) | map(attribute='ip') | list | join(', ') }}
{# Steps: neighbors list
      → selectattr (filter to eBGP only)
      → map (extract IP addresses)
      → list (convert generator to list)
      → join (combine into string) #}

{# Render a network statement with mask from CIDR #}
{% set loopback_cidr = loopbacks.Loopback0.ip + '/' + loopbacks.Loopback0.prefix | string %}
network {{ loopback_cidr | ansible.utils.ipaddr('network') }} mask {{ loopback_cidr | ansible.utils.ipaddr('netmask') }}
{# Renders: network 10.255.0.1 mask 255.255.255.255 #}
```

### ### 💡 Tip

> When building complex filter chains, I test them interactively using the `ansible` ad-hoc command with the `debug` module before putting them in a template:
> ```bash
> ansible wan-r1 -m ansible.builtin.debug \
>     -a "msg={{ bgp.neighbors | map(attribute='ip') | list | join(', ') }}"
> # Output: "10.10.10.2"
> ```
> This lets me verify the filter chain produces the expected output before embedding it in a template where failures are harder to trace.

---

## 11.8 — Building a Data-Driven Playbook

The goal of all this structure — data model, templates, filters — is to write playbooks that are completely data-driven. The playbook logic doesn't change when the network changes. Only the data in `host_vars` and `group_vars` changes.

Here's the pattern fully realized:

```yaml
---
# =============================================================
# deploy_from_data_model.yml
# Generates full device configurations from the data model
# and pushes them to all IOS-XE devices.
# To add a new device: add it to inventory + host_vars only.
# This playbook needs no changes.
# =============================================================

- name: "Deploy | Full IOS-XE configuration from data model"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  pre_tasks:
    - name: "Pre-flight | Gather facts before render"
      cisco.ios.ios_facts:
        gather_subset: default

    - name: "Pre-flight | Verify required data model variables exist"
      ansible.builtin.assert:
        that:
          - device_hostname is defined
          - interfaces is defined
          - bgp is defined
        fail_msg: >
          Required data model variables missing for {{ inventory_hostname }}.
          Ensure host_vars/{{ inventory_hostname }}.yml defines:
          device_hostname, interfaces, bgp
        success_msg: "Data model verified for {{ inventory_hostname }}"

  tasks:
    - name: "Template | Create render directory"
      ansible.builtin.file:
        path: "/tmp/ansible-rendered/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Template | Render device configuration"
      ansible.builtin.template:
        src: templates/ios/device_config.j2
        dest: "/tmp/ansible-rendered/{{ inventory_hostname }}/config.txt"
        mode: '0644'
      delegate_to: localhost

    - name: "Deploy | Push configuration to device"
      cisco.ios.ios_config:
        src: "/tmp/ansible-rendered/{{ inventory_hostname }}/config.txt"
      register: deploy_result

    - name: "Deploy | Save if changed"
      cisco.ios.ios_command:
        commands: [write memory]
      when: deploy_result.changed

  post_tasks:
    - name: "Verify | Confirm hostname matches data model after deploy"
      cisco.ios.ios_facts:
        gather_subset: default

    - name: "Verify | Assert hostname is correct"
      ansible.builtin.assert:
        that:
          - ansible_net_hostname == device_hostname
        fail_msg: "Hostname mismatch after deploy: device has '{{ ansible_net_hostname }}', expected '{{ device_hostname }}'"
        success_msg: "Hostname verified: {{ ansible_net_hostname }}"
```

This playbook will work correctly for `wan-r1`, `wan-r2`, and any future IOS-XE device added to the inventory. The only files that need updating when a new device is added are `inventory/hosts.yml` and `inventory/host_vars/<new-device>.yml`.

---

## 11.9 — Common Gotchas in This Section

### ### 🪲 Gotcha — Jinja2 whitespace in templates produces extra blank lines

Jinja2 control blocks (`{% %}`) leave blank lines in the rendered output. IOS generally ignores extra blank lines, but some configurations are sensitive to them. I use the `-` modifier to strip whitespace:

```jinja2
{# ❌ Leaves a blank line where the if block was #}
{% if ospf is defined %}
router ospf {{ ospf.process_id }}
{% endif %}

{# ✅ Strips the blank line with - modifier #}
{%- if ospf is defined %}
router ospf {{ ospf.process_id }}
{%- endif %}
```

Alternatively, I configure Jinja2 in `ansible.cfg`:
```ini
[defaults]
jinja2_extensions = jinja2.ext.do
# In templates, use {% ... -%} to strip trailing newline after block tags
```

### ### 🪲 Gotcha — `dict2items` is needed to loop over a dict

I can't loop directly over a dict in Jinja2 — I need `dict2items` to convert it to a list of `{key, value}` pairs:

```jinja2
{# ❌ Wrong — can't loop over a dict directly #}
{% for iface in interfaces %}

{# ✅ Correct — convert dict to list of {key, value} items #}
{% for iface_name, iface in interfaces.items() %}   {# Python-style .items() #}
{# OR #}
{% for iface in interfaces | dict2items %}          {# Jinja2 filter style #}
  {# iface.key = interface name, iface.value = interface data dict #}
{% endfor %}
```

Both `.items()` and `dict2items` work. I use `.items()` in templates because it's more readable to Python developers. `dict2items` is more common in playbook `loop:` statements.

### ### 🪲 Gotcha — `ansible.utils.ipaddr` filter not found

If I get `FilterModule object has no attribute 'ipaddr'`, the `ansible.utils` collection isn't installed:

```bash
# Verify the collection is installed
ansible-galaxy collection list | grep ansible.utils

# Install if missing
ansible-galaxy collection install ansible.utils

# Verify the filter works
ansible localhost -m ansible.builtin.debug \
    -a "msg={{ '24' | ansible.utils.ipaddr('netmask') }}"
# Should return: 255.255.255.0
```

### ### 🪲 Gotcha — Template renders correctly locally but fails on device

The most common cause is that `ios_config src:` pushes the rendered file as a config merge, but the file contains lines that conflict with IOS syntax expectations — extra blank lines, wrong indentation inside interface blocks, or missing `!` separators. I always review the rendered file before pushing:

```bash
# Render only, no push
ansible-playbook playbooks/deploy/deploy_ios_config.yml --tags render

# Review the output
cat /tmp/ansible-rendered/wan-r1/device_config.txt
```

Then manually compare it against a known-good running config to spot formatting differences.

---


The data model and templating foundation is in place. Every configuration from here on is generated from structured data — no more hardcoded values in playbooks. Part 12 addresses the one remaining gap: all those passwords and tokens in `group_vars` are still in plain text. That gets fixed with Ansible Vault.


