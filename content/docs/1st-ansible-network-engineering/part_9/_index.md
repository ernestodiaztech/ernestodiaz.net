---
draft: false
title: '9 - Adhoc'
description: "Part 9 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 9
---

{{< badge "Ansible" >}}
{{< badge content="Linux" color="red" >}}

---

{{< lab-callout type="info" >}}
This is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. Each part will build upon the last. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.
{{< /lab-callout >}}

---

## Ad-Hoc Commands

An ad-hoc command is a single Ansible module call executed directly from the command line. It's used for quick, one-time operations that don't need to be repeatable.

---

{{< subtle-label >}}Model{{< /subtle-label >}}



---

{{< subtle-label >}}Command Syntax{{< /subtle-label >}}

{{< codeblock lang="" copy="false" >}}
ansible <pattern> \
    -m <module_name> \
    -a "<module_arguments>" \
    -i <inventory_path> \
    [options]
{{< /codeblock >}}

**Flag Explanation:**

| Flag | Long form | Purpose |
|---|---|---|
| `-m` | `--module-name` | The Ansible module to run |
| `-a` | `--args` | Arguments to pass to the module |
| `-i` | `--inventory` | Path to inventory file or directory |
| `-l` | `--limit` | Limit to specific hosts or groups |
| `-u` | `--user` | Override the SSH username |
| `-k` | `--ask-pass` | Prompt for SSH password |
| `-b` | `--become` | Enable privilege escalation (enable mode) |
| `-K` | `--ask-become-pass` | Prompt for become password |
| `-v` | `--verbose` | Verbose output (stack for more: -vv, -vvv) |
| `-f` | `--forks` | Number of parallel connections |
| `-o` | `--one-line` | Compact one-line output per host |
| `-t` | `--tree` | Write output to a directory (one file per host) |
| `--check` | | Dry run |
| `--diff` | | Show before/after diff of changes |
| `--list-hosts` | | Show which hosts match the pattern, don't run |

---

{{< subtle-label >}}Pattern Field{{< /subtle-label >}}

The pattern is how I tell Ansible which devices to target.

- `ansible all` - Every device in the inventory
- `ansible cisco_ios` - All devices in the cisco_ios group
- `ansible spine_switches` - All devices in the spine_switches group
- `ansible wan-r1` - A specific device
- `ansible "wan-r1,wan-r2"` - Multiple devices
- `ansible "cisco_ios:cisco_nxos"` - 2 groups
- `ansible "all:!linux_hosts"` - All devices except linux_hosts
- `ansible "spine*"` - Wildcard, all devices whose name starts with "spine"

---

{{< subtle-label >}}Ping Test{{< /subtle-label >}}

The ad-hoc command `ansible.builtin.ping` doesn't send ICMP pings. It verifies that Ansible can connect to and authenticate with a device and that Python is available (for Linux hosts).

---

**Test all devices**

{{< codeblock lang="Bash" syntax="bash" >}}
ansible all -m ansible.builtin.ping
{{< /codeblock >}}

**Test a specific group**

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m ansible.builtin.ping
{{< /codeblock >}}

**Test a single device**

{{< codeblock lang="Bash" syntax="bash" >}}
ansible wan-r1 -m ansible.builtin.ping
{{< /codeblock >}}

----

**Output Explained**

{{< codeblock lang="Successful Output" copy="false" >}}
wan-r1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
wan-r2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
{{< /codeblock >}}

---

{{< codeblock lang="Failed output (device unreachable)" copy="false" >}}
wan-r1 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: ssh: connect to host 172.16.0.11
            port 22: Connection timed out",
    "unreachable": true
}
{{< /codeblock >}}

---

**Failed output (authentication error)**

{{< codeblock lang="Failed output (authentication error)" copy="false" >}}
wan-r1 | FAILED! => {
    "changed": false,
    "msg": "Failed to authenticate: Authentication failed.",
    "rc": 255
}
{{< /codeblock >}}

---

- `UNREACHABLE` - Network connectivity problem. The device is down, the IP is wrong, or a filewall is blocking port 22.
- `FAILED` with authentication message - Credentials are wrong. Check `ansible_user` and `ansible_password` in group_vars.
- `FAILED` with SSH key message - Key not accepted. Check `~/.ssh/known_hosts` or run with `-vvv`.

---

### Show Commands

`cisco.ios.ios_command` is the module for running CLI commands on IOS-XE devices. It's the most common ad-hoc module for investigation work.

---

{{< subtle-label >}}Basic Show Commands{{< /subtle-label >}}

Run a single show command on all IOS devices

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_command -a "commands='show version'"
{{< /codeblock >}}

Run on a single device

{{< codeblock lang="Bash" syntax="bash" >}}
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show ip interface brief'"
{{< /codeblock >}}

Run mutliple commands

{{< codeblock lang="Bash" syntax="bash" >}}
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands=['show ip bgp summary', 'show ip route']"
{{< /codeblock >}}

---

**Example output from running 'show ip interface brief'**

{{< codeblock lang="Bash" syntax="bash" >}}
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show ip interface brief'"
{{< /codeblock >}}

{{< codeblock lang="" copy="false" >}}
wan-r1 | SUCCESS => {
    "changed": false,
    "stdout": [
        "Interface              IP-Address      OK? Method Status                Protocol\nGigabitEthernet1       10.10.10.1      YES manual up                    up      \nGigabitEthernet2       10.10.20.1      YES manual up                    up      \nLoopback0              10.255.0.1      YES manual up                    up      "
    ],
    "stdout_lines": [
        [
            "Interface              IP-Address      OK? Method Status                Protocol",
            "GigabitEthernet1       10.10.10.1      YES manual up                    up      ",
            "GigabitEthernet2       10.10.20.1      YES manual up                    up      ",
            "Loopback0              10.255.0.1      YES manual up                    up      "
        ]
    ]
}
{{< /codeblock >}}

- `stdout` - a list containing the raw output string of each command.
- `stdout_lines` - the same output split into a list of lines.
- `changed: false` - show commands never report a change.

---

{{< subtle-label >}}Useful Show Commands{{< /subtle-label >}}

Verify device identity

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_command -a "commands='show version'" -o
{{< /codeblock >}}

Check interface states

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_command -a "commands='show ip interface brief'"
{{< /codeblock >}}

Check routing table

{{< codeblock lang="Bash" syntax="bash" >}}
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show ip route'"
{{< /codeblock >}}

Check BGP neighbors

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_command -a "commands='show ip bgp summary'"
{{< /codeblock >}}

Check OSPF neighbors

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_command -a "commands='show ip ospf neighbor'"
{{< /codeblock >}}

Check running config

{{< codeblock lang="Bash" syntax="bash" >}}
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show running-config'"
{{< /codeblock >}}

Check NTP sync status

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_command -a "commands='show ntp status'" -o
{{< /codeblock >}}

Check CDP/LLDP neighbors

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_command -a "commands='show cdp neighbors detail'"
{{< /codeblock >}}

---

{{< subtle-label >}}PAN-OS{{< /subtle-label >}}

PAN-OS uses the `httpapi` connection type. I have to use `paloaltonetworks.panos.panos_op` for commands.

Example:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible paloalto -m paloaltonetworks.panos.panos_op -a "cmd='show system info'"
{{< /codeblock >}}

---

{{< subtle-label >}}Linux Host{{< /subtle-label >}}

For Linus host Iuse standard Ansible modules.

Example:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible linux_hosts -m ansible.builtin.command -a "ip route show"
{{< /codeblock >}}

---

### Gathering Facts

Ansible's facts modules collect data from devices like software version, interface list, routing table, and hardware model.

---

Gather all default facts from IOS devices

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_facts
{{< /codeblock >}}

Gather only specific subsets to speed things up

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_facts -a "gather_subset=['default', 'interfaces']"
{{< /codeblock >}}

Gather facts from a single device

{{< codeblock lang="Bash" syntax="bash" >}}
ansible wan-r1 -m cisco.ios.ios_facts -a "gather_subset=['all']"
{{< /codeblock >}}

{{< codeblock lang="Example Output" copy="false" >}}
wan-r1 | SUCCESS => {
    "ansible_facts": {
        "ansible_net_all_ipv4_addresses": [
            "10.10.10.1",
            "10.10.20.1",
            "10.255.0.1"
        ],
        "ansible_net_hostname": "wan-r1",
        "ansible_net_interfaces": {
            "GigabitEthernet1": {
                "bandwidth": 1000000,
                "description": "WAN | To FW-01 eth1",
                "duplex": "Full",
                "ipv4": [{"address": "10.10.10.1", "subnet": "30"}],
                "lineprotocol": "up",
                "macaddress": "...",
                "mtu": 1500,
                "operstatus": "up",
                "type": "CSR vNIC"
            }
        },
        "ansible_net_model": "CSR1000V",
        "ansible_net_serialnum": "...",
        "ansible_net_version": "17.06.01",
        "ansible_net_neighbors": {...},
        "ansible_net_config": "...",    # Full running config as a string
    },
    "changed": false
}
{{< /codeblock >}}

---

| Fact Variable | What It Contains |
|---|---|
| `ansible_net_hostname` | Device hostname |
| `ansible_net_version` | IOS/NX-OS version string |
| `ansible_net_model` | Hardware model |
| `ansible_net_serialnum` | Serial number |
| `ansible_net_interfaces` | Dict of all interfaces with state and IP info |
| `ansible_net_all_ipv4_addresses` | List of all IPv4 addresses |
| `ansible_net_neighbors` | CDP/LLDP neighbor dict |
| `ansible_net_config` | Full running configuration |

---

{{< subtle-label >}}Limit Scope{{< /subtle-label >}}

The `-l` flag narrows an ad-hoc command to a subset of the targeted hosts.

---

Run against a group, but limit to a specific device

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_command -a "commands='show version'" -l wan-r1
{{< /codeblock >}}

Limit to multiple specific devices

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_nxos -m cisco.nxos.nxos_command -a "commands='show vlan brief'" -l "spine-01,spine-02"
{{< /codeblock >}}

Limit to a subgroup

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_nxos -m cisco.nxos.nxos_command -a "commands='show interface status'" -l spine_switches
{{< /codeblock >}}

Limit using wildcards

{{< codeblock lang="Bash" syntax="bash" >}}
ansible all -m anible.netcommon.net_ping -l "leaf*"
{{< /codeblock >}}

Limit to a group except one device

{{< codeblock lang="Bash" syntax="bash" >}}
ansible cisco_ios -m cisco.ios.ios_command -a "commands='show ip bgp summary'" -l "cisco_ios:!wan-r2"
{{< /codeblock >}}

---

## Verbosity Flags

The verbosity flags are extremely useful for debugging. Each level adds more detail about what Ansible is doing behind the scenes.

---

{{< subtle-label >}}Three Levels{{< /subtle-label >}}

- `-v` - Level 1, task results

Example:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show ntp status'" -v
{{< /codeblock >}}

{{< codeblock lang="Output" copy="false" >}}
wan-r1 | SUCCESS => {
    "changed": false,
    "invocation": {                      ← Added by -v: shows module and args used
        "module_args": {
            "commands": ["show ntp status"],
            "interval": 1,
            "match": "all",
            "retries": 10,
            "wait_for": null
        }
    },
    "stdout": [
        "Clock is synchronized, stratum 2, reference is 216.239.35.0\n..."
    ],
    "stdout_lines": [
        ["Clock is synchronized, stratum 2, reference is 216.239.35.0", ...]
    ]
}
{{< /codeblock >}}

The `invocation` block shows exactly which module arguments were used. This is useful for confirming that the arguments I passed are being interpreted correctly.

---

- `-vv` - Level 2, connection details added

Example:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show version'" -vv
{{< /codeblock >}}

{{< codeblock lang="Output" copy="false" >}}
ESTABLISH PERSISTENT CONNECTION FOR USER: ansible on PORT 22 TO 172.16.0.11  # ← New at -vv
SSH connection established to 172.16.0.11:22                                  # ← New at -vv
...
wan-r1 | SUCCESS => {
    ...
}
{{< /codeblock >}}

Connection establishment details shows which user, which IP, which port. This level is where you can see evidence of SSH connection issues.

---

- `-vvv` - Level 3, full SSH debug output

{{< codeblock lang="Bash" syntax="bash" >}}
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show version'" -vvv
{{< /codeblock >}}

{{< codeblock lang="Output" copy="false" >}}
Using /home/ansible/projects/ansible-network/ansible.cfg as config file
Connecting to 172.16.0.11:22 as user ansible                          
<wan-r1> ESTABLISH PERSISTENT CONNECTION FOR USER: ansible             
<wan-r1> SSH: EXEC ssh -o ControlMaster=no -o ControlPersist=no       
         -o StrictHostKeyChecking=no -o Port=22 
         ansible@172.16.0.11 '/bin/sh -c '"'"'echo ~ansible &&
         sleep 0'"'"''
<wan-r1> (0, '/home/ansible\n', '')                                    
<wan-r1> EXEC /bin/sh -c 'echo PLATFORM && uname'                     
<wan-r1> Sending XML...                                                
...
{{< /codeblock >}}

This adds the exact SSH command Ansible is running, the SSH key files being tried, the exact bytes being sent to and received from the device, and Python exceptions with full tracebacks when something fails.

---

## Check and Diff Flags

{{< subtle-label >}}Check{{< /subtle-label >}}

On Linux hosts `--check` does a dry-run of the playbook. Ansible simulates the change, reports whether it would change anything, but makes no actual modifications.

{{< codeblock lang="Bash" syntax="bash" >}}
ansible linux_hosts -m ansible.builtin.copy -a "src=files/banner.txt dest=/etc/motd" --check
{{< /codeblock >}}

---

On network devices `--check` behavior depends on the module.

Resources modules like `ios_vlans`, `ios_bgp_global`, and `nxos_vlans` support `--check`.

These modules can compare desired state against current state without applying changes because they use structured data.

With `ios_config` and `--check` Ansible reports would it would send but cannot verify wheather the config actually needs changing, since it has no way to compare current config against the desired lines without connecting and checking.

---

{{< subtle-label >}}Diff{{< /subtle-label >}}

`--diff` shows the before/after of configuration changes.

For `ios_config` the `--diff` shows the config lines Ansible would send.

{{< codeblock lang="Example Output" copy="false" >}}
wan-r1 | CHANGED => {
    "diff": {
        "prepared": "interface GigabitEthernet1\n description TEST\n"
    }
}
{{< /codeblock >}}

---

Ad-hoc commands are quick checks, fast debugging, and pre-change verification.

