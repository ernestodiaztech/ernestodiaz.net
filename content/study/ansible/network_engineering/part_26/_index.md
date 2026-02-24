---
draft: true
title: '26 - Netbox'
weight: 26
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 26: Netbox Integration — Dynamic Inventory & Source of Truth

> *Every part of this guide so far has stored device data in YAML files — host_vars, group_vars, data models. That works well for a single engineer managing a known set of devices. It breaks down when the team grows, devices are added and removed regularly, and multiple playbooks need to agree on what IPs, platforms, and roles exist. Netbox solves this by being a single authoritative database for network infrastructure. When Ansible reads its inventory from Netbox instead of static YAML files, the inventory is always current — because it reflects what Netbox says exists, not what someone last remembered to update in a file.*

---

## 26.1 — Netbox as Source of Truth

Netbox is an open-source network documentation and IPAM (IP Address Management) tool. In the context of Ansible automation it serves two roles:

```
Role 1 — Dynamic Inventory Source
  Ansible queries Netbox API at runtime
  Devices, groups, and variables come from Netbox
  No static hosts files or host_vars needed for inventory
  Filters let playbooks target devices by site/role/tag/platform

Role 2 — Post-Deployment Record
  After Ansible deploys a config, it updates Netbox
  Interface IPs, device status, cable connections stay current
  Netbox reflects actual deployed state, not planned state
  Other teams (NOC, capacity planning) see accurate data
```

The relationship between Ansible and Netbox flows in both directions:

```
Netbox (source of truth)
    │
    ├──→ Ansible reads inventory + variables at playbook run time
    │         (netbox.yml dynamic inventory plugin)
    │
    └──← Ansible writes back after deployment
              (netbox.netbox collection modules)
```

### Netbox Data Model Relevant to This Guide

Before configuring the inventory plugin, understanding the Netbox object hierarchy that maps to Ansible inventory:

```
Sites          → Ansible groups (lab-site, prod-site)
Device Roles   → Ansible groups (wan-router, core-switch, firewall)
Platforms      → ansible_network_os variable (cisco_ios, cisco_nxos, junos)
Device Types   → Hardware model (CSR1000v, N9Kv, vJunos)
Devices        → Ansible hosts
  ├── Primary IP → ansible_host variable
  ├── Custom fields → arbitrary Ansible variables
  └── Tags       → additional Ansible groups
Interfaces     → Interface data (name, IP, description, enabled)
IP Addresses   → Assigned to interfaces, linked to devices
Prefixes       → Subnet documentation
```

---

## 26.2 — Collection Setup and API Access

```bash
# Install the Netbox collection
ansible-galaxy collection install netbox.netbox

# Install the pynetbox Python library (required by the collection)
pip install pynetbox --break-system-packages

# Verify
python3 -c "import pynetbox; print('pynetbox OK')"
ansible-galaxy collection list | grep netbox
```

### API Token

```bash
# Generate an API token in Netbox:
# Netbox UI → Admin → API Tokens → Add Token
# Or via API:
curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"user": 1, "description": "Ansible automation token"}' \
  http://localhost:8000/api/users/tokens/ \
  | python3 -m json.tool | grep '"key"'

# Store in vault
ansible-vault encrypt_string --vault-id lab@.vault/lab.txt \
  'your_token_here' --name 'vault_netbox_token'
```

```bash
# group_vars/all/netbox.yml
cat > ~/projects/ansible-network/inventory/group_vars/all/netbox.yml << 'EOF'
---
netbox_url: "http://172.16.0.250:8000"    # Netbox instance URL
netbox_token: "{{ vault_netbox_token }}"   # API token from vault
EOF
```

---

## 26.3 — Populating Netbox with Lab Devices

Before the dynamic inventory plugin can return anything useful, the lab devices need to exist in Netbox. This playbook creates the minimum required objects: one site, one device role per platform, one platform per NOS, and the devices themselves with management IPs.

```bash
cat > ~/projects/ansible-network/playbooks/netbox/populate_lab.yml << 'EOF'
---
# =============================================================
# populate_lab.yml — Create minimum Netbox objects for lab inventory
# Run once to seed Netbox before using dynamic inventory
#
# Creates: site, manufacturers, device types, platforms,
#          device roles, devices, management interfaces, IPs
# =============================================================

- name: "Netbox | Populate lab infrastructure"
  hosts: localhost
  gather_facts: false
  connection: local

  vars:
    nb_url: "{{ netbox_url }}"
    nb_token: "{{ netbox_token }}"

  tasks:

    # ── Site ──────────────────────────────────────────────────
    - name: "Netbox | Create lab site"
      netbox.netbox.netbox_site:
        netbox_url: "{{ nb_url }}"
        netbox_token: "{{ nb_token }}"
        data:
          name: "Containerlab"
          slug: "containerlab"
          description: "Local Containerlab network automation lab"
          status: active
          physical_address: "Localhost"
        state: present

    # ── Manufacturers ─────────────────────────────────────────
    - name: "Netbox | Create manufacturers"
      netbox.netbox.netbox_manufacturer:
        netbox_url: "{{ nb_url }}"
        netbox_token: "{{ nb_token }}"
        data:
          name: "{{ item.name }}"
          slug: "{{ item.slug }}"
        state: present
      loop:
        - { name: "Cisco", slug: "cisco" }
        - { name: "Juniper Networks", slug: "juniper-networks" }
        - { name: "Palo Alto Networks", slug: "palo-alto-networks" }
      loop_control:
        label: "Manufacturer: {{ item.name }}"

    # ── Device Types ──────────────────────────────────────────
    - name: "Netbox | Create device types (virtual lab models)"
      netbox.netbox.netbox_device_type:
        netbox_url: "{{ nb_url }}"
        netbox_token: "{{ nb_token }}"
        data:
          manufacturer: "{{ item.manufacturer }}"
          model: "{{ item.model }}"
          slug: "{{ item.slug }}"
          is_full_depth: false
          u_height: 1
        state: present
      loop:
        - { manufacturer: "Cisco", model: "CSR1000v", slug: "csr1000v" }
        - { manufacturer: "Cisco", model: "Nexus 9000v", slug: "n9kv" }
        - { manufacturer: "Juniper Networks", model: "vJunos", slug: "vjunos" }
        - { manufacturer: "Palo Alto Networks", model: "PA-VM", slug: "pa-vm" }
      loop_control:
        label: "Device type: {{ item.model }}"

    # ── Platforms ─────────────────────────────────────────────
    # Platform slug maps directly to ansible_network_os in netbox.yml inventory
    - name: "Netbox | Create platforms"
      netbox.netbox.netbox_platform:
        netbox_url: "{{ nb_url }}"
        netbox_token: "{{ nb_token }}"
        data:
          name: "{{ item.name }}"
          slug: "{{ item.slug }}"
          manufacturer: "{{ item.manufacturer }}"
          napalm_driver: "{{ item.napalm_driver | default(omit) }}"
        state: present
      loop:
        - name: "Cisco IOS"
          slug: "cisco-ios"
          manufacturer: "Cisco"
          napalm_driver: "ios"
        - name: "Cisco IOS-XE"
          slug: "cisco-ios-xe"
          manufacturer: "Cisco"
          napalm_driver: "ios"
        - name: "Cisco NX-OS"
          slug: "cisco-nxos"
          manufacturer: "Cisco"
          napalm_driver: "nxos"
        - name: "Juniper Junos"
          slug: "juniper-junos"
          manufacturer: "Juniper Networks"
          napalm_driver: "junos"
        - name: "PAN-OS"
          slug: "pan-os"
          manufacturer: "Palo Alto Networks"
      loop_control:
        label: "Platform: {{ item.name }}"

    # ── Device Roles ──────────────────────────────────────────
    - name: "Netbox | Create device roles"
      netbox.netbox.netbox_device_role:
        netbox_url: "{{ nb_url }}"
        netbox_token: "{{ nb_token }}"
        data:
          name: "{{ item.name }}"
          slug: "{{ item.slug }}"
          color: "{{ item.color }}"
          vm_role: false
        state: present
      loop:
        - { name: "WAN Router",     slug: "wan-router",     color: "2196f3" }
        - { name: "Core Switch",    slug: "core-switch",    color: "4caf50" }
        - { name: "Spine Switch",   slug: "spine-switch",   color: "9c27b0" }
        - { name: "Leaf Switch",    slug: "leaf-switch",    color: "ff9800" }
        - { name: "Firewall",       slug: "firewall",       color: "f44336" }
      loop_control:
        label: "Role: {{ item.name }}"

    # ── Devices ───────────────────────────────────────────────
    - name: "Netbox | Create lab devices"
      netbox.netbox.netbox_device:
        netbox_url: "{{ nb_url }}"
        netbox_token: "{{ nb_token }}"
        data:
          name: "{{ item.name }}"
          device_type: "{{ item.device_type }}"
          role: "{{ item.role }}"
          platform: "{{ item.platform }}"
          site: "Containerlab"
          status: active
          tags: "{{ item.tags | default([]) }}"
          custom_fields:
            ansible_network_os: "{{ item.ansible_network_os }}"
            ansible_connection: "{{ item.ansible_connection }}"
            mgmt_ip: "{{ item.mgmt_ip }}"
        state: present
      loop:
        - name: "wan-r1"
          device_type: "CSR1000v"
          role: "WAN Router"
          platform: "Cisco IOS"
          ansible_network_os: "cisco.ios.ios"
          ansible_connection: "network_cli"
          mgmt_ip: "172.16.0.11"
          tags: ["lab", "ios", "ccna"]
        - name: "wan-r2"
          device_type: "CSR1000v"
          role: "WAN Router"
          platform: "Cisco IOS"
          ansible_network_os: "cisco.ios.ios"
          ansible_connection: "network_cli"
          mgmt_ip: "172.16.0.12"
          tags: ["lab", "ios", "ccna"]
        - name: "spine-01"
          device_type: "Nexus 9000v"
          role: "Spine Switch"
          platform: "Cisco NX-OS"
          ansible_network_os: "cisco.nxos.nxos"
          ansible_connection: "network_cli"
          mgmt_ip: "172.16.0.31"
          tags: ["lab", "nxos", "datacenter"]
        - name: "spine-02"
          device_type: "Nexus 9000v"
          role: "Spine Switch"
          platform: "Cisco NX-OS"
          ansible_network_os: "cisco.nxos.nxos"
          ansible_connection: "network_cli"
          mgmt_ip: "172.16.0.32"
          tags: ["lab", "nxos", "datacenter"]
        - name: "leaf-01"
          device_type: "Nexus 9000v"
          role: "Leaf Switch"
          platform: "Cisco NX-OS"
          ansible_network_os: "cisco.nxos.nxos"
          ansible_connection: "network_cli"
          mgmt_ip: "172.16.0.33"
          tags: ["lab", "nxos", "datacenter"]
        - name: "leaf-02"
          device_type: "Nexus 9000v"
          role: "Leaf Switch"
          platform: "Cisco NX-OS"
          ansible_network_os: "cisco.nxos.nxos"
          ansible_connection: "network_cli"
          mgmt_ip: "172.16.0.34"
          tags: ["lab", "nxos", "datacenter"]
        - name: "vjunos-01"
          device_type: "vJunos"
          role: "WAN Router"
          platform: "Juniper Junos"
          ansible_network_os: "junipernetworks.junos.junos"
          ansible_connection: "netconf"
          mgmt_ip: "172.16.0.41"
          tags: ["lab", "junos"]
        - name: "panos-fw01"
          device_type: "PA-VM"
          role: "Firewall"
          platform: "PAN-OS"
          ansible_network_os: "paloaltonetworks.panos.panos"
          ansible_connection: "local"
          mgmt_ip: "172.16.0.51"
          tags: ["lab", "panos", "firewall"]
      loop_control:
        label: "Device: {{ item.name }}"

    # ── Management interfaces and IPs ─────────────────────────
    - name: "Netbox | Create management interfaces"
      netbox.netbox.netbox_device_interface:
        netbox_url: "{{ nb_url }}"
        netbox_token: "{{ nb_token }}"
        data:
          device: "{{ item.device }}"
          name: "{{ item.iface }}"
          type: virtual
          mgmt_only: true
          description: "Out-of-band management"
        state: present
      loop:
        - { device: "wan-r1",     iface: "Management1" }
        - { device: "wan-r2",     iface: "Management1" }
        - { device: "spine-01",   iface: "mgmt0" }
        - { device: "spine-02",   iface: "mgmt0" }
        - { device: "leaf-01",    iface: "mgmt0" }
        - { device: "leaf-02",    iface: "mgmt0" }
        - { device: "vjunos-01",  iface: "fxp0" }
        - { device: "panos-fw01", iface: "Management" }
      loop_control:
        label: "{{ item.device }}/{{ item.iface }}"

    - name: "Netbox | Assign management IPs and set as primary"
      netbox.netbox.netbox_ip_address:
        netbox_url: "{{ nb_url }}"
        netbox_token: "{{ nb_token }}"
        data:
          address: "{{ item.ip }}/24"
          status: active
          assigned_object_type: dcim.interface
          assigned_object:
            device: "{{ item.device }}"
            name: "{{ item.iface }}"
          description: "Management IP for {{ item.device }}"
        state: present
      loop:
        - { device: "wan-r1",     iface: "Management1", ip: "172.16.0.11" }
        - { device: "wan-r2",     iface: "Management1", ip: "172.16.0.12" }
        - { device: "spine-01",   iface: "mgmt0",       ip: "172.16.0.31" }
        - { device: "spine-02",   iface: "mgmt0",       ip: "172.16.0.32" }
        - { device: "leaf-01",    iface: "mgmt0",       ip: "172.16.0.33" }
        - { device: "leaf-02",    iface: "mgmt0",       ip: "172.16.0.34" }
        - { device: "vjunos-01",  iface: "fxp0",        ip: "172.16.0.41" }
        - { device: "panos-fw01", iface: "Management",  ip: "172.16.0.51" }
      loop_control:
        label: "{{ item.device }}: {{ item.ip }}"
      register: ip_results

    - name: "Netbox | Set primary IPv4 on each device"
      netbox.netbox.netbox_device:
        netbox_url: "{{ nb_url }}"
        netbox_token: "{{ nb_token }}"
        data:
          name: "{{ item.device }}"
          primary_ip4: "{{ item.ip }}/24"
        state: present
      loop:
        - { device: "wan-r1",     ip: "172.16.0.11" }
        - { device: "wan-r2",     ip: "172.16.0.12" }
        - { device: "spine-01",   ip: "172.16.0.31" }
        - { device: "spine-02",   ip: "172.16.0.32" }
        - { device: "leaf-01",    ip: "172.16.0.33" }
        - { device: "leaf-02",    ip: "172.16.0.34" }
        - { device: "vjunos-01",  ip: "172.16.0.41" }
        - { device: "panos-fw01", ip: "172.16.0.51" }
      loop_control:
        label: "{{ item.device }} primary IP: {{ item.ip }}"
EOF
```

### Required Netbox Custom Fields

The playbook uses custom fields (`ansible_network_os`, `ansible_connection`, `mgmt_ip`) to store Ansible-specific variables in Netbox. Create these in Netbox before running the playbook:

```
Netbox UI → Customization → Custom Fields → Add
  Field 1:
    Content types: dcim | device
    Type: Text
    Name: ansible_network_os
    Label: Ansible Network OS

  Field 2:
    Content types: dcim | device
    Type: Text
    Name: ansible_connection
    Label: Ansible Connection

  Field 3:
    Content types: dcim | device
    Type: Text
    Name: mgmt_ip
    Label: Management IP
```

---

## 26.4 — The Dynamic Inventory Plugin: Full Reference

The `netbox.yml` file replaces all static inventory files. Ansible calls the Netbox API at run time, builds the inventory from the response, and groups devices according to the filter configuration.

```bash
mkdir -p ~/projects/ansible-network/inventory/netbox

cat > ~/projects/ansible-network/inventory/netbox/netbox.yml << 'EOF'
---
# =============================================================
# netbox.yml — Netbox dynamic inventory plugin configuration
#
# This file replaces hosts.yml for environments where Netbox
# is the source of truth.
#
# Usage:
#   ansible-inventory -i inventory/netbox/netbox.yml --list
#   ansible-playbook playbook.yml -i inventory/netbox/netbox.yml
#
# Or set as default in ansible.cfg:
#   [defaults]
#   inventory = inventory/netbox/netbox.yml
# =============================================================

plugin: netbox.netbox.nb_inventory

# ── Connection ────────────────────────────────────────────────────
api_endpoint: "http://172.16.0.250:8000"
token: "{{ lookup('env', 'NETBOX_TOKEN') | default(lookup('vars', 'netbox_token')) }}"
# Token lookup order:
#   1. NETBOX_TOKEN environment variable (CI/CD pipelines)
#   2. netbox_token Ansible variable (from vault)
validate_certs: false     # Set true in production with valid TLS cert

# ── Device filters ────────────────────────────────────────────────
# Only return devices that are active — ignore planned/decommissioned
device_query_filters:
  - status: "active"

# Optionally filter by site, role, or tag:
# device_query_filters:
#   - status: "active"
#   - site: "containerlab"         # Only this site
#   - role: "wan-router"           # Only this role
#   - tag: "production"            # Only tagged production

# ── Group construction ────────────────────────────────────────────
# Each setting controls how Ansible groups are created from Netbox data

# Group by site (group name = site slug)
group_by:
  - sites          # Creates groups: site_containerlab, site_production, etc.
  - device_roles   # Creates groups: device_roles_wan_router, device_roles_firewall, etc.
  - platforms      # Creates groups: platform_cisco_ios, platform_cisco_nxos, etc.
  - tags           # Creates groups: tag_lab, tag_datacenter, tag_production, etc.
  - manufacturers  # Creates groups: manufacturer_cisco, manufacturer_juniper, etc.

# ── Variable mapping ──────────────────────────────────────────────
# Controls what Netbox data becomes Ansible variables on each host

# Use the device's primary IP as ansible_host
compose:
  ansible_host: primary_ip4.address | ipaddr('address')
  # ipaddr('address') strips the prefix length (/24) leaving just the IP

  # Map platform slug to ansible_network_os
  # Platform slug in Netbox must match the collection's network_os value
  ansible_network_os: >-
    platform.slug | replace('-', '.') if platform else 'unknown'
  # This maps 'cisco-ios' → 'cisco.ios' — close but needs the final segment
  # Better: use custom fields for precise control (see below)

  # Pull ansible_network_os from custom field (most reliable approach)
  ansible_network_os: custom_fields.ansible_network_os
  ansible_connection: custom_fields.ansible_connection

# ── Variable enrichment ───────────────────────────────────────────
# Pull additional Netbox data into host variables
device_query_filters: []

# Include interface data on each host
interfaces: true           # Adds hostvars['interfaces'] with all interface data
services: false            # Service records (not needed for network devices)

# ── Keyed groups ──────────────────────────────────────────────────
# Create custom groups based on any device attribute or custom field
keyed_groups:
  # Group by platform slug — gives clean group names
  - prefix: ""
    key: platform.slug
    separator: ""
    # Results in groups: cisco-ios, cisco-nxos, juniper-junos, pan-os

  # Group by device role slug
  - prefix: ""
    key: device_roles[0].slug
    separator: ""
    # Results in groups: wan-router, spine-switch, leaf-switch, firewall

  # Group by tag name
  - prefix: "tag_"
    key: tags | map(attribute='slug') | list
    separator: ""
    # Results in groups: tag_lab, tag_ios, tag_datacenter

  # Custom group: Cisco devices only
  - prefix: "vendor_"
    key: device_type.manufacturer.slug
    separator: ""
    # Results in groups: vendor_cisco, vendor_juniper-networks

# ── Host variables from Netbox ────────────────────────────────────
# These become available as hostvars for every host in the inventory

# Include all custom fields as host variables
# Custom fields defined in Netbox become top-level hostvars
host_vars:
  include_named_hosts: true

# ── Group variables ───────────────────────────────────────────────
# Map Netbox groups to connection settings
# These supplement or replace group_vars files

group_vars:
  cisco-ios:
    ansible_become: true
    ansible_become_method: enable
  cisco-nxos:
    ansible_become: false
  juniper-junos:
    ansible_port: 830
  pan-os:
    ansible_connection: local
EOF
```

### Understanding the Group Structure Output

Running `ansible-inventory` against this plugin shows what groups are created:

```bash
# Preview the full inventory structure from Netbox
ansible-inventory \
  -i inventory/netbox/netbox.yml \
  --list \
  | python3 -m json.tool | head -80

# Tree view of groups and hosts
ansible-inventory \
  -i inventory/netbox/netbox.yml \
  --graph

# Expected output:
# @all:
#   |--@ungrouped:
#   |--@site_containerlab:
#   |  |--wan-r1
#   |  |--wan-r2
#   |  |--spine-01
#   |  |--spine-02
#   |  |--leaf-01
#   |  |--leaf-02
#   |  |--vjunos-01
#   |  |--panos-fw01
#   |--@cisco-ios:
#   |  |--wan-r1
#   |  |--wan-r2
#   |--@cisco-nxos:
#   |  |--spine-01
#   |  |--spine-02
#   |  |--leaf-01
#   |  |--leaf-02
#   |--@juniper-junos:
#   |  |--vjunos-01
#   |--@pan-os:
#   |  |--panos-fw01
#   |--@wan-router:
#   |  |--wan-r1
#   |  |--wan-r2
#   |  |--vjunos-01
#   |--@firewall:
#   |  |--panos-fw01
#   |--@tag_lab:
#   |  |--wan-r1 ...

# Inspect variables for a specific host
ansible-inventory \
  -i inventory/netbox/netbox.yml \
  --host wan-r1 \
  | python3 -m json.tool
```

### What Variables a Host Gets from Netbox

```json
{
  "ansible_host": "172.16.0.11",
  "ansible_network_os": "cisco.ios.ios",
  "ansible_connection": "network_cli",
  "device_type": {
    "model": "CSR1000v",
    "slug": "csr1000v",
    "manufacturer": { "name": "Cisco", "slug": "cisco" }
  },
  "platform": {
    "name": "Cisco IOS",
    "slug": "cisco-ios"
  },
  "device_roles": [{ "name": "WAN Router", "slug": "wan-router" }],
  "site": { "name": "Containerlab", "slug": "containerlab" },
  "tags": [
    { "name": "lab", "slug": "lab" },
    { "name": "ios", "slug": "ios" }
  ],
  "custom_fields": {
    "ansible_network_os": "cisco.ios.ios",
    "ansible_connection": "network_cli",
    "mgmt_ip": "172.16.0.11"
  },
  "interfaces": [
    {
      "name": "Management1",
      "type": { "value": "virtual" },
      "mgmt_only": true,
      "ip_addresses": [{ "address": "172.16.0.11/24" }]
    },
    {
      "name": "GigabitEthernet1",
      "description": "WAN | To upstream provider",
      "ip_addresses": [{ "address": "10.10.10.1/30" }]
    }
  ]
}
```

---

## 26.5 — Worked Example: Playbook Against Netbox Inventory

This playbook runs against the Netbox-sourced inventory. It uses groups and variables that come entirely from Netbox — no static host_vars files, no hardcoded IPs.

```bash
cat > ~/projects/ansible-network/playbooks/netbox/deploy_from_netbox.yml << 'EOF'
---
# =============================================================
# deploy_from_netbox.yml — Deploy config using Netbox as inventory
#
# Run with Netbox inventory:
#   ansible-playbook deploy_from_netbox.yml \
#     -i inventory/netbox/netbox.yml
#
# Filter to a site:
#   ansible-playbook deploy_from_netbox.yml \
#     -i inventory/netbox/netbox.yml \
#     --limit site_containerlab
#
# Filter to a role:
#   ansible-playbook deploy_from_netbox.yml \
#     -i inventory/netbox/netbox.yml \
#     --limit wan-router
# =============================================================

# ── Play 1: IOS devices (group from Netbox platform slug) ────────
- name: "Deploy | IOS devices from Netbox inventory"
  hosts: cisco-ios    # This group is created by the keyed_groups platform slug
  gather_facts: false
  connection: "{{ ansible_connection | default('network_cli') }}"
  become: true
  become_method: enable

  tasks:
    - name: "Netbox | Display host variables from Netbox"
      ansible.builtin.debug:
        msg:
          - "Host: {{ inventory_hostname }}"
          - "ansible_host: {{ ansible_host }}"
          - "Platform: {{ platform.name }}"
          - "Role: {{ device_roles[0].name }}"
          - "Site: {{ site.name }}"
          - "Tags: {{ tags | map(attribute='slug') | list | join(', ') }}"
          - "Custom fields: {{ custom_fields }}"
        verbosity: 1

    - name: "Netbox | Gather IOS facts"
      cisco.ios.ios_facts:
        gather_subset: [default, interfaces]

    - name: "NTP | Configure NTP from group_vars (still used for config data)"
      cisco.ios.ios_ntp_global:
        config:
          servers:
            - server: "{{ item }}"
        state: merged
      loop: "{{ ntp_servers | default([]) }}"
      loop_control:
        label: "NTP: {{ item }}"
      # Note: inventory (who to connect to) comes from Netbox
      #       config data (what to push) still comes from group_vars/host_vars
      #       This is the hybrid model — Netbox for inventory, YAML for config
      notify: Save IOS configuration

    - name: "Interfaces | Configure interfaces from Netbox interface data"
      cisco.ios.ios_interfaces:
        config:
          - name: "{{ item.name }}"
            description: "{{ item.description | default('') }}"
            enabled: "{{ not item.disabled | default(false) }}"
        state: merged
      loop: "{{ interfaces | default([]) | rejectattr('mgmt_only', 'defined') | list }}"
      loop_control:
        label: "{{ item.name }}"
      # Interface data comes directly from Netbox hostvars['interfaces']
      # No host_vars file needed for interface descriptions
      notify: Save IOS configuration

  handlers:
    - name: Save IOS configuration
      cisco.ios.ios_command:
        commands: [write memory]
      listen: Save IOS configuration

# ── Play 2: NX-OS devices ─────────────────────────────────────────
- name: "Deploy | NX-OS devices from Netbox inventory"
  hosts: cisco-nxos
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Netbox | Verify device role from Netbox data"
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} role: {{ device_roles[0].slug }}"

    - name: "Features | Enable features based on device role"
      cisco.nxos.nxos_feature:
        feature: "{{ item }}"
        state: enabled
      loop: "{{ spine_features if device_roles[0].slug == 'spine-switch' else leaf_features }}"
      vars:
        spine_features: [bgp, ospf, lldp]
        leaf_features: [bgp, ospf, vpc, lldp, lacp]
      loop_control:
        label: "Feature: {{ item }} on {{ device_roles[0].slug }}"

# ── Play 3: Cross-platform health check ──────────────────────────
- name: "Health | Cross-platform check using Netbox groups"
  hosts: site_containerlab    # All devices in the Containerlab site
  gather_facts: false

  tasks:
    - name: "Health | IOS reachability check"
      cisco.ios.ios_facts:
        gather_subset: [default]
      when: ansible_network_os == 'cisco.ios.ios'
      changed_when: false

    - name: "Health | NX-OS reachability check"
      cisco.nxos.nxos_facts:
        gather_subset: [default]
      when: ansible_network_os == 'cisco.nxos.nxos'
      changed_when: false

    - name: "Health | Display device status"
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} ({{ ansible_network_os }}) — reachable ✓"
EOF
```

---

## 26.6 — Closing the Loop: Updating Netbox After Deployment

### Pattern 1 — Inline (Simple, tightly coupled)

Best for small playbooks where the Netbox update is a natural part of the same workflow. The deployment and Netbox update are in the same play.

```bash
cat > ~/projects/ansible-network/playbooks/netbox/deploy_and_update.yml << 'EOF'
---
# Pattern 1: Deploy config AND update Netbox in the same play
# Use when: the deployment IS the source of truth change
# Example: provisioning a new interface — deploying it and recording it are one action

- name: "Provision | Deploy new interface and record in Netbox"
  hosts: cisco-ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  vars:
    new_interface:
      name: "GigabitEthernet3"
      description: "New link to DMZ switch"
      ip: "10.50.1.1"
      prefix: 30
      netbox_interface_type: "1000base-t"

  tasks:
    - name: "Deploy | Configure new interface on device"
      cisco.ios.ios_l3_interfaces:
        config:
          - name: "{{ new_interface.name }}"
            ipv4:
              - address: "{{ new_interface.ip }}/{{ new_interface.prefix }}"
        state: merged
      register: interface_deploy
      notify: Save IOS configuration

    - name: "Deploy | Enable interface"
      cisco.ios.ios_interfaces:
        config:
          - name: "{{ new_interface.name }}"
            description: "{{ new_interface.description }}"
            enabled: true
        state: merged
      notify: Save IOS configuration

    # Netbox update runs in the same play, immediately after deploy
    - name: "Netbox | Record new interface in Netbox"
      netbox.netbox.netbox_device_interface:
        netbox_url: "{{ netbox_url }}"
        netbox_token: "{{ netbox_token }}"
        data:
          device: "{{ inventory_hostname }}"
          name: "{{ new_interface.name }}"
          type: "{{ new_interface.netbox_interface_type }}"
          description: "{{ new_interface.description }}"
          enabled: true
        state: present
      delegate_to: localhost
      # delegate_to: localhost — Netbox API calls always run on control node

    - name: "Netbox | Record IP address in Netbox"
      netbox.netbox.netbox_ip_address:
        netbox_url: "{{ netbox_url }}"
        netbox_token: "{{ netbox_token }}"
        data:
          address: "{{ new_interface.ip }}/{{ new_interface.prefix }}"
          status: active
          assigned_object_type: dcim.interface
          assigned_object:
            device: "{{ inventory_hostname }}"
            name: "{{ new_interface.name }}"
          description: "Deployed by Ansible {{ lookup('pipe', 'date') }}"
        state: present
      delegate_to: localhost

  handlers:
    - name: Save IOS configuration
      cisco.ios.ios_command:
        commands: [write memory]
      listen: Save IOS configuration
EOF
```

### Pattern 2 — Separate Play (Clean, loosely coupled)

Best for large deployments where the deployment playbook should remain platform-focused. A dedicated Netbox sync play runs after and updates Netbox from the deployed state.

```bash
cat > ~/projects/ansible-network/playbooks/netbox/sync_to_netbox.yml << 'EOF'
---
# =============================================================
# sync_to_netbox.yml — Sync actual device state to Netbox
# Pattern 2: runs AFTER deployment, reads device state, updates Netbox
# Safe to run repeatedly — all operations are idempotent
#
# Usage:
#   After any deployment:
#   ansible-playbook deploy_ios_ccna.yml && \
#   ansible-playbook netbox/sync_to_netbox.yml
# =============================================================

- name: "Netbox Sync | Read IOS device state and update Netbox"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  tasks:
    - name: "Sync | Gather current device facts"
      cisco.ios.ios_facts:
        gather_subset: [default, interfaces]
      register: device_facts

    - name: "Sync | Update device status in Netbox (active)"
      netbox.netbox.netbox_device:
        netbox_url: "{{ netbox_url }}"
        netbox_token: "{{ netbox_token }}"
        data:
          name: "{{ inventory_hostname }}"
          status: active
          custom_fields:
            ios_version: "{{ device_facts.ansible_facts.ansible_net_version }}"
            last_synced: "{{ lookup('pipe', 'date --iso-8601=seconds') }}"
        state: present
      delegate_to: localhost

    - name: "Sync | Update interface states in Netbox"
      netbox.netbox.netbox_device_interface:
        netbox_url: "{{ netbox_url }}"
        netbox_token: "{{ netbox_token }}"
        data:
          device: "{{ inventory_hostname }}"
          name: "{{ item.key }}"
          enabled: "{{ device_facts.ansible_facts.ansible_net_interfaces[item.key].operstatus == 'up' }}"
          description: "{{ item.value.description | default('') }}"
          mac_address: "{{ device_facts.ansible_facts.ansible_net_interfaces[item.key].macaddress | default(omit) }}"
        state: present
      loop: "{{ interfaces | dict2items }}"
      loop_control:
        label: "{{ item.key }}"
      delegate_to: localhost
      when: interfaces is defined

    - name: "Sync | Update IP addresses in Netbox from device facts"
      netbox.netbox.netbox_ip_address:
        netbox_url: "{{ netbox_url }}"
        netbox_token: "{{ netbox_token }}"
        data:
          address: >-
            {{ device_facts.ansible_facts.ansible_net_interfaces[item.key].ipv4[0].address }}/{{
               device_facts.ansible_facts.ansible_net_interfaces[item.key].ipv4[0].masklen }}
          status: active
          assigned_object_type: dcim.interface
          assigned_object:
            device: "{{ inventory_hostname }}"
            name: "{{ item.key }}"
        state: present
      loop: >-
        {{ device_facts.ansible_facts.ansible_net_interfaces | dict2items
           | selectattr('value.ipv4', 'defined')
           | selectattr('value.ipv4', 'ne', [])
           | list }}
      loop_control:
        label: "{{ item.key }}: {{ item.value.ipv4[0].address | default('no IP') }}"
      delegate_to: localhost
      failed_when: false    # Non-critical — don't stop sync if one IP fails

- name: "Netbox Sync | Read NX-OS device state and update Netbox"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Sync | Gather NX-OS facts"
      cisco.nxos.nxos_facts:
        gather_subset: [default, interfaces]
      register: nxos_facts

    - name: "Sync | Update NX-OS device in Netbox"
      netbox.netbox.netbox_device:
        netbox_url: "{{ netbox_url }}"
        netbox_token: "{{ netbox_token }}"
        data:
          name: "{{ inventory_hostname }}"
          status: active
          custom_fields:
            nxos_version: "{{ nxos_facts.ansible_facts.ansible_net_version }}"
            last_synced: "{{ lookup('pipe', 'date --iso-8601=seconds') }}"
        state: present
      delegate_to: localhost
EOF
```

### When to Use Each Pattern

```
Pattern 1 — Inline (deploy + update in same play)
  Use when:
  ✓ Provisioning NEW resources (new device, new interface, new IP)
  ✓ The Netbox record doesn't exist yet — deployment creates both
  ✓ The team uses Netbox as the workflow trigger (create in Netbox → deploy)
  ✓ Small playbooks where tight coupling is acceptable

  Avoid when:
  ✗ Deploying to many devices — Netbox API calls slow down the play
  ✗ The deployment playbook is shared across teams who don't use Netbox
  ✗ Netbox is unavailable during a maintenance window

Pattern 2 — Separate sync play
  Use when:
  ✓ Running large deployments across many devices (keep deploy play fast)
  ✓ Syncing existing infrastructure that predates Netbox adoption
  ✓ The deployment playbook is platform-focused and should stay clean
  ✓ You want the option to run sync independently (scheduled, on-demand)
  ✓ Netbox availability shouldn't block network changes

  Recommendation for this lab:
  Use Pattern 2 as the default. Keep deploy playbooks clean and
  platform-focused. Schedule sync_to_netbox.yml to run after each
  deployment pipeline and optionally as a nightly cron job.
```

---

## 26.7 — Transitioning from Static to Netbox Inventory

Moving from the static `hosts.yml` file used through Parts 1-25 to the Netbox dynamic inventory is a one-line change in `ansible.cfg`:

```ini
# ansible.cfg — before (static inventory)
[defaults]
inventory = inventory/hosts.yml

# ansible.cfg — after (Netbox dynamic inventory)
[defaults]
inventory = inventory/netbox/netbox.yml
```

During transition, both can coexist:

```bash
# Use static inventory explicitly
ansible-playbook deploy_ios_ccna.yml -i inventory/hosts.yml

# Use Netbox inventory explicitly
ansible-playbook deploy_ios_ccna.yml -i inventory/netbox/netbox.yml

# Merge both (Ansible supports multiple -i flags)
ansible-playbook deploy_ios_ccna.yml \
  -i inventory/hosts.yml \
  -i inventory/netbox/netbox.yml
# When both are used: host_vars from static files supplement
# the Netbox-sourced inventory variables

# Verify Netbox inventory matches static inventory
diff \
  <(ansible-inventory -i inventory/hosts.yml --list | python3 -m json.tool | grep '"hosts"' -A 20) \
  <(ansible-inventory -i inventory/netbox/netbox.yml --list | python3 -m json.tool | grep '"hosts"' -A 20)
```

---


Netbox integration is complete — the lab now has a living source of truth that both feeds Ansible inventory and receives updates after every deployment. The guide has covered the full network automation stack from control node setup through multi-vendor device management, validation, maintenance, security, and now source-of-truth integration. The next and final part draws everything together.

