---
draft: true
title: '9 - Adhoc'
weight: 9
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 9: Ansible Ad-Hoc Commands

> *Before I run a single playbook, ad-hoc commands are how I interact with the lab. They're the command-line equivalent of SSHing into a device and typing a show command — except Ansible handles the connection, the authentication, and runs against every device in a group simultaneously. This part covers ad-hoc commands thoroughly because they're also the fastest debugging tool I have when something in a playbook isn't working the way I expect.*

---

## 9.1 — What Ad-Hoc Commands Are and When to Use Them

An ad-hoc command is a single Ansible module call executed directly from the command line — no playbook file, no YAML, just a command and a result. It's designed for quick, one-time operations that don't need to be repeatable or documented in a playbook.

### The Mental Model

```
Ad-hoc command:
ansible <hosts> -m <module> -a "<arguments>"
    │              │              │
    │              │              └── Module parameters (like task args in a playbook)
    │              └── The module to run (ios_command, ping, debug, etc.)
    └── Which hosts or groups to target (from inventory)

Equivalent playbook task:
- hosts: <hosts>
  tasks:
    - <module>:
        <arguments>
```

They're the same operation — one typed at the command line, one written in YAML.

### When to Use Ad-Hoc Commands

```
✅ USE AD-HOC WHEN:                    ❌ USE A PLAYBOOK WHEN:
────────────────────────────────────   ────────────────────────────────────
Quick show command across devices      Changes need to be repeatable
Testing connectivity before a run      Work needs to be version-controlled
Checking a single variable or fact     Multiple tasks need to run in order
Verifying a module works correctly     The change is going to production
One-time information gathering         The output needs to be parsed/saved
Debugging a failed playbook            Multiple people need to run it
Lab exploration and learning           The operation needs review/approval
```

### ### ℹ️ Info

> Ad-hoc commands are also the right tool during an incident. If a BGP session drops at 2am and I need to check neighbor states across 20 routers immediately, I don't write a playbook — I run an ad-hoc command and get the answer in seconds. The discipline of "everything must be a playbook" is correct for planned changes, not for read-only investigation during an outage.

### ### 🏢 Real-World Scenario

> On a network operations team I know of, engineers had a rule: any `show` command that needed to be run across more than 3 devices got run as an Ansible ad-hoc command instead of manual SSH sessions. Commands like checking NTP sync status across all routers, verifying interface states before a maintenance window, or confirming BGP neighbors after a routing change — all ad-hoc. The manual SSH approach took 20 minutes for 20 devices. The ad-hoc command took 20 seconds. The engineers who learned this pattern early stopped doing manual multi-device checks entirely.

---

## 9.2 — The Ad-Hoc Command Syntax

```bash
ansible <pattern> \
    -m <module_name> \
    -a "<module_arguments>" \
    -i <inventory_path> \
    [options]
```

### Every Flag Explained

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
| `--check` | | Dry run — predict changes without applying |
| `--diff` | | Show before/after diff of changes |
| `--list-hosts` | | Show which hosts match the pattern, don't run |

### The `<pattern>` Field

The pattern is how I tell Ansible which devices to target. It maps directly to inventory groups and hostnames:

```bash
ansible all          # Every device in the inventory
ansible cisco_ios    # All devices in the cisco_ios group
ansible spine_switches   # All devices in spine_switches group
ansible wan-r1       # A single specific device
ansible "wan-r1,wan-r2"  # Multiple specific devices (comma-separated)
ansible "cisco_ios:cisco_nxos"  # Union of two groups (colon separator)
ansible "all:!linux_hosts"      # All devices except linux_hosts
ansible "spine*"     # Wildcard — all devices whose name starts with "spine"
```

### ### 💡 Tip

> Before running any ad-hoc command that makes changes, I always run `--list-hosts` first to confirm which devices will be targeted:
> ```bash
> ansible "spine*" --list-hosts
> # hosts (2):
> #   spine-01
> #   spine-02
> ```
> This prevents the "I thought I was targeting staging but I hit production" mistake. Takes two seconds and can save a lot of pain.

---

## 9.3 — The Ansible Ping Test

The most basic ad-hoc command. `ansible.builtin.ping` doesn't send ICMP pings — it verifies that Ansible can connect to and authenticate with a device and that Python is available (for Linux hosts).

```bash
# Test all devices
ansible all -m ansible.builtin.ping

# Test a specific group
ansible cisco_ios -m ansible.builtin.ping

# Test a single device
ansible wan-r1 -m ansible.builtin.ping
```

### ### ⚠️ Warning

> `ansible.builtin.ping` only works on Linux hosts where Python is available. It does **not** work on network devices (IOS, NX-OS, PAN-OS) — they can't run Python modules. For network devices, use `ansible.netcommon.net_ping` instead:
> ```bash
> # For network devices
> ansible cisco_ios -m ansible.netcommon.net_ping
> ansible cisco_nxos -m ansible.netcommon.net_ping
>
> # For Linux hosts
> ansible linux_hosts -m ansible.builtin.ping
> ```

### Understanding the Output

```bash
ansible cisco_ios -m ansible.netcommon.net_ping
```

Successful output:
```yaml
wan-r1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
wan-r2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Failed output (device unreachable):
```yaml
wan-r1 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: ssh: connect to host 172.16.0.11
            port 22: Connection timed out",
    "unreachable": true
}
```

Failed output (authentication error):
```yaml
wan-r1 | FAILED! => {
    "changed": false,
    "msg": "Failed to authenticate: Authentication failed.",
    "rc": 255
}
```

Each failure type tells me something different:
- `UNREACHABLE` — network connectivity problem. The device is down, the IP is wrong, or a firewall is blocking port 22.
- `FAILED` with authentication message — credentials are wrong. Check `ansible_user` and `ansible_password` in group_vars.
- `FAILED` with SSH key message — key not accepted. Check `~/.ssh/known_hosts` or run with `-vvv`.

---

## 9.4 — Running Show Commands (IOS-XE in Depth)

`cisco.ios.ios_command` is the module for running arbitrary CLI commands on IOS-XE devices. It's the most common ad-hoc module for network investigation work.

### Basic Show Commands

```bash
# Run a single show command on all IOS devices
ansible cisco_ios -m cisco.ios.ios_command -a "commands='show version'"

# Run on a single device
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show ip interface brief'"

# Run multiple commands (comma-separated in a list)
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands=['show ip bgp summary', 'show ip route']"
```

### Example Output — `show ip interface brief`

```bash
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show ip interface brief'"
```

```yaml
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
```

**Understanding the output structure:**
- `stdout` — a list containing the raw output string of each command. Index 0 is the first command's output.
- `stdout_lines` — the same output split into a list of lines. Much easier to work with in playbooks.
- `changed: false` — show commands never report a change. Only configuration changes set `changed: true`.

### Running Commands Against All IOS Devices Simultaneously

```bash
# Check BGP summary on all IOS devices at once
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip bgp summary'" -o
```

The `-o` flag (one-line output) compresses each host's result to a single line — useful when I want a quick status check across many devices without scrolling through pages of output:

```
wan-r1 | SUCCESS => {"changed": false, "stdout": ["BGP router identifier 10.255.0.1..."], ...}
wan-r2 | SUCCESS => {"changed": false, "stdout": ["BGP router identifier 10.255.0.2..."], ...}
```

### ### 💡 Tip

> For multi-line output that's hard to read in the terminal, I pipe to `less`:
> ```bash
> ansible wan-r1 -m cisco.ios.ios_command \
>     -a "commands='show running-config'" | less
> ```
> Or write output to a file for later review:
> ```bash
> ansible wan-r1 -m cisco.ios.ios_command \
>     -a "commands='show running-config'" > /tmp/wan-r1-runningconfig.txt
> ```

### Useful Show Commands for Each Stage of Lab Work

```bash
# ---- Verify device identity ----
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show version'" -o

# ---- Check interface states ----
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip interface brief'"

# ---- Check routing table ----
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show ip route'"

# ---- Check BGP neighbors ----
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip bgp summary'"

# ---- Check OSPF neighbors ----
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip ospf neighbor'"

# ---- Check running config ----
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show running-config'"

# ---- Check NTP sync status ----
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ntp status'" -o

# ---- Check CDP/LLDP neighbors ----
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show cdp neighbors detail'"

# ---- Check ACL hit counts ----
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip access-lists'"
```

### NX-OS Differences

NX-OS uses `cisco.nxos.nxos_command` — the syntax is identical, just the module name changes:

```bash
# NX-OS equivalent commands
ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show interface status'"

ansible spine-01 -m cisco.nxos.nxos_command \
    -a "commands='show bgp summary'"

ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show vlan brief'" -o

# NX-OS feature states
ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show feature'" -o
```

### PAN-OS Differences

PAN-OS uses the `httpapi` connection type, so I can't use CLI command modules. Instead, I use `paloaltonetworks.panos.panos_op` for operational commands:

```bash
# PAN-OS operational commands
ansible paloalto -m paloaltonetworks.panos.panos_op \
    -a "cmd='show system info'"

ansible fw-01 -m paloaltonetworks.panos.panos_op \
    -a "cmd='show interface all'"
```

### Linux Host Differences

For Linux hosts, I use standard Ansible modules:

```bash
# Run a shell command on Linux hosts
ansible linux_hosts -m ansible.builtin.command \
    -a "ip route show"

# Check a service status
ansible linux_hosts -m ansible.builtin.shell \
    -a "systemctl status ssh"

# Check disk space
ansible linux_hosts -m ansible.builtin.command \
    -a "df -h"
```

---

## 9.5 — Gathering Facts from Devices

Ansible's facts modules collect structured data from devices — software version, interface list, routing table snapshot, hardware model. Facts are the foundation of dynamic playbooks that make decisions based on what's actually on the device.

### Gathering IOS-XE Facts

```bash
# Gather all default facts from IOS devices
ansible cisco_ios -m cisco.ios.ios_facts

# Gather only specific subsets to speed things up
ansible cisco_ios -m cisco.ios.ios_facts \
    -a "gather_subset=['default', 'interfaces']"

# Gather facts from a single device (most readable)
ansible wan-r1 -m cisco.ios.ios_facts \
    -a "gather_subset=['all']"
```

The facts output is large. Here's the structure of what comes back (abbreviated):

```yaml
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
```

**Key facts variables I'll use constantly in playbooks:**

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

### Gathering NX-OS Facts

```bash
ansible cisco_nxos -m cisco.nxos.nxos_facts \
    -a "gather_subset=['default', 'hardware', 'interfaces']"
```

NX-OS facts use the same `ansible_net_*` namespace — the variable names are consistent across platforms, which makes writing cross-platform playbooks much easier.

### ### ℹ️ Info

> Facts gathered via ad-hoc command are not stored anywhere — they're just printed to the terminal. When I use `ios_facts` inside a playbook, the gathered facts are stored in memory for the duration of the play and can be referenced in subsequent tasks. The ad-hoc command is useful for exploration — "let me see what facts are available" — before writing playbook logic that uses those facts.

---

## 9.6 — Using `-l` to Limit Scope

The `-l` (or `--limit`) flag narrows an ad-hoc command to a subset of the targeted hosts. It's one of the most important flags for safe operations.

```bash
# Run against a group, but limit to a specific device
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show version'" \
    -l wan-r1

# Limit to multiple specific devices
ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show vlan brief'" \
    -l "spine-01,spine-02"

# Limit to a subgroup
ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show interface status'" \
    -l spine_switches

# Limit using wildcards
ansible all -m ansible.netcommon.net_ping \
    -l "leaf*"

# Limit to a group except one device
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip bgp summary'" \
    -l "cisco_ios:!wan-r2"
```

### ### 🏢 Real-World Scenario

> The `-l` flag is what I use during a staged rollout. Imagine I'm deploying a new BGP configuration. I don't run it against all 20 routers at once — I run it against one device with `-l router-01`, verify the output, then expand to a small group with `-l "router-01,router-02"`, verify again, then run against the full group. Ad-hoc commands with `-l` let me validate each stage of the rollout interactively before committing to the full deployment. In enterprise change management, this staged validation approach is often a requirement documented in the change ticket.

---

## 9.7 — Verbosity Flags: `-v`, `-vv`, `-vvv`

The verbosity flags are my primary debugging tool. Each level adds more detail about what Ansible is doing behind the scenes.

### The Four Levels

```bash
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show version'"        # Default — minimal output

ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show version'" -v     # Level 1 — task results expanded

ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show version'" -vv    # Level 2 — connection details added

ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show version'" -vvv   # Level 3 — full SSH debug output
```

### Level 1 (`-v`) — Task Results

Without verbosity, a successful task just shows `SUCCESS`. With `-v`, I see the full return dictionary:

```bash
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show ntp status'" -v
```

```yaml
wan-r1 | SUCCESS => {
    "changed": false,
    "invocation": {                      # ← Added by -v: shows module and args used
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
```

**What `-v` adds:** The `invocation` block shows exactly which module arguments were used. This is useful for confirming that the arguments I passed are being interpreted correctly — especially when using complex argument structures.

### Level 2 (`-vv`) — Connection Details

```bash
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show version'" -vv
```

```
ESTABLISH PERSISTENT CONNECTION FOR USER: ansible on PORT 22 TO 172.16.0.11  # ← New at -vv
SSH connection established to 172.16.0.11:22                                  # ← New at -vv
...
wan-r1 | SUCCESS => {
    ...
}
```

**What `-vv` adds:** Connection establishment details — which user, which IP, which port. This level is where I first see evidence of SSH connection issues. If the connection hangs, `-vv` shows me exactly which step it's stuck on.

### Level 3 (`-vvv`) — Full SSH Debug

This is the level I use when I have a genuine connection problem:

```bash
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show version'" -vvv 2>&1 | head -60
```

```
Using /home/ansible/projects/ansible-network/ansible.cfg as config file
Connecting to 172.16.0.11:22 as user ansible                          # ← SSH target
<wan-r1> ESTABLISH PERSISTENT CONNECTION FOR USER: ansible             # ← Persistent conn
<wan-r1> SSH: EXEC ssh -o ControlMaster=no -o ControlPersist=no       # ← Exact SSH command
         -o StrictHostKeyChecking=no -o Port=22                        #    used by Ansible
         ansible@172.16.0.11 '/bin/sh -c '"'"'echo ~ansible &&
         sleep 0'"'"''
<wan-r1> (0, '/home/ansible\n', '')                                    # ← SSH response
<wan-r1> EXEC /bin/sh -c 'echo PLATFORM && uname'                     # ← Platform detection
<wan-r1> Sending XML...                                                # ← Module communication
...
```

**What `-vvv` adds:**
- The exact SSH command Ansible is running (including all flags)
- The SSH key files being tried
- The exact bytes being sent to and received from the device
- Python exceptions with full tracebacks when something fails

### Reading a Real Failure at `-vvv`

Here's an annotated example of a connection failure:

```
<wan-r1> ESTABLISH PERSISTENT CONNECTION FOR USER: ansible    # Step 1: trying to connect
<wan-r1> SSH: EXEC ssh -o StrictHostKeyChecking=no            # Step 2: SSH command launched
         ansible@172.16.0.11                                   # Step 3: target
<wan-r1> (255, b'', b'ssh: connect to host 172.16.0.11        # Step 4: SSH returned error 255
         port 22: Connection refused\n')                       #         port 22 refused
...
wan-r1 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh:             # ← The actual error
            ssh: connect to host 172.16.0.11 port 22:
            Connection refused",
    "unreachable": true
}
```

Reading from the bottom up (as I learned in Part 2):
1. `UNREACHABLE` + `Connection refused` — SSH port 22 is not open on this IP
2. The IP `172.16.0.11` is correct (it's in the inventory)
3. Possible causes: device not booted yet, SSH not enabled on device, wrong IP in inventory, Containerlab not running

### ### 💡 Tip

> I keep `-vvv` output out of my terminal's scroll buffer by piping to a file:
> ```bash
> ansible wan-r1 -m cisco.ios.ios_command \
>     -a "commands='show version'" -vvv > /tmp/debug_output.txt 2>&1
> less /tmp/debug_output.txt
> ```
> The `2>&1` redirects stderr (where most debug output goes) to stdout, so both streams go to the file. Without it, the debug output goes to the terminal while only the task result goes to the file.

---

## 9.8 — The `--check` and `--diff` Flags on Network Devices

### What `--check` Does on Linux Hosts

On Linux hosts, `--check` is a reliable dry-run mode. Ansible simulates the change, reports whether it would change anything, but makes no actual modifications. This works because Linux modules can compare the desired state against the current state without executing the change.

```bash
# On Linux hosts -- check works reliably
ansible linux_hosts -m ansible.builtin.copy \
    -a "src=files/banner.txt dest=/etc/motd" \
    --check
# Reports "would be changed" or "ok" without modifying anything
```

### What `--check` Does (and Doesn't Do) on Network Devices

On network devices, `--check` behavior depends entirely on the module:

**Resource modules** (like `ios_vlans`, `ios_bgp_global`, `nxos_vlans`) **support `--check`:**

```bash
ansible cisco_ios -m cisco.ios.ios_vlans \
    -a "config=[{vlan_id: 100, name: TEST}] state=merged" \
    --check
```

These modules can compare desired state against current state without applying changes because they use structured data. `--check` works reliably here.

**`ios_config` and `ios_command` do NOT support `--check` reliably:**

```bash
ansible wan-r1 -m cisco.ios.ios_config \
    -a "lines='router bgp 65100'" \
    --check
```

With `ios_config` and `--check`, Ansible reports what it *would* send but cannot verify whether the config actually needs changing — it has no way to compare current config against the desired lines without connecting and checking. In practice, `--check` with `ios_config` often reports `changed: true` even when no change is needed, or `changed: false` when the change actually would be applied. It's unreliable.

### ### ⚠️ Warning

> Do not rely on `--check` with `ios_config`, `nxos_config`, or any raw command module as a safety mechanism before running in production. These modules don't have a true dry-run mode. The safer approach is:
> 1. Test in Containerlab first (the whole point of having a lab)
> 2. Use `--diff` to see what config lines Ansible would push
> 3. Use resource modules (`ios_vlans`, `ios_bgp_global`, etc.) instead of `ios_config` where possible — they support `--check` reliably

### `--diff` — Showing Configuration Changes

`--diff` shows the before/after diff of configuration changes. This is more useful than `--check` for network devices because it shows exactly what lines would be added or removed:

```bash
ansible wan-r1 -m cisco.ios.ios_vlans \
    -a "config=[{vlan_id: 100, name: TEST_VLAN}] state=merged" \
    --diff --check
```

```yaml
wan-r1 | SUCCESS => {
    "changed": true,
    "diff": {
        "after": "vlan 100\n  name TEST_VLAN\n",     # ← What would be added
        "before": ""                                   # ← Nothing currently there
    }
}
```

For `ios_config`, `--diff` shows the config lines Ansible would send:

```bash
ansible wan-r1 -m cisco.ios.ios_config \
    -a "lines='description TEST' parents='interface GigabitEthernet1'" \
    --diff
```

```yaml
wan-r1 | CHANGED => {
    "diff": {
        "prepared": "interface GigabitEthernet1\n description TEST\n"
    }
}
```

### The Correct Pre-Change Workflow for Network Devices

Since `--check` isn't fully reliable for network devices, my actual pre-change safety workflow is:

```bash
# Step 1: Test the exact command in Containerlab lab first
ansible wan-r1 -m cisco.ios.ios_vlans \
    -a "config=[{vlan_id: 100, name: PROD_VLAN}] state=merged"

# Step 2: Verify the change took effect in the lab
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show vlan brief'"

# Step 3: Use --diff against production to preview the change
ansible wan-r1 -m cisco.ios.ios_vlans \
    -a "config=[{vlan_id: 100, name: PROD_VLAN}] state=merged" \
    --diff --check

# Step 4: If diff looks correct, run without --check
ansible wan-r1 -m cisco.ios.ios_vlans \
    -a "config=[{vlan_id: 100, name: PROD_VLAN}] state=merged"

# Step 5: Validate
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show vlan brief'"
```

---

## 9.9 — A Reference of Useful Ad-Hoc Commands for Network Engineers

This is the quick-reference section I come back to repeatedly. Organized by task type.

### Connectivity and Reachability

```bash
# Test Ansible can reach all devices
ansible all -m ansible.netcommon.net_ping
ansible linux_hosts -m ansible.builtin.ping

# Test a specific group
ansible cisco_nxos -m ansible.netcommon.net_ping -o

# Check which hosts match a pattern without running anything
ansible "spine*" --list-hosts
ansible "cisco_ios:cisco_nxos" --list-hosts
```

### Device Information

```bash
# Quick version check across all IOS devices (one-line output)
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show version | include Version'" -o

# Model numbers
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show version | include cisco'" -o

# Check uptime on all devices
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show version | include uptime'" -o

# NX-OS version check
ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show version | include NXOS'" -o
```

### Interface and Connectivity Checks

```bash
# Interface status on all IOS devices
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip interface brief'"

# Find any interfaces that are administratively down
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show interface | include down'" -o

# NX-OS interface status
ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show interface status'"

# CDP neighbors from all IOS devices
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show cdp neighbors'"

# LLDP neighbors (NX-OS)
ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show lldp neighbors'"
```

### Routing and BGP

```bash
# BGP summary from all IOS devices
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip bgp summary'"

# BGP summary from specific router
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show ip bgp summary'"

# OSPF neighbors
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip ospf neighbor'" -o

# Routing table (specific prefix)
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show ip route 10.0.0.0'"

# NX-OS BGP summary
ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show bgp summary'"
```

### NTP and Time

```bash
# NTP sync status — is the clock synchronized?
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ntp status | include Clock'" -o

ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show ntp status | include Clock'" -o

# NTP associations (which server is it talking to?)
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ntp associations'"
```

### Gathering Facts (Structured Data)

```bash
# Gather all facts from a single device
ansible wan-r1 -m cisco.ios.ios_facts -a "gather_subset=['all']"

# Gather only interface facts
ansible cisco_ios -m cisco.ios.ios_facts \
    -a "gather_subset=['interfaces']" -o

# Gather NX-OS hardware facts
ansible cisco_nxos -m cisco.nxos.nxos_facts \
    -a "gather_subset=['hardware']"
```

### Linux Hosts

```bash
# Check disk space on Linux hosts
ansible linux_hosts -m ansible.builtin.command -a "df -h /"

# Check network interfaces
ansible linux_hosts -m ansible.builtin.command -a "ip addr show"

# Check routing table
ansible linux_hosts -m ansible.builtin.command -a "ip route show"

# Install a package on Alpine Linux hosts
ansible linux_hosts -m ansible.builtin.package \
    -a "name=curl state=present" -b

# Copy a file to Linux hosts
ansible linux_hosts -m ansible.builtin.copy \
    -a "src=files/test.txt dest=/tmp/test.txt"
```

---

## 9.10 — Building a Pre-Change Checklist Script

I create a script that runs a set of ad-hoc commands to document the network state before any change window. I compare this output to post-change output to verify nothing unexpected changed.

```bash
nano ~/projects/ansible-network/scripts/pre-change-check.sh
```

```bash
#!/bin/bash
# =============================================================
# pre-change-check.sh
# Run before any change window to capture current network state.
# Compare output before and after the change.
#
# Usage: bash scripts/pre-change-check.sh [optional-change-id]
# Example: bash scripts/pre-change-check.sh CHG0019823
# =============================================================

CHANGE_ID=${1:-"no-change-id"}
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTPUT_DIR="/tmp/pre-change-${CHANGE_ID}-${TIMESTAMP}"
mkdir -p "$OUTPUT_DIR"

echo "============================================"
echo "Pre-change check: ${CHANGE_ID}"
echo "Timestamp: ${TIMESTAMP}"
echo "Output directory: ${OUTPUT_DIR}"
echo "============================================"

# Activate virtual environment if not already active
if [ -z "$VIRTUAL_ENV" ]; then
    source ~/venvs/ansible-network/bin/activate
fi

# Navigate to project root
cd ~/projects/ansible-network

echo ""
echo "[1/6] Testing Ansible connectivity to all devices..."
ansible network_devices -m ansible.netcommon.net_ping -o \
    | tee "${OUTPUT_DIR}/01_connectivity.txt"

echo ""
echo "[2/6] Gathering IOS-XE interface states..."
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip interface brief'" \
    | tee "${OUTPUT_DIR}/02_ios_interfaces.txt"

echo ""
echo "[3/6] Gathering IOS-XE BGP summary..."
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ip bgp summary'" \
    | tee "${OUTPUT_DIR}/03_ios_bgp.txt"

echo ""
echo "[4/6] Gathering NX-OS interface states..."
ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show interface status'" \
    | tee "${OUTPUT_DIR}/04_nxos_interfaces.txt"

echo ""
echo "[5/6] Gathering NX-OS BGP summary..."
ansible cisco_nxos -m cisco.nxos.nxos_command \
    -a "commands='show bgp summary'" \
    | tee "${OUTPUT_DIR}/05_nxos_bgp.txt"

echo ""
echo "[6/6] Gathering NTP sync status..."
ansible cisco_ios -m cisco.ios.ios_command \
    -a "commands='show ntp status | include Clock'" -o \
    | tee "${OUTPUT_DIR}/06_ntp_status.txt"

echo ""
echo "============================================"
echo "Pre-change check complete."
echo "Output saved to: ${OUTPUT_DIR}"
echo "Run the same script after the change as:"
echo "  bash scripts/pre-change-check.sh ${CHANGE_ID}-POST"
echo "Then compare with:"
echo "  diff ${OUTPUT_DIR} /tmp/pre-change-${CHANGE_ID}-POST-*/"
echo "============================================"
```

```bash
chmod +x scripts/pre-change-check.sh
```

Using it:

```bash
# Before the change window
bash scripts/pre-change-check.sh CHG0019823

# After the change window
bash scripts/pre-change-check.sh CHG0019823-POST

# Compare the two
diff /tmp/pre-change-CHG0019823-*/ /tmp/pre-change-CHG0019823-POST-*/
```

### ### 🏢 Real-World Scenario

> Pre/post change documentation is a formal requirement in most enterprise change management processes. Before any network change, the engineer is expected to document the current state — interface statuses, routing neighbors, BGP sessions — and after the change, document the new state and confirm everything that should have changed did change, and nothing that should have stayed the same changed unexpectedly. This script automates what engineers used to do by hand (copy-pasting show command output into a change ticket). Run it before the change, run it again after, and attach both files to the change ticket.

---

## 9.11 — Common Gotchas in This Section

### ### 🪲 Gotcha — `-a` argument quoting on the command line

Shell quoting around the `-a` arguments can be tricky. The safest pattern is always to use single quotes around the entire `-a` value:

```bash
# ✅ Correct — single quotes around entire argument string
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show version'"

# ✅ Also correct — double quotes, single inside
ansible wan-r1 -m cisco.ios.ios_command -a 'commands="show version"'

# ❌ Problem — unquoted list with spaces
ansible wan-r1 -m cisco.ios.ios_command -a commands=['show version','show ip route']
# Shell interprets the brackets and quotes incorrectly

# ✅ Correct for lists — use JSON-style inside the string
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands=['show version', 'show ip route']"
```

### ### 🪲 Gotcha — Ad-hoc commands ignore `ansible.cfg` `become` settings for network devices

Even if `ansible_become: true` is set in `group_vars/cisco_ios.yml`, ad-hoc commands sometimes need the `-b` flag explicitly:

```bash
# If a command requires enable mode and returns a permissions error:
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show running-config'" -b

# Or check that become is set correctly in group_vars
ansible-inventory --host wan-r1 | grep become
```

### ### 🪲 Gotcha — `ansible.builtin.ping` vs `ansible.netcommon.net_ping` confusion

This trips everyone up at some point:

```bash
# ❌ Wrong — builtin.ping won't work on network devices
ansible cisco_ios -m ansible.builtin.ping
# ERROR: Python not found on device

# ✅ Correct — net_ping works on network devices
ansible cisco_ios -m ansible.netcommon.net_ping

# ✅ Correct — builtin.ping works on Linux hosts
ansible linux_hosts -m ansible.builtin.ping
```

If I accidentally use `ansible.builtin.ping` against a Cisco router, I'll see a Python-related error that looks confusing because Python does run on my control node — the error message doesn't make it clear the issue is on the remote device.

### ### 🪲 Gotcha — `-vvv` output is enormous — use `grep` to find what matters

`-vvv` output for even a simple command against a single device can be hundreds of lines. Rather than reading all of it:

```bash
# Find the actual error message
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show version'" -vvv 2>&1 | grep -i "error\|fail\|fatal\|refused\|timeout"

# Find the SSH command being used
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show version'" -vvv 2>&1 | grep "EXEC ssh"

# Find the connection establishment line
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='show version'" -vvv 2>&1 | grep "ESTABLISH"
```

### ### 🪲 Gotcha — Ad-hoc `ios_config` changes are applied but not saved

Ad-hoc `ios_config` changes go into the running config but are not written to NVRAM. A device reload will lose the change:

```bash
# ❌ This applies the change but doesn't save it
ansible wan-r1 -m cisco.ios.ios_config \
    -a "lines='description TEST'"

# ✅ Add a separate write memory command
ansible wan-r1 -m cisco.ios.ios_command \
    -a "commands='write memory'"

# Or combine in a two-step ad-hoc run
ansible wan-r1 -m cisco.ios.ios_config \
    -a "lines='description TEST' save_when=always"
```

---


Ad-hoc commands are the fast lane — quick checks, fast debugging, pre-change verification, post-change validation. From Part 10 onward, I start writing real playbooks that automate configuration tasks. The ad-hoc skills from this part become the debugging layer underneath everything else.

