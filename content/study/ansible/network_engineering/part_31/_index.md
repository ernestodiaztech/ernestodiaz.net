---
draft: true
title: '31 - Collections'
weight: 31
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 31: Ansible Collections — Going Deeper

> *Collections are how the Ansible ecosystem distributes reusable automation content. Every vendor module used in this guide — cisco.ios, junipernetworks.junos, paloaltonetworks.panos — ships as a collection. But collections aren't just for vendors. There are community collections that add IP address manipulation, data validation, and platform-agnostic network utilities that make playbooks cleaner regardless of which device they're targeting. And when none of the existing collections do exactly what's needed, writing a minimal custom collection takes less time than most engineers expect.*

---

## 31.1 — What a Collection Is

A collection is a distributable package of Ansible content with a consistent directory structure and a `galaxy.yml` manifest that declares its namespace, name, and version.

```
namespace.collection_name   ← fully qualified collection name (FQCN)
    │
    ├── galaxy.yml          ← metadata: name, version, author, dependencies
    ├── README.md
    ├── plugins/
    │   ├── modules/        ← custom modules (cisco.ios.ios_config lives here in cisco.ios)
    │   ├── filter/         ← Jinja2 filter plugins (ansible.utils.ipaddr lives here)
    │   ├── lookup/         ← lookup plugins
    │   └── callback/       ← callback plugins
    ├── roles/              ← reusable roles distributed with the collection
    ├── playbooks/          ← playbooks bundled with the collection
    └── docs/
```

**Namespaces** separate authors: `cisco.ios` and `community.general` can both have a module called `config` without conflict because the namespace is part of the FQCN. The namespace is typically the vendor or GitHub organization name.

**Versioning** follows semantic versioning (MAJOR.MINOR.PATCH). A version pin of `cisco.ios==6.1.0` in `requirements.yml` gives reproducible installs — the same version is installed regardless of when the command runs, which is why Part 23 pinned every collection version.

### How a Collection Maps to Playbook Usage

```yaml
# Without FQCN (short name — works but fails ansible-lint fqcn check)
- ios_config:
    lines: [...]

# With FQCN (correct — namespace.collection.module_name)
- cisco.ios.ios_config:
    lines: [...]
  # │     │   └── module name
  # │     └── collection name
  # └── namespace

# Filters follow the same pattern but use a different syntax
- ansible.builtin.debug:
    msg: "{{ '192.168.1.0/24' | ansible.utils.ipaddr('network') }}"
  #                              │              │      └── filter argument
  #                              │              └── filter name
  #                              └── collection namespace.name
```

### Installed Collection Locations

```bash
# Show all installed collections and their paths
ansible-galaxy collection list

# Default install paths (checked in order):
# 1. ./collections/ansible_collections/    ← project-local (highest priority)
# 2. ~/.ansible/collections/ansible_collections/
# 3. /etc/ansible/collections/ansible_collections/
# 4. Paths in ANSIBLE_COLLECTIONS_PATH environment variable

# Inspect what's inside an installed collection
ls ~/.ansible/collections/ansible_collections/cisco/ios/plugins/modules/ | head -20
ls ~/.ansible/collections/ansible_collections/ansible/utils/plugins/filter/

# Read a module's documentation
ansible-doc cisco.ios.ios_config
ansible-doc -t filter ansible.utils.ipaddr
```

---

## 31.2 — The collections/requirements.yml File

Every collection the project depends on is declared here. This file was introduced in Part 23 for maintenance — here's the full picture of what it supports.

```bash
cat ~/projects/ansible-network/collections/requirements.yml
```

```yaml
---
# collections/requirements.yml
# Install: ansible-galaxy collection install -r collections/requirements.yml
# Upgrade: ansible-galaxy collection install -r collections/requirements.yml --upgrade
# Force:   ansible-galaxy collection install -r collections/requirements.yml --force

collections:

  # ── Ansible built-in and platform-agnostic ─────────────────────
  - name: ansible.netcommon
    version: "==7.1.0"
    # Shared network connection plugins and modules used by all vendor collections

  - name: ansible.utils
    version: "==5.1.2"
    # IP address filters, data validation, network data manipulation

  - name: ansible.posix
    version: "==2.0.0"
    # POSIX platform modules: authorized_key, firewalld, synchronize

  - name: community.general
    version: "==10.1.0"
    # Large community collection: ini_file, json_query, many utilities

  # ── Cisco ──────────────────────────────────────────────────────
  - name: cisco.ios
    version: "==9.0.3"

  - name: cisco.nxos
    version: "==9.2.1"

  # ── Juniper ────────────────────────────────────────────────────
  - name: junipernetworks.junos
    version: "==9.1.0"

  # ── Palo Alto ──────────────────────────────────────────────────
  - name: paloaltonetworks.panos
    version: "==2.21.2"

  # ── Netbox ─────────────────────────────────────────────────────
  - name: netbox.netbox
    version: "==3.20.0"

  # ── Internal custom collection (local install — see Section 31.4) ──
  - name: netops.network_filters
    version: "==1.0.0"
    source: "https://github.com/your-org/netops.network_filters/releases/download/1.0.0/netops-network_filters-1.0.0.tar.gz"
    # Or for local development (not in requirements.yml — install manually):
    # ansible-galaxy collection install ./collections/netops/network_filters/
```

### Installing and Verifying

```bash
# Install all pinned versions
ansible-galaxy collection install -r collections/requirements.yml

# Verify installed versions match requirements
ansible-galaxy collection list | grep -E "netcommon|utils|cisco|juniper|panos|netbox"

# Check for available upgrades without applying them
ansible-galaxy collection install -r collections/requirements.yml --dry-run
```

---

## 31.3 — ansible.utils — Deep Coverage

`ansible.utils` is the most broadly useful non-vendor collection for network automation. It provides IP address manipulation filters, data validation, and structured data utilities that appear in almost every network playbook that does anything with addresses or subnets.

### Installation and Setup

```bash
# Install (already in requirements.yml)
ansible-galaxy collection install ansible.utils

# ansible.utils filters require the netaddr Python library
pip install netaddr --break-system-packages

# Verify
python3 -c "import netaddr; print('netaddr OK')"
ansible-doc -t filter ansible.utils.ipaddr | head -30
```

### The ipaddr Filter — 4-5 Essential Uses

The `ansible.utils.ipaddr` filter is a Swiss-army knife for IP address work. It takes an IP address or CIDR prefix as input and returns different representations or derived values based on the argument passed.

**Use 1 — Extract just the IP address from a CIDR string**

```yaml
# Input from host_vars or device facts often includes the prefix length
# Many modules need just the IP

vars:
  interface_cidr: "10.20.30.1/30"

- ansible.builtin.debug:
    msg: "{{ interface_cidr | ansible.utils.ipaddr('address') }}"
# → 10.20.30.1

# Equivalent: ipv4 filter with 'address' argument
- ansible.builtin.debug:
    msg: "{{ interface_cidr | ansible.utils.ipv4('address') }}"
# → 10.20.30.1  (ipv4 also validates the input is actually IPv4)
```

**Use 2 — Extract the network address and prefix**

```yaml
vars:
  interface_cidr: "10.20.30.5/27"

- ansible.builtin.debug:
    msg:
      - "Network:   {{ interface_cidr | ansible.utils.ipaddr('network') }}"
      - "Broadcast: {{ interface_cidr | ansible.utils.ipaddr('broadcast') }}"
      - "Prefix:    {{ interface_cidr | ansible.utils.ipaddr('prefix') }}"
      - "Netmask:   {{ interface_cidr | ansible.utils.ipaddr('netmask') }}"
      - "Wildcard:  {{ interface_cidr | ansible.utils.ipaddr('wildcard') }}"
      - "Network/prefix: {{ interface_cidr | ansible.utils.ipaddr('network/prefix') }}"
# → Network:   10.20.30.0
# → Broadcast: 10.20.30.31
# → Prefix:    27
# → Netmask:   255.255.255.224
# → Wildcard:  0.0.0.31
# → Network/prefix: 10.20.30.0/27
```

**Use 3 — Validate that a string is a valid IP address**

```yaml
# ipaddr returns false if the input is not a valid IP or CIDR
- ansible.builtin.assert:
    that:
      - item | ansible.utils.ipaddr    # Returns falsy if invalid
    fail_msg: "{{ item }} is not a valid IP address or CIDR"
  loop:
    - "10.0.0.1"          # valid → passes
    - "10.0.0.1/24"       # valid CIDR → passes
    - "not-an-ip"         # invalid → fails assertion
    - "999.0.0.1"         # invalid → fails assertion
```

**Use 4 — Check if an IP is within a subnet**

```yaml
# Useful for validating that interface IPs belong to the right subnet
vars:
  management_subnet: "172.16.0.0/24"
  device_mgmt_ip: "172.16.0.11"

- ansible.builtin.assert:
    that:
      - device_mgmt_ip | ansible.utils.ipaddr(management_subnet)
    fail_msg: "{{ device_mgmt_ip }} is not in management subnet {{ management_subnet }}"
# Returns the IP if it's in the subnet, false if not
```

**Use 5 — Calculate peer IP on a point-to-point /30 link**

```yaml
# Given one end of a /30, calculate the other end
# Useful for generating BGP neighbor configs from interface data
vars:
  local_ip_cidr: "10.10.10.1/30"

- ansible.builtin.set_fact:
    # Get the index of our IP within the /30 (1 or 2)
    local_host_index: "{{ local_ip_cidr | ansible.utils.ipaddr('index') }}"
    # Get the peer: if we're host 1, peer is host 2, and vice versa
    peer_ip: >-
      {{ local_ip_cidr | ansible.utils.ipaddr(
          3 - (local_ip_cidr | ansible.utils.ipaddr('index') | int)
         ) | ansible.utils.ipaddr('address') }}

- ansible.builtin.debug:
    msg:
      - "Local IP:  {{ local_ip_cidr | ansible.utils.ipaddr('address') }}"
      - "Peer IP:   {{ peer_ip }}"
# → Local IP:  10.10.10.1
# → Peer IP:   10.10.10.2
```

### Full ipaddr Documentation

The `ansible.utils.ipaddr` filter has dozens of arguments beyond these five. The authoritative reference:

```bash
# Built-in docs
ansible-doc -t filter ansible.utils.ipaddr

# Online: https://docs.ansible.com/ansible/latest/collections/ansible/utils/docsite/filters_ipaddr.html
# The docs show every argument with examples: address, network, broadcast,
# prefix, netmask, wildcard, host, network_id, size, first_usable,
# last_usable, next_usable, previous_usable, revdns, subnet, and more
```

### Other Useful ansible.utils Filters

```yaml
# ansible.utils.cidr_merge — merge overlapping or adjacent subnets
- ansible.builtin.debug:
    msg: "{{ ['192.168.0.0/24', '192.168.1.0/24'] | ansible.utils.cidr_merge }}"
# → ['192.168.0.0/23']

# ansible.utils.network_in_network — check subnet containment
- ansible.builtin.debug:
    msg: "{{ '10.10.10.0/30' | ansible.utils.network_in_network('10.10.10.0/24') }}"
# → True

# ansible.utils.ip_math — arithmetic on IP addresses
- ansible.builtin.debug:
    msg: "{{ '10.0.0.1' | ansible.utils.ip_math('+', 5) }}"
# → 10.0.0.6

# ansible.utils.validate — validate data against a JSON schema
# Useful for validating host_vars data model before deploying
- ansible.utils.validate:
    data: "{{ interfaces }}"
    criteria:
      - "{{ lookup('file', 'schemas/interfaces_schema.json') }}"
    engine: ansible.utils.jsonschema
```

---

## 31.4 — ansible.netcommon and community.general (Brief)

### ansible.netcommon

The shared foundation that all vendor network collections build on. Playbooks rarely call `ansible.netcommon` modules directly — it's a dependency that provides connection plugins and shared utilities.

```
What it provides (used indirectly):
  Connection plugins: network_cli, netconf, httpapi
    — These are what cisco.ios, junipernetworks.junos, etc. use to connect
  Modules you might call directly:
    ansible.netcommon.cli_command   — send raw CLI commands (platform-agnostic)
    ansible.netcommon.cli_config    — push raw config (platform-agnostic)
    ansible.netcommon.netconf_get   — raw NETCONF get
    ansible.netcommon.net_ping      — network device ping
  Filters:
    ansible.netcommon.parse_cli     — parse CLI output with TextFSM templates
    ansible.netcommon.parse_cli_textfsm — alias for above
    ansible.netcommon.vlan_compress_list — compress VLAN ranges (1,2,3,4 → 1-4)
    ansible.netcommon.vlan_expand_list  — expand VLAN ranges (1-4 → 1,2,3,4)
```

```yaml
# vlan_compress_list and vlan_expand_list are directly useful
vars:
  vlan_list: [10, 11, 12, 13, 20, 30, 31, 32]

- ansible.builtin.debug:
    msg: "{{ vlan_list | ansible.netcommon.vlan_compress_list }}"
# → "10-13,20,30-32"

# Useful when building NX-OS switchport trunk configs:
- cisco.nxos.nxos_config:
    lines:
      - "switchport trunk allowed vlan {{ allowed_vlans | ansible.netcommon.vlan_compress_list }}"
    parents: "interface {{ item }}"
  loop: "{{ trunk_interfaces }}"
```

### community.general

A large catch-all collection maintained by the Ansible community. For network automation, the most useful modules are utilities that run on the control node rather than on devices.

```
Most useful for network automation:
  community.general.json_query    — JMESPath queries on complex data structures
  community.general.ini_file      — manage .ini config files on the control node
  community.general.slack         — send Slack notifications (alternative to uri)
  community.general.mail          — send email notifications
  community.general.filesize      — get file size information
  community.general.git_config    — configure Git settings
  community.general.make          — run Makefiles (useful for custom tooling)
```

```yaml
# json_query for extracting data from complex nested structures
# Requires: pip install jmespath
vars:
  bgp_neighbors:
    - {ip: "10.0.0.1", state: "Established", as: 65001}
    - {ip: "10.0.0.2", state: "Idle", as: 65002}
    - {ip: "10.0.0.3", state: "Established", as: 65003}

- ansible.builtin.debug:
    msg: "{{ bgp_neighbors | community.general.json_query('[?state==`Established`].ip') }}"
# → ["10.0.0.1", "10.0.0.3"]

# Equivalent using Ansible's built-in selectattr (no extra dep needed):
- ansible.builtin.debug:
    msg: "{{ bgp_neighbors | selectattr('state', 'equalto', 'Established') | map(attribute='ip') | list }}"
# → ["10.0.0.1", "10.0.0.3"]
# Use json_query when the query is complex enough that JMESPath is cleaner
```

---

## 31.5 — Writing a Minimum Viable Custom Collection

When the same Jinja2 filter logic is copied across multiple playbooks, it belongs in a custom collection. This example builds `netops.network_filters` — a collection with one filter plugin containing IP/subnet helper functions useful across all the network automation playbooks.

### The Collection Structure

```bash
COLLECTION_DIR="$HOME/projects/ansible-network/collections/netops/network_filters"
mkdir -p "${COLLECTION_DIR}"/{plugins/filter,docs}
cd "${COLLECTION_DIR}"
```

### galaxy.yml — Collection Manifest

```bash
cat > "${COLLECTION_DIR}/galaxy.yml" << 'EOF'
---
namespace: netops
name: network_filters
version: 1.0.0
readme: README.md
description: Custom Jinja2 filters for network automation playbooks
authors:
  - Your Name <you@lab.local>
license:
  - MIT
tags:
  - networking
  - filters
  - ip
dependencies: {}
repository: https://github.com/your-org/netops.network_filters
documentation: https://github.com/your-org/netops.network_filters/blob/main/docs/
homepage: https://github.com/your-org/netops.network_filters
issues: https://github.com/your-org/netops.network_filters/issues
EOF
```

### The Filter Plugin

```bash
cat > "${COLLECTION_DIR}/plugins/filter/network_filters.py" << 'EOF'
# -*- coding: utf-8 -*-
# plugins/filter/network_filters.py
# Custom Jinja2 filters for network automation
#
# Usage in playbooks (after installing the collection):
#   {{ '10.20.30.1/30' | netops.network_filters.peer_ip }}
#   {{ '10.0.0.0/8' | netops.network_filters.subnet_summary }}
#   {{ interfaces | netops.network_filters.interfaces_with_ip }}

from __future__ import absolute_import, division, print_function
__metaclass__ = type

DOCUMENTATION = r"""
  name: network_filters
  short_description: Custom filters for network automation
  description:
    - peer_ip: Calculate the peer IP on a point-to-point /30 or /31 link
    - subnet_summary: Return a human-readable summary of a subnet
    - interfaces_with_ip: Filter a list of interfaces to only those with IPs configured
    - ios_banner_format: Format a banner string for IOS banner commands
    - wildcard_mask: Return the wildcard mask for a given prefix
"""

try:
    import netaddr
    HAS_NETADDR = True
except ImportError:
    HAS_NETADDR = False


def peer_ip(cidr):
    """
    Given one side of a /30 or /31 link as CIDR, return the peer's IP address.

    Usage: {{ '10.10.10.1/30' | netops.network_filters.peer_ip }}
    Returns: '10.10.10.2'

    Works for /30 (usable hosts 1 and 2) and /31 (RFC 3021, hosts 0 and 1).
    Returns the original input unchanged if prefix length is not /30 or /31.
    """
    if not HAS_NETADDR:
        raise ImportError("The netaddr Python library is required. Run: pip install netaddr")

    try:
        network = netaddr.IPNetwork(cidr)
    except netaddr.AddrFormatError as e:
        raise ValueError(f"Invalid IP/CIDR: {cidr}") from e

    prefix_len = network.prefixlen

    if prefix_len == 30:
        # /30: usable hosts are .1 and .2 within the /30
        hosts = list(network.iter_hosts())  # .1 and .2
        if len(hosts) != 2:
            return str(cidr)
        local_ip = network.ip
        # Return the other usable host
        for host in hosts:
            if host != local_ip:
                return str(host)

    elif prefix_len == 31:
        # /31 (RFC 3021): both addresses are usable
        hosts = [network.network, network.broadcast]
        local_ip = network.ip
        for host in hosts:
            if host != local_ip:
                return str(host)
    else:
        # Not a point-to-point link — return original
        return str(cidr)

    return str(cidr)


def subnet_summary(cidr):
    """
    Return a dict with key subnet attributes for a given CIDR prefix.

    Usage: {{ '10.20.0.0/22' | netops.network_filters.subnet_summary }}
    Returns:
      network: '10.20.0.0'
      broadcast: '10.20.3.255'
      netmask: '255.255.252.0'
      wildcard: '0.0.3.255'
      prefix: 22
      size: 1024
      first_usable: '10.20.0.1'
      last_usable: '10.20.3.254'
      usable_hosts: 1022
    """
    if not HAS_NETADDR:
        raise ImportError("The netaddr Python library is required.")

    try:
        network = netaddr.IPNetwork(cidr)
    except netaddr.AddrFormatError as e:
        raise ValueError(f"Invalid CIDR: {cidr}") from e

    hosts = list(network.iter_hosts())
    return {
        'network': str(network.network),
        'broadcast': str(network.broadcast),
        'netmask': str(network.netmask),
        'wildcard': str(network.hostmask),
        'prefix': network.prefixlen,
        'size': network.size,
        'first_usable': str(hosts[0]) if hosts else str(network.network),
        'last_usable': str(hosts[-1]) if hosts else str(network.broadcast),
        'usable_hosts': len(hosts),
    }


def interfaces_with_ip(interface_list):
    """
    Filter a list of interface dicts to only those that have an 'ip' key defined
    and non-empty. Used to skip loopbacks or management interfaces with no IP.

    Usage: {{ interfaces | dict2items | netops.network_filters.interfaces_with_ip }}
    Input: list of {'key': 'GigE1', 'value': {'ip': '10.0.0.1', 'prefix': 30}}
    Output: filtered list with only interfaces that have an IP
    """
    if not isinstance(interface_list, list):
        return interface_list
    return [
        iface for iface in interface_list
        if isinstance(iface, dict) and iface.get('value', {}).get('ip')
    ]


def ios_banner_format(banner_text, delimiter='EOF'):
    """
    Format a banner string for use in IOS banner commands.
    Strips leading/trailing whitespace and ensures lines don't contain the delimiter.

    Usage: {{ banner_text | netops.network_filters.ios_banner_format }}
    Returns: string formatted for 'banner login <delimiter>\\n<text>\\n<delimiter>'
    """
    lines = banner_text.strip().splitlines()
    # Ensure no line contains the delimiter (would break IOS banner parsing)
    safe_lines = [line for line in lines if delimiter not in line]
    return f"{delimiter}\n" + "\n".join(safe_lines) + f"\n{delimiter}"


def wildcard_mask(cidr):
    """
    Return the wildcard (inverse) mask for a given prefix or CIDR.

    Usage: {{ '192.168.1.0/24' | netops.network_filters.wildcard_mask }}
    Returns: '0.0.0.255'

    Useful for IOS ACL entries: 'permit ip 192.168.1.0 0.0.0.255 any'
    """
    if not HAS_NETADDR:
        raise ImportError("The netaddr Python library is required.")

    try:
        network = netaddr.IPNetwork(cidr)
        return str(network.hostmask)
    except netaddr.AddrFormatError as e:
        raise ValueError(f"Invalid CIDR: {cidr}") from e


class FilterModule(object):
    """Register filters with Ansible."""

    def filters(self):
        return {
            'peer_ip': peer_ip,
            'subnet_summary': subnet_summary,
            'interfaces_with_ip': interfaces_with_ip,
            'ios_banner_format': ios_banner_format,
            'wildcard_mask': wildcard_mask,
        }
EOF
```

### README

```bash
cat > "${COLLECTION_DIR}/README.md" << 'EOF'
# netops.network_filters

Custom Jinja2 filter plugins for network automation playbooks.

## Installation

```bash
# Local development install
ansible-galaxy collection install ./collections/netops/network_filters/

# From tarball (after building)
ansible-galaxy collection build
ansible-galaxy collection install netops-network_filters-1.0.0.tar.gz
```

## Requirements

```bash
pip install netaddr
```

## Filters

| Filter | Input | Output | Use case |
|---|---|---|---|
| `peer_ip` | `'10.0.0.1/30'` | `'10.0.0.2'` | BGP neighbor from interface IP |
| `subnet_summary` | `'10.0.0.0/24'` | dict with network/broadcast/etc | Subnet documentation |
| `interfaces_with_ip` | list of interface dicts | filtered list | Skip unconfigured interfaces |
| `ios_banner_format` | banner string | IOS-formatted string | Banner config generation |
| `wildcard_mask` | `'192.168.1.0/24'` | `'0.0.0.255'` | IOS ACL entry generation |

## Usage Examples

```yaml
# Calculate BGP peer IP from interface CIDR
- ansible.builtin.set_fact:
    bgp_peer_ip: "{{ 'GigabitEthernet1' | vars | netops.network_filters.peer_ip }}"

# Use in an IOS ACL task
- cisco.ios.ios_config:
    lines:
      - "permit ip {{ item.network }} {{ item.network | netops.network_filters.wildcard_mask }} any"
    parents: "ip access-list extended {{ acl_name }}"
  loop: "{{ allowed_networks }}"
```
EOF
```

### Installing and Testing the Custom Collection

```bash
# Install locally from the directory (development mode)
ansible-galaxy collection install \
  ~/projects/ansible-network/collections/netops/network_filters/ \
  --force-with-deps

# Verify it's installed
ansible-galaxy collection list | grep netops

# Test filters directly in an ad-hoc debug playbook
cat > /tmp/test_filters.yml << 'EOF'
---
- hosts: localhost
  gather_facts: false
  tasks:
    - ansible.builtin.debug:
        msg:
          - "peer_ip:        {{ '10.10.10.1/30' | netops.network_filters.peer_ip }}"
          - "wildcard_mask:  {{ '192.168.100.0/23' | netops.network_filters.wildcard_mask }}"
          - "subnet_summary: {{ '10.0.0.0/27' | netops.network_filters.subnet_summary }}"
EOF

ansible-playbook /tmp/test_filters.yml

# Expected output:
# peer_ip:        10.10.10.2
# wildcard_mask:  0.0.1.255
# subnet_summary: {'network': '10.0.0.0', 'broadcast': '10.0.0.31',
#                  'netmask': '255.255.255.224', 'wildcard': '0.0.0.31',
#                  'prefix': 27, 'size': 32, 'first_usable': '10.0.0.1',
#                  'last_usable': '10.0.0.30', 'usable_hosts': 30}
```

### Building and Distributing the Collection

```bash
# Build a distributable tarball
cd ~/projects/ansible-network/collections/netops/network_filters/
ansible-galaxy collection build
# Creates: netops-network_filters-1.0.0.tar.gz

# Install from the tarball (what teammates do)
ansible-galaxy collection install netops-network_filters-1.0.0.tar.gz

# Publish to Ansible Galaxy (requires Galaxy API token)
# ansible-galaxy collection publish netops-network_filters-1.0.0.tar.gz \
#   --api-key <your-galaxy-token>

# For internal use without Galaxy: host the tarball on an internal server
# or commit it to a Git repo and install via URL in requirements.yml
```

---

## 31.6 — Using the Custom Filters in Real Playbooks

The custom filters integrate naturally into the existing playbooks. Here's how `peer_ip` and `wildcard_mask` clean up the IOS deployment playbook:

```yaml
# Before — manual peer IP calculation embedded in template
- cisco.ios.ios_config:
    parents: "router bgp {{ bgp.as_number }}"
    lines:
      - "neighbor {{ item.peer_ip }} remote-as {{ item.remote_as }}"
  loop: "{{ bgp.neighbors }}"

# After — derive peer IP from interface data using the custom filter
# No need to maintain a separate peer_ip field in the data model
- cisco.ios.ios_config:
    parents: "router bgp {{ bgp.as_number }}"
    lines:
      - "neighbor {{ interfaces[item.via_interface].ip + '/' + interfaces[item.via_interface].prefix | string | netops.network_filters.peer_ip }} remote-as {{ item.remote_as }}"
  loop: "{{ bgp.neighbors }}"
```

```yaml
# Before — wildcard mask calculated in Jinja2 inline (verbose and error-prone)
- cisco.ios.ios_config:
    lines:
      - "permit ip {{ item | ansible.utils.ipaddr('network') }} \
         {{ item | ansible.utils.ipaddr('wildcard') }} any"

# After — clean filter name makes intent clear
- cisco.ios.ios_config:
    lines:
      - "permit ip {{ item | ansible.utils.ipaddr('network') }} \
         {{ item | netops.network_filters.wildcard_mask }} any"
```

---


Collections are complete — the architecture and namespace model are clear, `ansible.utils.ipaddr` is in active use for IP manipulation, and the project now has its own custom collection with five reusable filter plugins that reduce duplication across playbooks. The guide has now covered every layer of Ansible network automation: connection, inventory, playbooks, roles, collections, scheduling, and enterprise management.


