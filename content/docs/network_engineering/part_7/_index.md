---
draft: false
title: '7 - Inventory'
description: "Part 7 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 7
---

{{< badge "Ansible" >}}
{{< badge content="Linux" color="red" >}}

This is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. Each part will build upon the last. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Ansible Inventory

In Part 6 I created a minimal inventory just to get Ansible talking to the lab. That was enough to prove connectivity. This part is where the inventory becomes a proper, production-quality data source with well-structured groups, realistic per-device variables, platform-specific connection settings, and a preview of how dynamic inventory will eventually replace static files when Netbox is running. The inventory isn't just a list of IPs. It's the foundation every playbook is built on.

---

## What an Inventory File Is

Before Ansible can do anything, it needs to know three things about each device: where it is (IP or hostname), how to connect to it (SSH credentials, connection type), and what kind of device it is (IOS, NX-OS, PAN-OS). The inventory file is where all of this lives.

Every `ansible-playbook` command references an inventory either explicitly with `-i inventory/hosts.yml` or implicitly through the `inventory =` setting in `ansible.cfg`. Without an inventory, Ansible has nothing to act on.

The inventory also defines groups, logical collections of hosts that I can target in playbooks. Instead of writing a playbook that names every device individually, I write `hosts: cisco_ios` and Ansible runs it against every device in the `cisco_ios` group. Add a new router to the group and the playbook automatically covers it.

---

#### What Lives in the Inventory vs. What Lives in Playbooks

This is a distinction worth being deliberate about from the start:

| Belongs in Inventory | Belongs in Playbooks |
|---|---|
| Device hostnames and IPs | What tasks to run |
| Connection credentials | Which modules to use |
| Platform/OS type | Task logic and conditions |
| Device-specific variables (interfaces, VLANs) | Templates and configurations |
| Group membership | Error handling |
| Environment (lab, staging, prod) | Roles and includes |

The inventory is the 'what exists and how to reach it'. Playbooks are the 'what to do'. Keeping these concerns separate makes both easier to maintain.

---

## INI vs YAML Inventory Formats

Ansible supports multiple inventory file formats. The two most common are INI and YAML. I'll show both side by side so I understand what I'm looking at when I encounter either in the wild, then commit to YAML for this guide.

#### The Same Inventory in Both Formats

**INI Format:**

```ini
# inventory/hosts.ini

[wan]
wan-r1 ansible_host=172.16.0.11
wan-r2 ansible_host=172.16.0.12

[spine_switches]
spine-01 ansible_host=172.16.0.21
spine-02 ansible_host=172.16.0.22

[leaf_switches]
leaf-01 ansible_host=172.16.0.23
leaf-02 ansible_host=172.16.0.24

[paloalto]
fw-01 ansible_host=172.16.0.10

[linux_hosts]
host-01 ansible_host=172.16.0.31
host-02 ansible_host=172.16.0.32

# Group of groups — cisco_ios contains the wan group
[cisco_ios:children]
wan

# Group of groups — cisco_nxos contains spine and leaf groups
[cisco_nxos:children]
spine_switches
leaf_switches

# Group of groups — datacenter contains nxos switches
[datacenter:children]
cisco_nxos

# Group-level variables (can also go in group_vars/ directory)
[cisco_ios:vars]
ansible_network_os=cisco.ios.ios
ansible_connection=network_cli

[cisco_nxos:vars]
ansible_network_os=cisco.nxos.nxos
ansible_connection=network_cli

[all:vars]
ansible_user=ansible
ansible_password=ansible123
```

**YAML Format (equivalent):**

```yaml
# inventory/hosts.yml
---
all:
  children:
    cisco_ios:
      children:
        wan:
          hosts:
            wan-r1:
              ansible_host: 172.16.0.11
            wan-r2:
              ansible_host: 172.16.0.12
      vars:
        ansible_network_os: cisco.ios.ios
        ansible_connection: network_cli

    cisco_nxos:
      children:
        spine_switches:
          hosts:
            spine-01:
              ansible_host: 172.16.0.21
            spine-02:
              ansible_host: 172.16.0.22
        leaf_switches:
          hosts:
            leaf-01:
              ansible_host: 172.16.0.23
            leaf-02:
              ansible_host: 172.16.0.24
      vars:
        ansible_network_os: cisco.nxos.nxos
        ansible_connection: network_cli

    paloalto:
      hosts:
        fw-01:
          ansible_host: 172.16.0.10

    linux_hosts:
      hosts:
        host-01:
          ansible_host: 172.16.0.31
        host-02:
          ansible_host: 172.16.0.32

  vars:
    ansible_user: ansible
    ansible_password: ansible123
```

---

#### Comparison

| Feature | INI | YAML |
|---|---|---|
| Readability for small inventories |  Simpler, less indentation |  More verbose |
| Readability for large/nested inventories |  Gets messy fast |  Hierarchical structure is clear |
| Group of groups | Requires `[group:children]` syntax | Native nesting |
| Variables per host | Inline on same line | Can be inline or in `host_vars/` |
| Works with dynamic inventory |  INI plugins are deprecated |  Standard format |
| Jinja2 and complex values | Limited |  Full support |
| Recommended for | Quick one-off tests | All real projects |

---

#### My Recommendation

I use YAML exclusively for all real projects. The INI format is fine for a five-device quick test but breaks down as inventory grows. YAML's nested structure maps naturally to how I think about network hierarchy (core vs access, datacenter vs WAN, production vs lab). The extra verbosity is a worthwhile tradeoff.

The rest of this lab uses YAML inventory.

>[!Tip]
> When I find an old INI-format inventory and want to convert it to YAML, I can use `ansible-inventory` to do the conversion:
> ```bash
> ansible-inventory -i old_inventory.ini --list -y > inventory/hosts.yml
> ```
> The `-y` flag outputs in YAML format. I always review the output because automatic conversion sometimes flattens nested group structures that need to be manually reorganized.

---

## Static Inventory

Part 6's inventory was a starting point. Now I build it out properly with full group hierarchy, logical groupings, and a structure that mirrors how I'll actually target devices in playbooks.

#### The Expanded `inventory/hosts.yml`

```bash
nano ~/projects/ansible-network/inventory/hosts.yml
```

```yaml
---
# =============================================================
# Ansible Inventory — Enterprise Lab
# Containerlab Management Network: 172.16.0.0/24
#
# Group Hierarchy:
#   all
#   ├── network_devices
#   │   ├── cisco_ios        (WAN routers)
#   │   │   └── wan
#   │   ├── cisco_nxos       (DC switches)
#   │   │   ├── spine_switches
#   │   │   └── leaf_switches
#   │   └── paloalto         (Firewall)
#   └── linux_hosts          (End hosts)
#
# Logical Groups (cross-platform targeting):
#   ├── wan                  (WAN-facing devices)
#   ├── datacenter           (All DC devices)
#   └── edge                 (Firewall + WAN routers)
# =============================================================

all:
  children:

    # =========================================================
    # PLATFORM GROUPS
    # These groups define HOW to connect (connection type,
    # network OS). Group vars in group_vars/ handle credentials
    # and connection settings per platform.
    # =========================================================

    cisco_ios:
      children:
        wan:
          hosts:
            wan-r1:
              ansible_host: 172.16.0.11
            wan-r2:
              ansible_host: 172.16.0.12

    cisco_nxos:
      children:
        spine_switches:
          hosts:
            spine-01:
              ansible_host: 172.16.0.21
            spine-02:
              ansible_host: 172.16.0.22
        leaf_switches:
          hosts:
            leaf-01:
              ansible_host: 172.16.0.23
            leaf-02:
              ansible_host: 172.16.0.24

    paloalto:
      hosts:
        fw-01:
          ansible_host: 172.16.0.10

    linux_hosts:
      hosts:
        host-01:
          ansible_host: 172.16.0.31
        host-02:
          ansible_host: 172.16.0.32

    # =========================================================
    # LOGICAL GROUPS
    # These groups define WHAT role a device plays. Used for
    # targeting specific device roles regardless of platform.
    # A device can belong to both a platform group AND a
    # logical group simultaneously.
    # =========================================================

    # All devices involved in WAN connectivity
    wan_edge:
      hosts:
        wan-r1:
        wan-r2:
        fw-01:

    # All datacenter switching infrastructure
    datacenter:
      children:
        spine_switches:
        leaf_switches:

    # All network devices (excludes Linux hosts)
    network_devices:
      children:
        cisco_ios:
        cisco_nxos:
        paloalto:
```

---

#### Understanding Group Hierarchy

When a device belongs to a group, it also implicitly belongs to all **parent groups** above it. For example, `spine-01` is in:

```
spine-01
  └── spine_switches      (direct parent)
      └── cisco_nxos      (grandparent)
          └── network_devices  (great-grandparent)
              └── all          (top-level — always exists)
```

This matters for variable inheritance and for playbook targeting. If I write `hosts: cisco_nxos`, my playbook runs against `spine-01`, `spine-02`, `leaf-01`, and `leaf-02`. All devices in the hierarchy beneath `cisco_nxos`.

>[!Info]
> The `all` group always exists in Ansible, even if I don't define it explicitly. Every host in any inventory is automatically a member of `all`. This is why `group_vars/all.yml` is the place for truly global variables because they apply to every single device regardless of what other groups they belong to. I use `all` for things like NTP servers, DNS servers, and global credentials that are the same across every platform.

---

#### Targeting Devices in Playbooks

The group structure I've built gives me flexible targeting:

Target all Cisco IOS devices (wan-r1, wan-r2)

```yaml
hosts: cisco_ios
```

Target only spine switches

```yaml
hosts: spine_switches
```

Target all NX-OS devices (all 4 switches)

```yaml
hosts: cisco_nxos
```

Target all network devices (everything except Linux hosts)

```yaml
hosts: network_devices
```

Target a single device

```yaml
hosts: wan-r1
```

Target multiple specific groups

```yaml
hosts: spine_switches,leaf_switches
```

Target all devices EXCEPT Linux hosts

```yaml
hosts: all,!linux_hosts
```

Target WAN edge devices only

```yaml
hosts: wan_edge
```

>[!Tip]
> The `!` operator excludes a group. `hosts: all,!linux_hosts` means every device in `all` except those in `linux_hosts`.

---

## Host Variables

Host variables are variables specific to a single device. Ansible has a set of magic variables,special variable names it recognizes and uses to control connection behavior.

---

#### The Essential Connection Variables

| Variable | Purpose | Example Value |
|---|---|---|
| `ansible_host` | The IP or hostname Ansible connects to | `172.16.0.11` |
| `ansible_port` | SSH port (default: 22) | `22` |
| `ansible_user` | SSH username | `ansible` |
| `ansible_password` | SSH password (use Vault in production) | `ansible123` |
| `ansible_network_os` | Platform identifier for network_cli | `cisco.ios.ios` |
| `ansible_connection` | Connection plugin to use | `network_cli` |
| `ansible_become` | Enable privilege escalation | `true` |
| `ansible_become_method` | How to escalate (enable, sudo) | `enable` |
| `ansible_become_password` | Enable/sudo password | `ansible123` |
| `ansible_ssh_private_key_file` | Path to SSH private key | `~/.ssh/lab_key` |
| `ansible_ssh_common_args` | Extra SSH arguments | `-o StrictHostKeyChecking=no` |
| `ansible_httpapi_use_ssl` | Use HTTPS for httpapi connection | `true` |
| `ansible_httpapi_validate_certs` | Validate SSL certificates | `false` |
| `ansible_persistent_connect_timeout` | Timeout for persistent connection | `30` |
| `ansible_command_timeout` | Timeout per command | `30` |

#### Where to Set Variables

Host variables can be set in three places:
1. **Inline in `hosts.yml`** - on the same line/block as the host definition
2. **In `host_vars/<hostname>.yml`** - a dedicated file per device
3. **In `group_vars/<groupname>.yml`** - shared across all devices in a group

The rule I follow:

- If a variable is **the same for every device in a group** → `group_vars/`
- If a variable is **unique to a single device** → `host_vars/<hostname>.yml`
- If a variable is **truly global** (same for every device everywhere) → `group_vars/all.yml`
- **Never** put credentials inline in `hosts.yml` - they should always be in `group_vars/` or `host_vars/` where Vault can encrypt them

---

## Group Variables

The `group_vars/` directory is where I define variables that apply to an entire group. Ansible automatically reads these files when it processes the inventory, no explicit reference needed.

---

#### Directory Structure

```
inventory/
├── hosts.yml
├── group_vars/
│   ├── all.yml              ← Applies to every device
│   ├── cisco_ios.yml        ← Applies to all cisco_ios hosts
│   ├── cisco_nxos.yml       ← Applies to all cisco_nxos hosts
│   ├── paloalto.yml         ← Applies to all paloalto hosts
│   ├── spine_switches.yml   ← Applies to spine switches only
│   ├── leaf_switches.yml    ← Applies to leaf switches only
│   ├── wan.yml              ← Applies to WAN group only
│   └── linux_hosts.yml      ← Applies to Linux hosts
└── host_vars/
    ├── wan-r1.yml
    ├── wan-r2.yml
    ├── spine-01.yml
    └── ...
```

---

{{% steps %}}

#### Global Variables

```bash
nano ~/projects/ansible-network/inventory/group_vars/all.yml
```

```yaml
---
# =============================================================
# Global variables
# =============================================================

# --- Credentials ---
ansible_user: ansible
ansible_password: "ansible123"

# --- NTP Servers ---
ntp_servers:
  - 216.239.35.0
  - 216.239.35.4

# --- DNS Servers ---
dns_servers:
  - 8.8.8.8
  - 8.8.4.4

# --- Syslog Server ---
syslog_server: 172.16.0.100

# --- Domain Name ---
domain_name: lab.local

# --- SNMP Community ---
snmp_community_ro: "lab-read-only"

# --- Default SSH timeout settings ---
ansible_persistent_connect_timeout: 30
ansible_command_timeout: 30
```

---

#### IOS-XE Connection Settings

```bash
nano ~/projects/ansible-network/inventory/group_vars/cisco_ios.yml
```

```yaml
---
# =============================================================
# Cisco IOS / IOS-XE
# =============================================================

ansible_network_os: cisco.ios.ios
ansible_connection: network_cli

# Privilege escalation
ansible_become: true
ansible_become_method: enable
ansible_become_password: "ansible123"

# --- IOS-specific platform settings ---
# SSH terminal width - prevent line wrapping in show commands
ansible_terminal_length: 0

# --- IOS Feature Flags ---
# Whether this IOS device supports the newer Resource Module style
# (IOS-XE 16.x+ supports declarative resource modules)
ios_resource_modules_supported: true

# --- Backup settings ---
backup_dir: backups/cisco_ios
```

---

#### NX-OS Connection Settings

```bash
nano ~/projects/ansible-network/inventory/group_vars/cisco_nxos.yml
```

```yaml
---
# =============================================================
# Cisco NX-OS
# =============================================================

ansible_network_os: cisco.nxos.nxos
ansible_connection: network_cli

# NX-OS does not use enable mode
ansible_become: false

# --- NX-OS specific settings ---
# Terminal width setting for NX-OS
ansible_terminal_length: 0

# --- NX-OS Features to ensure are enabled ---
# These will be referenced in the NX-OS base config role
nxos_features_required:
  - bgp
  - ospf
  - interface-vlan
  - lacp
  - lldp
  - vpc
  - nxapi

# --- Backup settings ---
backup_dir: backups/cisco_nxos
```

---

#### Spine-Specific Variables

```bash
nano ~/projects/ansible-network/inventory/group_vars/spine_switches.yml
```

```yaml
---
# =============================================================
# Spine Switches — DC spine layer specific variables
# Inherits all cisco_nxos.yml variables automatically
# =============================================================

# Role identifier — used in templates and conditionals
device_role: spine

# BGP AS number for spine layer (used in BGP config templates)
bgp_as: 65000

# Spine-to-spine link subnet
spine_interconnect_subnet: "10.0.0.0/30"

# Loopback interface for BGP router-id and overlay
loopback_interface: "Loopback0"

# MTU for fabric links (jumbo frames for VXLAN)
fabric_mtu: 9216
```

---

#### Leaf-Specific Variables

```bash
nano ~/projects/ansible-network/inventory/group_vars/leaf_switches.yml
```

```yaml
---
# =============================================================
# Leaf Switches — DC leaf layer specific variables
# Inherits all cisco_nxos.yml variables automatically
# =============================================================

device_role: leaf

# BGP AS numbers for leaf layer (unique per leaf in eBGP spine-leaf)
# Individual leaf AS numbers are defined in host_vars/

# VLANs that all leaf switches should have
base_vlans:
  - id: 10
    name: MGMT
    subnet: "192.168.10.0/24"
  - id: 20
    name: APP_SERVERS
    subnet: "192.168.20.0/24"
  - id: 30
    name: DB_SERVERS
    subnet: "192.168.30.0/24"
  - id: 99
    name: NATIVE
    subnet: ""

# SVI (Layer 3 VLAN interface) settings
svi_enabled: true

# VPC domain settings (for dual-homed servers)
vpc_keepalive_vlan: 4094
```

---

#### PAN-OS Connection Settings

```bash
nano ~/projects/ansible-network/inventory/group_vars/paloalto.yml
```

```yaml
---
# =============================================================
# Palo Alto PAN-OS — Connection and platform settings
# PAN-OS uses HTTPS API, not SSH CLI
# =============================================================

ansible_network_os: paloaltonetworks.panos.panos
ansible_connection: ansible.netcommon.httpapi
ansible_httpapi_use_ssl: true
ansible_httpapi_validate_certs: false    # Lab uses self-signed cert

# PAN-OS API port
ansible_port: 443

# --- PAN-OS specific settings ---
# Device group (for Panorama-managed devices — standalone for now)
panos_device_group: "lab"

# Vsys to target
panos_vsys: "vsys1"

# Backup settings
backup_dir: backups/paloalto

# Security zone names (referenced in security policy playbooks)
zones:
  untrust: "untrust"
  trust: "trust"
  dmz: "dmz"
```

---

#### WAN Group Variables

```bash
nano ~/projects/ansible-network/inventory/group_vars/wan.yml
```

```yaml
---
# =============================================================
# WAN Group — Variables for WAN-facing routers
# Inherits cisco_ios.yml variables automatically
# =============================================================

device_role: wan_router

# BGP AS for WAN routers
bgp_as: 65100

# OSPF process ID for WAN routing
ospf_process_id: 1
ospf_area: 0

# WAN interface naming convention
wan_interface: GigabitEthernet1
```

---

#### Linux Host Connection Settings

```bash
nano ~/projects/ansible-network/inventory/group_vars/linux_hosts.yml
```

```yaml
---
# =============================================================
# Linux Hosts — Alpine Linux containers
# =============================================================

ansible_connection: ssh
ansible_python_interpreter: /usr/bin/python3

# Default shell
ansible_shell_type: sh

device_role: server
```

>[!Tip]
> When a host belongs to multiple groups (e.g., `spine-01` belongs to both `spine_switches` and `cisco_nxos`), it inherits variables from all of them. If the same variable is defined in multiple group files, Ansible resolves the conflict using 'variable precedence'' which means more specific groups win over less specific ones. `host_vars/` always wins over `group_vars/`.

{{% /steps %}}

---

## Host Variables

While `group_vars/` handles settings shared across a group, `host_vars/` is for variables that are unique to a specific device such as its loopback IP, its BGP AS number, its specific interface assignments. I build realistic `host_vars` files for one device from each platform.

---

{{% steps %}}

#### Directory Structure

```bash
mkdir -p ~/projects/ansible-network/inventory/host_vars/{wan-r1,wan-r2,fw-01,spine-01,spine-02,leaf-01,leaf-02,host-01,host-02}
```

#### IOS-XE WAN Router

```bash
nano ~/projects/ansible-network/inventory/host_vars/wan-r1.yml
```

```yaml
---
# =============================================================
# wan-r1 — Cisco IOS-XE WAN Router 1
# Management IP: 172.16.0.11
# Role: Primary WAN router, BGP to upstream ISP
# =============================================================

# --- Device Identity ---
device_hostname: wan-r1
device_role: wan_router
device_vendor: cisco
device_platform: ios-xe
device_location: HQ-DC-Rack-A1
device_serial: LAB-SERIAL-001   # Placeholder for lab

# --- Management ---
ansible_host: 172.16.0.11

# --- Loopback Interface (Router ID, BGP peer address) ---
loopback0:
  ip: 10.255.0.1
  mask: 255.255.255.255
  description: "Loopback0 | Router-ID"

# --- WAN Interface (connects to simulated ISP / FW) ---
interfaces:
  GigabitEthernet1:
    description: "WAN | To FW-01 eth1"
    ip: 10.10.10.1
    mask: 255.255.255.252
    state: up
  GigabitEthernet2:
    description: "WAN | To WAN-R2 (inter-router link)"
    ip: 10.10.20.1
    mask: 255.255.255.252
    state: up

# --- Routing ---
ospf_router_id: 10.255.0.1
ospf_area: 0
ospf_interfaces:
  - GigabitEthernet1
  - GigabitEthernet2
  - Loopback0

bgp_as: 65100
bgp_router_id: 10.255.0.1
bgp_neighbors:
  - neighbor_ip: 10.10.10.2    # fw-01 outside interface
    remote_as: 65200
    description: "eBGP to FW-01"

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

# --- NTP (overrides global if needed) ---
ntp_source_interface: Loopback0

# --- SNMP ---
snmp_location: "HQ-DC-Rack-A1-U12"
snmp_contact: "netops@lab.local"
```

#### IOS-XE WAN Router 2

```bash
nano ~/projects/ansible-network/inventory/host_vars/wan-r2.yml
```

```yaml
---
# =============================================================
# wan-r2 — Cisco IOS-XE WAN Router 2
# Management IP: 172.16.0.12
# Role: Secondary WAN router, BGP redundancy
# =============================================================

device_hostname: wan-r2
device_role: wan_router
device_vendor: cisco
device_platform: ios-xe
device_location: HQ-DC-Rack-A2
device_serial: LAB-SERIAL-002

ansible_host: 172.16.0.12

loopback0:
  ip: 10.255.0.2
  mask: 255.255.255.255
  description: "Loopback0 | Router-ID"

interfaces:
  GigabitEthernet1:
    description: "WAN | To FW-01 eth2"
    ip: 10.10.11.1
    mask: 255.255.255.252
    state: up
  GigabitEthernet2:
    description: "WAN | To WAN-R1 (inter-router link)"
    ip: 10.10.20.2
    mask: 255.255.255.252
    state: up

ospf_router_id: 10.255.0.2
ospf_area: 0
ospf_interfaces:
  - GigabitEthernet1
  - GigabitEthernet2
  - Loopback0

bgp_as: 65100
bgp_router_id: 10.255.0.2
bgp_neighbors:
  - neighbor_ip: 10.10.11.2    # fw-01 second outside interface
    remote_as: 65200
    description: "eBGP to FW-01"

snmp_location: "HQ-DC-Rack-A2-U12"
snmp_contact: "netops@lab.local"
```

#### Palo Alto PAN-OS Firewall

```bash
nano ~/projects/ansible-network/inventory/host_vars/fw-01.yml
```

```yaml
---
# =============================================================
# fw-01 — Palo Alto PAN-OS Firewall
# Management IP: 172.16.0.10
# Role: WAN edge firewall, security policy enforcement
# =============================================================

device_hostname: fw-01
device_role: firewall
device_vendor: paloalto
device_platform: pan-os
device_location: HQ-DC-Rack-A3
device_serial: LAB-SERIAL-003

ansible_host: 172.16.0.10

# --- Security Zones ---
security_zones:
  - name: untrust
    mode: layer3
    interfaces:
      - ethernet1/1    # To WAN-R1
      - ethernet1/2    # To WAN-R2
  - name: trust
    mode: layer3
    interfaces:
      - ethernet1/3    # To SPINE-01
      - ethernet1/4    # To SPINE-02
  - name: mgmt
    mode: layer3
    interfaces: []

# --- Interfaces ---
interfaces:
  ethernet1/1:
    description: "Untrust | To WAN-R1"
    ip: 10.10.10.2
    mask: 255.255.255.252
    zone: untrust
  ethernet1/2:
    description: "Untrust | To WAN-R2"
    ip: 10.10.11.2
    mask: 255.255.255.252
    zone: untrust
  ethernet1/3:
    description: "Trust | To SPINE-01"
    ip: 10.20.0.1
    mask: 255.255.255.252
    zone: trust
  ethernet1/4:
    description: "Trust | To SPINE-02"
    ip: 10.20.0.5
    mask: 255.255.255.252
    zone: trust

# --- Address Objects (used in security policies) ---
address_objects:
  - name: RFC1918-10
    type: ip-netmask
    value: 10.0.0.0/8
  - name: RFC1918-172
    type: ip-netmask
    value: 172.16.0.0/12
  - name: RFC1918-192
    type: ip-netmask
    value: 192.168.0.0/16
  - name: DC-SERVERS
    type: ip-netmask
    value: 192.168.20.0/24

# --- BGP ---
bgp_as: 65200
bgp_router_id: 10.255.0.10

snmp_location: "HQ-DC-Rack-A3-U10"
snmp_contact: "netops@lab.local"
```

#### NX-OS Spine Switch 1

```bash
nano ~/projects/ansible-network/inventory/host_vars/spine-01.yml
```

```yaml
---
# =============================================================
# spine-01 — Cisco NX-OS Spine Switch 1
# Management IP: 172.16.0.21
# Role: DC spine, BGP route reflector, VXLAN VTEP
# =============================================================

device_hostname: spine-01
device_role: spine
device_vendor: cisco
device_platform: nxos
device_location: HQ-DC-Rack-B1
device_serial: LAB-SERIAL-021

ansible_host: 172.16.0.21

# --- Loopback (Router-ID, BGP, VTEP) ---
loopback0:
  ip: 10.255.1.1
  mask: 255.255.255.255
  description: "Loopback0 | Router-ID"

loopback1:
  ip: 10.255.2.1
  mask: 255.255.255.255
  description: "Loopback1 | VTEP NVE source"

# --- Fabric Interfaces (to leaf switches) ---
interfaces:
  Ethernet1/1:
    description: "Fabric | To FW-01 eth3"
    ip: 10.20.0.2
    mask: 255.255.255.252
    mtu: 9216
    state: up
  Ethernet1/2:
    description: "Fabric | To SPINE-02 (inter-spine)"
    ip: 10.20.1.1
    mask: 255.255.255.252
    mtu: 9216
    state: up
  Ethernet1/3:
    description: "Fabric | To LEAF-01"
    ip: 10.20.2.1
    mask: 255.255.255.252
    mtu: 9216
    state: up
  Ethernet1/4:
    description: "Fabric | To LEAF-02"
    ip: 10.20.3.1
    mask: 255.255.255.252
    mtu: 9216
    state: up

# --- BGP (eBGP spine-leaf model) ---
bgp_as: 65000
bgp_router_id: 10.255.1.1
bgp_neighbors:
  - neighbor_ip: 10.20.0.1      # fw-01
    remote_as: 65200
    description: "eBGP to FW-01"
  - neighbor_ip: 10.20.2.2      # leaf-01
    remote_as: 65001
    description: "eBGP to LEAF-01"
  - neighbor_ip: 10.20.3.2      # leaf-02
    remote_as: 65002
    description: "eBGP to LEAF-02"

# --- NX-OS VPC Domain ---
vpc_domain_id: 1
vpc_peer_ip: 172.16.0.22    # spine-02 mgmt IP for keepalive

snmp_location: "HQ-DC-Rack-B1-U20"
snmp_contact: "netops@lab.local"
```

#### NX-OS Leaf Switch 1

```bash
nano ~/projects/ansible-network/inventory/host_vars/leaf-01.yml
```

```yaml
---
# =============================================================
# leaf-01 — Cisco NX-OS Leaf Switch 1
# Management IP: 172.16.0.23
# Role: DC access layer, server connectivity, VPC peer
# =============================================================

device_hostname: leaf-01
device_role: leaf
device_vendor: cisco
device_platform: nxos
device_location: HQ-DC-Rack-C1
device_serial: LAB-SERIAL-023

ansible_host: 172.16.0.23

loopback0:
  ip: 10.255.1.3
  mask: 255.255.255.255
  description: "Loopback0 | Router-ID"

# --- Fabric uplinks ---
interfaces:
  Ethernet1/1:
    description: "Fabric | To SPINE-01"
    ip: 10.20.2.2
    mask: 255.255.255.252
    mtu: 9216
    state: up
  Ethernet1/2:
    description: "Fabric | To SPINE-02"
    ip: 10.20.4.2
    mask: 255.255.255.252
    mtu: 9216
    state: up
  Ethernet1/3:
    description: "Access | To HOST-01"
    mode: access
    access_vlan: 20
    state: up

# --- BGP ---
bgp_as: 65001
bgp_router_id: 10.255.1.3
bgp_neighbors:
  - neighbor_ip: 10.20.2.1
    remote_as: 65000
    description: "eBGP to SPINE-01"
  - neighbor_ip: 10.20.4.1
    remote_as: 65000
    description: "eBGP to SPINE-02"

# --- VLANs specific to this leaf ---
local_vlans:
  - id: 20
    name: APP_SERVERS
  - id: 30
    name: DB_SERVERS

# --- VPC ---
vpc_domain_id: 10
vpc_peer_ip: 172.16.0.24    # leaf-02

snmp_location: "HQ-DC-Rack-C1-U18"
snmp_contact: "netops@lab.local"
```

>[!Tip]
> I keep `host_vars` files focused on data, not logic. They describe what a device is and what it has like IPs, interface names, VLAN assignments, BGP AS numbers. They don't contain conditional logic or task definitions. If I find myself putting `when:` conditions or complex Jinja2 in a `host_vars` file, that's a sign the logic belongs in a playbook or template instead.

{{% /steps %}}

---

## Variable Precedence Preview

Since I now have variables defined in `group_vars/all.yml`, `group_vars/cisco_ios.yml`, `group_vars/wan.yml`, and `host_vars/wan-r1.yml`, I need to understand which one wins when the same variable is defined in multiple places.

The rule is simple: **more specific always wins**.

```
host_vars/wan-r1.yml         ← Highest priority (most specific)
    ↑
group_vars/wan.yml           ← More specific group
    ↑
group_vars/cisco_ios.yml     ← Less specific group (parent)
    ↑
group_vars/all.yml           ← Lowest priority (applies to everything)
```

A practical example: `ansible_become_password` is set in `group_vars/cisco_ios.yml` as `ansible123`. If `wan-r1` has a different enable password (common in real networks where devices have unique credentials), I override it in `host_vars/wan-r1.yml`:

```yaml
# host_vars/wan-r1.yml
ansible_become_password: "unique-enable-password-for-r1"
```

This overrides the group-level setting for `wan-r1` only. All other IOS devices still use the group setting. Full variable precedence is covered in depth in Part 11.

---

## Dynamic Inventory

So far I've been maintaining a static inventory, a YAML file I update manually whenever a device is added, removed, or changed. This works fine for a lab with 9 devices. It doesn't work for an enterprise with 500 devices changing constantly.

Dynamic inventory solves this by generating the inventory automatically from an external source of truth. Typically a CMDB, IPAM, or network management platform. Instead of a static YAML file, I provide a dynamic inventory plugin that queries the source of truth at runtime and returns a live inventory.

---

{{% steps %}}

#### How Dynamic Inventory Works

```
ansible-playbook site.yml -i inventory/
         │
         ▼
Ansible sees inventory/ directory
         │
         ├── hosts.yml         ← Static hosts (if any)
         └── netbox.yml        ← Dynamic inventory plugin config
                  │
                  ▼
         Plugin queries Netbox API
                  │
                  ▼
         Returns host list + variables
                  │
                  ▼
         Ansible merges static + dynamic inventory
         and proceeds with playbook execution
```

The key insight: from a playbook's perspective, there's no difference between a statically-defined host and a dynamically-discovered one. Both show up the same way. The difference is only in how the inventory is maintained.

#### The Netbox Dynamic Inventory Plugin

Netbox is the industry-standard network source of truth. When Netbox is running, I replace (or supplement) my static `hosts.yml` with a Netbox inventory plugin config.

Here's what that config file looks like (I'm creating it now even though Netbox isn't running yet) so the structure is familiar when I get to it:

```bash
nano ~/projects/ansible-network/inventory/netbox.yml
```

```yaml
---
# =============================================================
# Netbox Dynamic Inventory Plugin Configuration
# This file will be active once Netbox is set up in Part 26
# For now, comment out the plugin line to prevent errors
# =============================================================

# Uncomment when Netbox is running:
# plugin: netbox.netbox.nb_inventory

plugin: netbox.netbox.nb_inventory

# --- Netbox Connection ---
api_endpoint: http://192.168.1.100:8000    # Replace with actual Netbox IP
token: "{{ lookup('env', 'NETBOX_TOKEN') }}"    # Token from environment variable
validate_certs: false                           # Lab — self-signed cert

# --- What to pull from Netbox ---
# Pull devices (not virtual machines) into the inventory
compose:
  ansible_host: primary_ip4.address | ipaddr('address')

# --- Grouping ---
# Create Ansible groups based on Netbox attributes
group_by:
  - device_roles
  - platforms
  - sites
  - tags

# --- Filters ---
# Only pull active devices
device_query_filters:
  - status: active

# Only pull devices that have a primary IP set
# (devices without IPs can't be automated)
filters:
  has_primary_ip: true

# --- Variable mapping ---
# Map Netbox fields to Ansible variables
compose:
  ansible_network_os: >-
    {
      'cisco-ios-xe': 'cisco.ios.ios',
      'cisco-nxos': 'cisco.nxos.nxos',
      'panos': 'paloaltonetworks.panos.panos'
    }[platform.slug] | default('unknown')
  device_role: device_role.slug
  site_name: site.name
  rack: rack.name | default('unknown')
```

**What this configuration does when active:**
- Connects to the Netbox API and pulls all active devices that have a primary IP
- Creates Ansible groups automatically based on device role, platform, site, and tags defined in Netbox
- Maps Netbox's platform slugs to Ansible's `ansible_network_os` values
- Pulls device-specific data (rack, site, role) as Ansible variables automatically

>[!Info]
> The `token: "{{ lookup('env', 'NETBOX_TOKEN') }}"` syntax reads the API token from an environment variable rather than hardcoding it in the file. The config file goes in Git, the token stays out of Git. I set the environment variable before running Ansible: `export NETBOX_TOKEN=my_token_here`. This can also be stored in a `.env` file (which is in `.gitignore`) and sourced before running playbooks.

#### Static vs Dynamic

| Scenario | Use |
|---|---|
| Lab environment with fixed devices | Static (`hosts.yml`) |
| Production with 50+ devices | Dynamic (Netbox plugin) |
| Devices change frequently (cloud, SDN) | Dynamic |
| No source of truth system available | Static |
| Mixed environment (some in Netbox, some not) | Both — Ansible merges them |
| CI/CD pipeline where inventory must be deterministic | Static (pinned) |

For this lab, I use static inventory through Part 25. Part 26 migrates to Netbox dynamic inventory.

{{% /steps %}}

---

## Testing the Inventory

With the full inventory built out, I validate it thoroughly before running any playbooks against it.

---

{{% steps %}}

#### `ansible-inventory --graph`

The graph view shows the group hierarchy visually:

```bash
cd ~/projects/ansible-network
ansible-inventory -i inventory/ --graph
```

Expected output:
```
@all:
  |--@cisco_ios:
  |  |--@wan:
  |  |  |--wan-r1
  |  |  |--wan-r2
  |--@cisco_nxos:
  |  |--@spine_switches:
  |  |  |--spine-01
  |  |  |--spine-02
  |  |--@leaf_switches:
  |  |  |--leaf-01
  |  |  |--leaf-02
  |--@paloalto:
  |  |--fw-01
  |--@linux_hosts:
  |  |--host-01
  |  |--host-02
  |--@wan_edge:
  |  |--wan-r1
  |  |--wan-r2
  |  |--fw-01
  |--@datacenter:
  |  |--spine-01
  |  |--spine-02
  |  |--leaf-01
  |  |--leaf-02
  |--@network_devices:
  |  |--wan-r1
  |  |--wan-r2
  |  |--fw-01
  |  |--spine-01
  |  |--spine-02
  |  |--leaf-01
  |  |--leaf-02
  |--@ungrouped:
```

#### `ansible-inventory --list`

The list view shows the full inventory as JSON including all variables:

```bash
ansible-inventory -i inventory/ --list
```

This is verbose but useful for confirming that variables from `group_vars` and `host_vars` are being read correctly.

#### `ansible-inventory --host <hostname>`

Shows all variables that apply to a specific host — including inherited group variables:

```bash
ansible-inventory -i inventory/ --host wan-r1
```

Expected output:
```json
{
    "ansible_become": true,
    "ansible_become_method": "enable",
    "ansible_become_password": "ansible123",
    "ansible_command_timeout": 30,
    "ansible_connection": "network_cli",
    "ansible_host": "172.16.0.11",
    "ansible_network_os": "cisco.ios.ios",
    "ansible_password": "ansible123",
    "ansible_persistent_connect_timeout": 30,
    "ansible_user": "ansible",
    "backup_dir": "backups/cisco_ios",
    "bgp_as": 65100,
    "bgp_neighbors": [...],
    "device_hostname": "wan-r1",
    "device_location": "HQ-DC-Rack-A1",
    "device_role": "wan_router",
    "dns_servers": ["8.8.8.8", "8.8.4.4"],
    "domain_name": "lab.local",
    "interfaces": {...},
    "loopback0": {...},
    "ntp_servers": ["216.239.35.0", "216.239.35.4"],
    "ospf_area": 0,
    ...
}
```

This output shows exactly what variables Ansible will have available when running a task against `wan-r1`. If a variable is missing here, it won't be available in playbooks or templates.

>[!Tip]
> I run `ansible-inventory --host <device>` whenever a playbook fails with an `undefined variable` error. It immediately shows whether the variable exists in the inventory at all, and if so, what value it has. This is faster than grep-ing through multiple `group_vars` and `host_vars` files manually.

#### Testing Connectivity Against the Inventory

With the lab running, I test that Ansible can actually reach all devices:

Test all IOS-XE devices

```bash
ansible cisco_ios -i inventory/ -m ansible.netcommon.net_ping
```

Test all NX-OS devices

```bash
ansible cisco_nxos -i inventory/ -m ansible.netcommon.net_ping
```

Test all network devices at once

```bash
ansible network_devices -i inventory/ -m ansible.netcommon.net_ping
```

Test Linux hosts

```bash
ansible linux_hosts -i inventory/ -m ansible.builtin.ping
```

#### Limiting to Specific Hosts During Testing

Run against a single host

```bash
ansible cisco_ios -i inventory/ -m ansible.netcommon.net_ping --limit wan-r1
```

Run against a pattern

```bash
ansible all -i inventory/ -m ansible.netcommon.net_ping --limit "spine*"
```

Run against everything except one device

```bash
ansible all -i inventory/ -m ansible.netcommon.net_ping --limit "all,!fw-01"
```

{{% /steps %}}

---

## Committing the Expanded Inventory to Git

```bash
cd ~/projects/ansible-network
```

```bash
git add inventory/
git status
```

```bash
git commit -m "feat(inventory): expand inventory with full group_vars and host_vars

- Add comprehensive group_vars for all platforms (cisco_ios, cisco_nxos,
  paloalto, linux_hosts, spine_switches, leaf_switches, wan, all)
- Add realistic host_vars for all 9 lab devices with interface data,
  routing parameters, BGP AS numbers, and device metadata
- Add logical group structure (wan_edge, datacenter, network_devices)
- Add Netbox dynamic inventory config (inactive until Part 26)
- Expand hosts.yml with full group hierarchy and comments"
```

---


The inventory is now a proper data source. Not just a list of IPs but a structured representation of the entire network with per-device variables, platform groups, logical groups, and the foundation for future dynamic inventory.


