---
draft: false
title: '5 - Ansible'
description: "Part 5 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 5
---

{{< badge "Ansible" >}}
{{< badge content="Linux" color="red" >}}

---

{{< lab-callout type="info" >}}
This is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. Each part will build upon the last. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.
{{< /lab-callout >}}

---

## Installing Ansible

This part isn't about reinstalling it, it's about understanding what I actually installed, why it's structured the way it is, and what's happening behind the scenes when a playbook runs. That understanding is what separates someone who can copy-paste playbooks from someone who can write, debug, and maintain them confidently.

---

## Installation and Verification

Ansible was installed in Part 3 inside the `ansible-network` virtual environment. Before going any further, I confirm everything is in place and pointing to the right locations.

---

{{< subtle-label >}}Activate the Environment and Verify{{< /subtle-label >}}

Activate the virtualenv

{{< codeblock lang="Bash" syntax="bash" >}}
source ~/venvs/ansible-network/bin/activate
{{< /codeblock >}}

Confirm the prompt changed

{{< codeblock lang="" copy="false" >}}
# (ansible-network) ansible@ubuntu:~$
{{< /codeblock >}}

Verify Ansible is installed and check the version

{{< codeblock lang="Bash" syntax="bash" >}}
ansible --version
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
ansible [core 2.17.x]
  config file = None
  configured module search path = ['/home/ansible/.ansible/plugins/modules', ...]
  ansible python module location = /home/ansible/venvs/ansible-network/lib/python3.10/site-packages/ansible
  ansible collection location = /home/ansible/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/ansible/venvs/ansible-network/bin/ansible
  python version = 3.10.12 (main, ...) [GCC 11.4.0]
  jinja version = 3.1.x
  libyaml = True
{{< /codeblock >}}

**The three lines I always check:**

- `ansible python module location` - must point into my virtualenv (`venvs/ansible-network/...`), not `/usr/lib/...`
- `executable location` - must point into my virtualenv (`venvs/ansible-network/bin/ansible`), not `/usr/bin/ansible`
- `libyaml = True` - confirms the fast C-based YAML parser is available. If this shows `False`, YAML parsing will be significantly slower. Fix: `sudo apt install -y python3-yaml`

---

{{< subtle-label >}}Verify All Supporting Tools{{< /subtle-label >}}

ansible-lint:

{{< codeblock lang="Bash" syntax="bash" >}}
# ansible-lint 24.x.x using ansible-core:2.17.x ...
{{< /codeblock >}}

yamllint:

{{< codeblock lang="Bash" syntax="bash" >}}
yamllint --version
# yamllint 1.35.x
{{< /codeblock >}}

ansible-navigator:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-navigator --version
# ansible-navigator 24.x.x
{{< /codeblock >}}

Check installed collections

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection list
{{< /codeblock >}}

---

## ansible vs ansible-core vs Collections

When I ran `pip install ansible` in Part 3, I didn't just install one thing. I need to understand what I actually have, because the distinction matters when reading documentation, troubleshooting errors, and managing upgrades.

{{< subtle-label >}}The 3 Layers{{< /subtle-label >}}

{{< topology1 src="diagrams/threelayers.svg" >}}

---

{{< subtle-label >}}Ansible-Core{{< /subtle-label >}}

This is the engine, the actual Ansible runtime. It includes:

- The `ansible-playbook` command
- The `ansible` ad-hoc command
- The `ansible-inventory` command
- The `ansible-vault` command
- The `ansible-galaxy` command
- The built-in modules under the `ansible.builtin` namespace (`copy`, `template`, `file`, `debug`, `assert`, etc.)
- The variable system, Jinja2 integration, and connection plugins

`ansible-core` alone is a minimal install. It can run playbooks but has very few modules beyond the basics.

---

{{< subtle-label >}}Ansible (The Full Package){{< /subtle-label >}}

`pip install ansible` installs `ansible-core` plus a curated set of ~85 community collections. This is what most people mean when they say "install Ansible."

To see what version of ansible-core is inside my ansible install:

{{< codeblock lang="Bash" syntax="bash" >}}
python3 -c "import ansible; print(ansible.__version__)"

# ansible and ansible-core version numbers differ
# ansible 9.x.x contains ansible-core 2.16.x
# ansible 10.x.x contains ansible-core 2.17.x
{{< /codeblock >}}

---

{{< subtle-label >}}Collections{{< /subtle-label >}}

Collections are the packaging format for Ansible content like modules, roles, plugins, and documentation bundled together by namespace and name (`cisco.ios`, `ansible.netcommon`, etc.).

There are two types:
- **Bundled collections** - come with `pip install ansible`, no separate install needed
- **External collections** - installed separately with `ansible-galaxy collection install`

The network vendor collections (`cisco.ios`, `cisco.nxos`, etc.) are external, they're maintained by the vendors themselves and updated independently of Ansible's release cycle.

---

{{< subtle-label >}}When to Use each{{< /subtle-label >}}

In enterprise environments with strict package management, some teams prefer:

{{< codeblock lang="Bash" syntax="bash" >}}
pip install ansible-core
ansible-galaxy collection install cisco.ios cisco.nxos junipernetworks.junos paloaltonetworks.panos ansible.netcommon ansible.utils
{{< /codeblock >}}

This gives complete control over exactly which collection versions are installed and avoids pulling in collections that aren't needed. The tradeoff is more explicit management. For this guide, the full `ansible` package is fine.

---

## The Ansible Architecture

Understanding this diagram saves hours of debugging. When something goes wrong, knowing which component is responsible tells me where to look.

---

{{< subtle-label >}}The Big Picture{{< /subtle-label >}}

{{< topology1 src="diagrams/bigpicture.svg" >}}

---

{{< subtle-label >}}Key Components{{< /subtle-label >}}

**Control Node**:
The Ubuntu VM where Ansible is installed and where I run `ansible-playbook`. All the intelligence lives here, the devices themselves run nothing special.

**Inventory**:
The file (or files) that tell Ansible what devices exist, how to reach them, and how to group them. This is where `ansible_host`, `ansible_user`, `ansible_network_os`, and similar connection variables are defined. Without an inventory, Ansible has nothing to connect to.

**Playbook**:
A YAML file that describes what I want to happen: which hosts to target, in what order, using which modules, with which parameters. The playbook is the "what" and the inventory is the "where."

**`ansible.cfg`**:
The configuration file that controls Ansible's behavior: where to find the inventory, which connection timeout to use, whether to check SSH host keys, and dozens of other settings. Ansible searches for this file in the current directory first, then `~/.ansible.cfg`, then `/etc/ansible/ansible.cfg`.

**Modules**:
The individual units of work e.g. `ios_config`, `ios_facts`, `nxos_vlans`, `debug`, `template`, etc. Each module is a Python script that performs one specific task and returns a result dictionary with at minimum the keys `changed` and `failed`.

**Plugins**:
Extend Ansible's core functionality. The most relevant for network work:
- **Connection plugins** - define how Ansible connects to devices (`network_cli`, `netconf`, `httpapi`)
- **Callback plugins** - control how output is formatted and displayed
- **Filter plugins** - extend Jinja2 with extra functions (`ipaddr`, `selectattr`, `combine`)
- **Lookup plugins** - pull data from external sources into variables

{{< lab-callout type="tip" >}}
The single most important thing to understand about this architecture is that the control node does all the work. My Ubuntu VM is where Python runs, where SSH sessions are opened, where Jinja2 templates are rendered, and where results are processed. The network devices just receive CLI commands over SSH and send back text output. This is fundamentally different from how Ansible manages Linux servers (where it uploads and runs Python on the remote machine). Understanding this distinction explains why network playbooks use `connection: network_cli` and `gather_facts: false` (there's no Python on a Cisco router to run facts gathering).
{{< /lab-callout >}}

---

## How Ansible Connects to Network Devices

{{< subtle-label >}}Network CLI Connection Plugin{{< /subtle-label >}}

For Cisco IOS, NX-OS, Juniper, and Palo Alto CLI automation, I use `connection: network_cli`. This is the standard for SSH-based network device automation.

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: Configure IOS devices
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli

  tasks:
    - name: Show version
      cisco.ios.ios_command:
        commands: show version
{{< /codeblock >}}

When Ansible sees `connection: network_cli`, here's what happens:

1. Ansible loads the `network_cli` connection plugin
2. The connection plugin uses Paramiko (by default) to open an SSH connection to the device
3. The plugin handles the device's CLI prompts like login banners, `#` prompts, `--More--` pagination, config mode prompts
4. The plugin sends CLI commands and reads back the output
5. The module parses that output and builds the result dictionary
6. The SSH session is kept alive for all tasks in the play (not reopened per task)

---

{{< subtle-label >}}Paramiko{{< /subtle-label >}}

Paramiko is a pure-Python SSH implementation. It's the default SSH library for network device connections in Ansible because:

- It handles vendor-specific SSH quirks (different banners, prompts, timing)
- It's controllable from Python code. Modules can set per-command timeouts, handle `--More--` prompts, etc.
- It works even when OpenSSH's strict key exchange requirements conflict with older network device SSH implementations


Confirm paramiko is installed in the virtualenv

{{< codeblock lang="Bash" syntax="bash" >}}
python3 -c "import paramiko; print(f'Paramiko version: {paramiko.__version__}')"
{{< /codeblock >}}

---

{{< subtle-label >}}Paramiko vs OpenSSH{{< /subtle-label >}}

| Feature | Paramiko | OpenSSH |
|---|---|---|
| Where it runs | Pure Python, inside Ansible | System binary (`/usr/bin/ssh`) |
| Default for network devices |  Yes |  No |
| Handles old device SSH ciphers |  Better | Requires manual cipher config |
| Performance (many hosts) | Slightly slower | Faster |
| ControlMaster (multiplexing) |  No |  Yes |
| Debugging | Python exceptions | `-vvv` SSH debug output |

To switch a specific host or group from Paramiko to OpenSSH:

{{< codeblock lang="YAML" syntax="yaml" >}}
ansible_connection: network_cli
ansible_network_cli_ssh_type: openssh
{{< /codeblock >}}

I only do this if I have a specific reason since Paramiko is the right default for network devices.

---

{{< subtle-label >}}ansible_network_os{{< /subtle-label >}}

This variable is mandatory for `network_cli` connections. It tells Ansible which terminal handler to use like how to detect the prompt, how to enter config mode, how to handle pagination.

{{< codeblock lang="YAML" syntax="yaml" >}}
# group_vars/cisco_ios.yml
ansible_network_os: cisco.ios.ios

# group_vars/cisco_nxos.yml
ansible_network_os: cisco.nxos.nxos

# group_vars/juniper.yml
ansible_network_os: junipernetworks.junos.junos

# group_vars/paloalto.yml
ansible_network_os: paloaltonetworks.panos.panos
{{< /codeblock >}}

The format is `<collection_namespace>.<collection_name>.<platform>`.

---

{{< subtle-label >}}Netconf{{< /subtle-label >}}

NETCONF is an XML-based network management protocol supported on modern IOS-XE and Juniper devices. It's more structured than CLI parsing and is the future direction for network automation but it requires the device to have NETCONF enabled.

On IOS-XE, enable NETCONF

{{< codeblock lang="" copy="false" >}}
# netconf-yang
{{< /codeblock >}}

Playbook using NETCONF connection:

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: Configure IOS-XE via NETCONF
  hosts: iosxe_devices
  gather_facts: false
  connection: netconf

  tasks:
    - name: Get device facts via NETCONF
      ansible.netcommon.netconf_get:
        filter: <interfaces xmlns="..."/>
{{< /codeblock >}}

For this lab, I focus on `network_cli` since it works with all four platforms (IOS, NX-OS, Juniper, Palo Alto) without requiring any device-side configuration changes. NETCONF becomes relevant in advanced IOS-XE and Juniper automation and is covered where appropriate in later parts.

---

## Agentless Automation

Traditional monitoring and management tools require an agent; a piece of software running on the managed device that communicates with a central server. Network devices can't run agents. They run proprietary operating systems with no mechanism to install third-party software.

Ansible is agentless by design. Here's what that means in practice:

{{< codeblock lang="" copy="false" >}}
Control Server ←──── Agent (running on device) ←── Device

Ansible (Agentless):
Control Node ──SSH──▶ Device CLI ──▶ Returns output ──▶ Control Node processes it
{{< /codeblock >}}

Ansible's only requirement on managed network devices:
- SSH enabled (`ip ssh version 2`)
- A user account with appropriate privilege level

No Python. No agent. No daemon. No TCP port other than 22 (or whatever SSH port is configured). This is why Ansible works on Cisco IOS devices from 2005 just as well as IOS-XE devices from 2024 because if it has SSH, Ansible can automate it.

---

{{< subtle-label >}}The Implication for gather_facts{{< /subtle-label >}}

In server automation, Ansible's first task is always to gather facts. It runs a Python script on the remote host to collect OS version, IP addresses, disk space, etc. On network devices, there's no Python to run, so I disable this:

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: Configure routers
  hosts: cisco_ios
  gather_facts: false 
{{< /codeblock >}}

- `gather_facts: false` - Always false for network devices.

---

## `ansible.cfg` Configuration File

`ansible.cfg` controls Ansible's behavior project-wide. Having a good `ansible.cfg` means I don't have to pass dozens of flags on every command.

---

{{< subtle-label >}}Where Ansible Looks for ansible.cfg{{< /subtle-label >}}

Ansible checks these locations in order, using the first one it finds:

{{< codeblock lang="" copy="false" >}}
1. $ANSIBLE_CONFIG environment variable (if set)
2. ./ansible.cfg (current working directory) ← I always use this
3. ~/.ansible.cfg (home directory)
4. /etc/ansible/ansible.cfg (system-wide)
{{< /codeblock >}}

I always put `ansible.cfg` in the project root e.g. `~/projects/ansible-network/ansible.cfg`. This means the config is project-specific, version-controlled with Git, and automatically used whenever I `cd` into the project directory.

---

{{< subtle-label >}}Creating the Project ansible.cfg{{< /subtle-label >}}

{{< codeblock lang="Bash" syntax="bash" >}}
nano ~/projects/ansible-network/ansible.cfg
{{< /codeblock >}}

{{< codeblock file="ansible.cfg" syntax="ini" >}}
[defaults]
# Inventory file location
inventory = inventory/hosts.yml

# Default remote user for SSH connections
remote_user = ansible

# Disable host key checking for lab environments
# CHANGE TO True IN PRODUCTION
host_key_checking = False

# SSH connection timeout in seconds
timeout = 30

# Number of parallel connections (forks)
forks = 10

# Where to look for roles
roles_path = roles/

# Where to look for collections
collections_path = collections/

# Retry files — where to write them (or disable entirely)
retry_files_enabled = False

# Stdout callback — prettier output
stdout_callback = yaml

# Don't show warnings about deprecations (set to True when debugging)
deprecation_warnings = False

# Default interpreter for Python (uses the virtualenv's Python)
interpreter_python = auto_silent


[inventory]
# Enable inventory plugins
enable_plugins = yaml, ini, auto


[ssh_connection]
# Use Paramiko as the default SSH transport for network devices
# (This is the default for network_cli but explicit is better)
ssh_args = -o ControlMaster=no -o ControlPersist=no

# Pipelining speeds up Linux host connections (leave False for network devices)
pipelining = False


[persistent_connection]
# How long to keep a persistent SSH connection alive between tasks (seconds)
connect_timeout = 30

# How long to wait for a command to complete (seconds)
command_timeout = 30
{{< /codeblock >}}

{{< line-explain >}}
host_key_checking = False:
: disables SSH host key verification. Fine for a lab where I'm spinning up and tearing down Containerlab topologies constantly (new devices get new keys). Must be `True` in production.

forks = 10:
: run tasks against 10 devices simultaneously. The default is 5. Increase for larger inventories.

stdout_callback = yaml:
: formats task output as clean YAML instead of the default ugly single-line format. Much easier to read.

retry_files_enabled = False:
: disables `.retry` files (which are in `.gitignore` anyway). I find them more annoying than useful.

interpreter_python = auto_silent:
: silences the Python interpreter warning that Ansible sometimes shows.
{{< /line-explain >}}

- `connect_timeout` and `command_timeout` in `[persistent_connection]` - critical for network devices. If a device is slow to respond (common on older hardware or WAN links), the default timeouts cause false failures.

---

{{< subtle-label >}}Checking the Active Configuration{{< /subtle-label >}}

Dump the entire Ansible configuration, showing which file each setting comes from

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-config dump
{{< /codeblock >}}

Show only settings that differ from the defaults

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-config dump --only-changed
{{< /codeblock >}}

Show the value of a specific setting

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-config dump | grep HOST_KEY_CHECKING
{{< /codeblock >}}

`ansible-config dump` is my first stop when Ansible behaves unexpectedly since it shows exactly what configuration is in effect and where each setting came from.

{{< lab-callout type="tip" >}}
`ansible-config dump --only-changed` is especially useful when inheriting an existing Ansible environment. It shows exactly which defaults have been overridden and where.
{{< /lab-callout >}}

---

## Ansible Galaxy and Collections

Ansible Galaxy (`galaxy.ansible.com`) is the public hub for Ansible collections and roles. It's where vendor-maintained collections like `cisco.ios` live, and where I install them from.

---

{{< subtle-label >}}ansible-galaxy{{< /subtle-label >}}

Install a single collection

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install cisco.ios
{{< /codeblock >}}

Install a specific version

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install cisco.ios:==8.0.0
{{< /codeblock >}}

Install a minimum version

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install cisco.ios:>=8.0.0
{{< /codeblock >}}

Upgrade an existing collection to the latest version

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install cisco.ios --upgrade
{{< /codeblock >}}

Install from the collections/requirements.yml file

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install -r collections/requirements.yml
{{< /codeblock >}}

List all installed collections and their versions

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection list
{{< /codeblock >}}

---

{{< subtle-label >}}Where Collections Are Installed{{< /subtle-label >}}

By default, collections install to `~/.ansible/collections/ansible_collections/`:

{{< codeblock lang="" copy="false" >}}
~/.ansible/collections/
└── ansible_collections/
    ├── cisco/
    │   ├── ios/
    │   │   ├── plugins/modules/     #ios_config.py, ios_facts.py, etc.
    │   │   ├── roles/
    │   │   └── README.md
    │   └── nxos/
    ├── junipernetworks/
    │   └── junos/
    ├── paloaltonetworks/
    │   └── panos/
    └── ansible/
        ├── netcommon/
        └── utils/
{{< /codeblock >}}

I can also install collections inside the project directory (useful for CI/CD):

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install cisco.ios -p ./collections/
{{< /codeblock >}}

This installs into `./collections/ansible_collections/` relative to the project. Combined with `collections_path = collections/` in `ansible.cfg`, Ansible finds them automatically.

---

### Collections

The 6 collections I use throughout this project.

{{< subtle-label >}}cisco.ios{{< /subtle-label >}}

The official Cisco collection for IOS and IOS-XE devices. Contains modules for every aspect of IOS configuration.

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install cisco.ios
{{< /codeblock >}}

Key modules I'll use constantly:

- `cisco.ios.ios_facts` - Gather device facts (version, interfaces, routing)
- `cisco.ios.ios_command` - Run show commands and capture output
- `cisco.ios.ios_config` - Push configuration lines (the workhorse module)
- `cisco.ios.ios_vlans` - Manage VLANs declaratively
- `cisco.ios.ios_interfaces` - Manage interface settings
- `cisco.ios.ios_l2_interfaces` - Manage Layer 2 interface settings (switchport)
- `cisco.ios.ios_l3_interfaces` - Manage Layer 3 interface settings (IP addressing)
- `cisco.ios.ios_bgp_global` - Manage BGP global configuration
- `cisco.ios.ios_ospfv2` - Manage OSPFv2 configuration
- `cisco.ios.ios_acls` - Manage access control lists
- `cisco.ios.ios_static_routes` - Manage static routes
- `cisco.ios.ios_users` - Manage local user accounts
- `cisco.ios.ios_banner` - Manage login and MOTD banners

---

{{< subtle-label >}}cisco.nxos{{< /subtle-label >}}

The official Cisco collection for Nexus NX-OS devices.

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install cisco.nxos
{{< /codeblock >}}

Key modules:

- `cisco.nxos.nxos_facts` - Gather NX-OS device facts
- `cisco.nxos.nxos_command` - Run NX-OS show commands
- `cisco.nxos.nxos_config` - Push NX-OS configuration
- `cisco.nxos.nxos_vlans` - Manage VLANs on NX-OS
- `cisco.nxos.nxos_interfaces` - Manage NX-OS interfaces
- `cisco.nxos.nxos_bgp_global` - Manage NX-OS BGP
- `cisco.nxos.nxos_feature` - Enable/disable NX-OS features (bgp, ospf, vpc, etc.)
- `cisco.nxos.nxos_vrf` - Manage VRFs on NX-OS
- `cisco.nxos.nxos_vpc` - Manage Virtual Port Channel

---

{{< subtle-label >}}junipernetworks.junos{{< /subtle-label >}}

The official Juniper Networks collection for Junos OS.

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install junipernetworks.junos
{{< /codeblock >}}

Key modules:

- `junipernetworks.junos.junos_facts` - Gather Junos facts
- `junipernetworks.junos.junos_command` - Run Junos operational commands
- `junipernetworks.junos.junos_config` - Push Junos configuration (set or XML format)
- `junipernetworks.junos.junos_interfaces` - Manage Junos interfaces
- `junipernetworks.junos.junos_vlans` - Manage Junos VLANs

Junos has a fundamentally different configuration model than Cisco. Junos uses a candidate configuration which means changes are staged but not active until a `commit` is issued. Ansible's `junos_config` module handles this automatically: it stages changes, then commits. If the commit fails, it automatically rolls back. This is more reliable than Cisco's IOS model where `ios_config` applies changes line by line with no built-in rollback.

---

{{< subtle-label >}}paloaltonetworks.panos{{< /subtle-label >}}

The official Palo Alto Networks collection for PAN-OS firewalls.

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install paloaltonetworks.panos
{{< /codeblock >}}

Key modules:

- `paloaltonetworks.panos.panos_facts` - Gather PAN-OS facts
- `paloaltonetworks.panos.panos_address_object` - Manage address objects
- `paloaltonetworks.panos.panos_security_rule` - Manage security policies
- `paloaltonetworks.panos.panos_nat_rule` - Manage NAT rules
- `paloaltonetworks.panos.panos_interface` - Manage interfaces
- `paloaltonetworks.panos.panos_zone` - Manage security zones
- `paloaltonetworks.panos.panos_commit` - Commit configuration changes

PAN-OS automation works differently from CLI-based devices. Palo Alto firewalls expose an XML API, and the `panos` collection communicates via that API rather than SSH CLI. This means the connection type for PAN-OS playbooks is `httpapi`, not `network_cli`. Additionally, PAN-OS requires an explicit `panos_commit` task at the end which changes staged via API are not active until committed.

---

{{< subtle-label >}}ansible.netcommon{{< /subtle-label >}}

This collection provides shared networking modules and plugins used across all vendor collections.

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install ansible.netcommon
{{< /codeblock >}}

Key modules and plugins:

- `ansible.netcommon.net_ping` - Vendor-neutral ping test
- `ansible.netcommon.netconf_get` - NETCONF get operations
- `ansible.netcommon.netconf_config` - NETCONF config operations
- `ansible.netcommon.cli_command` - Send raw CLI commands (vendor-neutral)
- `ansible.netcommon.cli_config` - Send raw CLI config (vendor-neutral)

Connection plugins (used automatically, not called directly)

- `network_cli` - SSH-based CLI connection for all vendors
- `netconf` - NETCONF protocol connection
- `httpapi` - HTTP/HTTPS API connection (used by PAN-OS, some IOS-XE)

---

{{< subtle-label >}}ansible.utils{{< /subtle-label >}}

This collection provides utility modules and Jinja2 filters that are helpful for network automation (particularly for working with IP addresses and validating data).

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install ansible.utils
{{< /codeblock >}}

Key features:

- `ansible.utils.validate` - Validate data against a schema
- `ansible.utils.fact_diff` - Compare two sets of facts

Jinja2 filters (used inside {{ }} expressions in playbooks and templates)

- `ansible.utils.ipaddr` - IP address manipulation
- `ansible.utils.ipsubnet` - Subnet calculations
- `ansible.utils.ipv4` - Filter for IPv4 addresses only
- `ansible.utils.ipv6` - Filter for IPv6 addresses only
- `ansible.utils.network_in_usable` - Check if IP is usable in a subnet

Example of `ansible.utils` filters in a playbook:

{{< codeblock lang="YAML" syntax="yaml" >}}
vars:
  mgmt_network: "192.168.1.0/24"

tasks:
  - name: Show network information
    ansible.builtin.debug:
      msg:
        - "Network:   {{ mgmt_network | ansible.utils.ipaddr('network') }}"
        - "Broadcast: {{ mgmt_network | ansible.utils.ipaddr('broadcast') }}"
        - "First host:{{ mgmt_network | ansible.utils.ipaddr('1') }}"
        - "Last host: {{ mgmt_network | ansible.utils.ipaddr('-2') }}"
        - "Prefix:    {{ mgmt_network | ansible.utils.ipaddr('prefix') }}"
{{< /codeblock >}}

{{< codeblock lang="Output" copy="false" >}}
Network:    192.168.1.0
Broadcast:  192.168.1.255
First host: 192.168.1.1
Last host:  192.168.1.254
Prefix:     24
{{< /codeblock >}}

The `ansible.utils.ipaddr` filter is one of the most useful tools in network automation. Instead of hardcoding broadcast addresses, network addresses, and first/last usable IPs in variable files, I derive them dynamically from a single subnet definition. This reduces errors and makes templates work correctly across any subnet size without manual calculation.

---

{{< subtle-label >}}Keeping Collections Updated{{< /subtle-label >}}

Check what's installed vs what's latest

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection list
{{< /codeblock >}}

Upgrade all installed collections

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install -r collections/requirements.yml --upgrade
{{< /codeblock >}}

Upgrade a specific collection

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install cisco.ios --upgrade
{{< /codeblock >}}

{{< lab-callout type="warning" >}}
Collection updates can introduce breaking changes. Cisco and Juniper occasionally deprecate module parameters or change module behavior between minor versions. Before upgrading collections in a production environment, I test the upgrade in Containerlab first. If `collections/requirements.yml` uses pinned versions (`version: ==8.0.0`), collections never update without my explicit approval.
{{< /lab-callout >}}

---

## `ansible-lint` and `yamllint`

These two tools are my first line of defense against broken playbooks. I run them before committing any YAML file.

---

{{< subtle-label >}}YAML Syntax and Style Checker{{< /subtle-label >}}

`yamllint` checks for YAML syntax errors and style violations. It catches issues before Ansible even sees the file.

Lint a single file

{{< codeblock lang="Bash" syntax="bash" >}}
yamllint playbooks/site.yml
{{< /codeblock >}}

Lint the entire project

{{< codeblock lang="Bash" syntax="bash" >}}
yamllint .
{{< /codeblock >}}

Lint with a specific config

{{< codeblock lang="Bash" syntax="bash" >}}
yamllint -c .yamllint playbooks/
{{< /codeblock >}}

I create a `.yamllint` config file in the project root to configure the rules:

{{< codeblock file="~/projects/ansible-network/.yamllint" syntax="yaml" >}}
---
extends: default

rules:
  line-length:
    max: 120
    level: warning
  truthy:
    allowed-values: ['true', 'false', 'yes', 'no']
    check-keys: false
  comments:
    min-spaces-from-content: 1
  comments-indentation: disable
{{< /codeblock >}}

{{< line-explain >}}
max: 120:
: Allow longer lines than the 80-char default

level: warning:
: Warn rather than error on long lines
{{< /line-explain >}}

---

{{< subtle-label >}}ansible-lint Best Practice{{< /subtle-label >}}

`ansible-lint` goes further than `yamllint` because it understands Ansible-specific syntax and checks for:

- Using deprecated module names or parameters
- Tasks missing a `name:` field
- Using `command` or `shell` when a proper module exists
- Hardcoded passwords in playbooks
- Incorrect YAML formatting for Ansible files
- Risky file permission settings

---

Lint a single playbook

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-lint playbooks/site.yml
{{< /codeblock >}}

Lint the entire project

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-lint
{{< /codeblock >}}

Show all available rules

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-lint --list-rules
{{< /codeblock >}}

Skip specific rules

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-lint --skip-list yaml[line-length]
{{< /codeblock >}}

I create an `.ansible-lint` config file in the project root:

{{< codeblock file="~/projects/ansible-network/.ansible-lint" syntax="yaml" >}}
---
profile: moderate  

# Rules to skip
skip_list:
  - yaml[line-length]

# Warn but don't fail on these rules
warn_list:
  - experimental

# Exclude paths from linting
exclude_paths:
  - collections/
  - .git/
  - venvs/
{{< /codeblock >}}

{{< line-explain >}}
profile: moderate:
: Bash, moderate, safety, shared, production
{{< /line-explain >}}

---

## Ansible-Navigator

`ansible-navigator` is a newer tool that provides a text-based UI (TUI) for running and inspecting Ansible playbooks. It's especially useful for reviewing the full output of a complex playbook run interactively.

---

Run a playbook with ansible-navigator

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-navigator run playbooks/site.yml
{{< /codeblock >}}

Review a previously saved playbook run

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-navigator replay playbook-artifact.json
{{< /codeblock >}}

Browse available collections interactively

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-navigator collections
{{< /codeblock >}}

Browse available modules

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-navigator doc cisco.ios.ios_config
{{< /codeblock >}}

---

The TUI lets me:
- Drill into individual task results
- See the full diff of what changed on each device
- Navigate through plays and tasks with arrow keys
- Search through output

For quick ad-hoc work, I still use `ansible-playbook`. For complex multi-device playbook runs where I want to inspect results interactively, `ansible-navigator` is better.

---

Ansible is installed, configured, and verified. I understand the architecture. What the control node does, how it connects to devices via Paramiko over network_cli, and why the devices themselves need nothing installed.


