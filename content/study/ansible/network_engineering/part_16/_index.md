---
draft: true
title: '16 - Paramiko'
weight: 16
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 16: Paramiko & SSH Connectivity Deep Dive

> *Every Ansible task that runs against a network device travels over SSH. When connectivity works, it's invisible. When it breaks — wrong key algorithm, host key conflict from a Containerlab redeploy, SSH timeout on a slow NOS image — the playbook stops and the error messages are often cryptic. This part covers what's actually happening under the hood, how to diagnose SSH problems systematically, and how Paramiko and raw Python scripting relate to Ansible's connectivity model.*

---

## 16.1 — What Paramiko Is and Where It Sits

Paramiko is a pure-Python implementation of the SSH protocol. When Ansible connects to a network device using `connection: network_cli`, it uses Paramiko by default — not the system `ssh` binary that a human would type at a terminal.

### Why Paramiko Instead of OpenSSH

OpenSSH (the `ssh` command) is built for interactive human sessions. It reads `~/.ssh/config`, handles multiplexing through ControlMaster, and renders terminal output for a human to read. Ansible's network_cli plugin needs something different: a Python library that opens an SSH connection, sends commands programmatically, reads back output, and detects prompts — all from within a Python process. Paramiko provides exactly that API.

```
Ansible playbook
      │
      ▼
network_cli connection plugin (Python)
      │
      ▼
Paramiko SSH client (Python SSH implementation)
      │  ← TCP port 22
      ▼
Network device (IOS-XE, NX-OS, JunOS...)
```

The entire connection lives inside the Ansible process. No `ssh` subprocess is spawned. Paramiko handles the full SSH handshake — key exchange, cipher negotiation, authentication — entirely in Python.

### What Paramiko Actually Does

When Ansible connects to `wan-r1`:

```
1. TCP connection → 172.16.0.11:22
2. SSH version exchange (SSH-2.0-Paramiko_3.x.x)
3. Key exchange algorithm negotiation (ecdh-sha2-nistp256, etc.)
4. Host key verification (checks known_hosts or skips if host_key_checking=False)
5. User authentication (password or public key)
6. Shell channel opened
7. Terminal type sent (xterm)
8. Login banner received and consumed
9. Device prompt detected using regex from the platform's terminal handler
10. Commands sent, output read until prompt returns
11. Connection held open for subsequent tasks (persistent connection)
```

Steps 9-11 are what make `network_cli` different from a plain SSH connection. The platform-specific terminal handler knows the IOS enable prompt looks like `wan-r1#` and the config mode prompt looks like `wan-r1(config)#`. It sends commands, waits for the right prompt, and returns the text between them as the task output.

### Paramiko vs OpenSSH — The Lab-Relevant Differences

The full comparison matters mainly in one scenario: when Paramiko can't negotiate a connection that OpenSSH could. This happens with older or unusual SSH server configurations on network devices.

```
Paramiko                          OpenSSH (ssh binary)
────────────────────────────────  ────────────────────────────────────────
Pure Python — no system deps      Uses system OpenSSH libraries
Supports SSHv2 only               Supports SSHv2 (SSHv1 obsolete/disabled)
Limited cipher/kex support        Full cipher suite support
No ControlMaster support          ControlMaster multiplexing available
No ~/.ssh/config parsing          Full ~/.ssh/config support
Reads ansible.cfg ssh_args        Reads ansible.cfg ssh_args + ~/.ssh/config
Used by: network_cli (default)    Used by: ansible_connection: ssh (Linux hosts)
```

The Containerlab-specific situation where this matters: some NOS images present SSH server configurations with key exchange algorithms that Paramiko's version in the current venv doesn't support. The symptom is:

```
fatal: [wan-r1]: FAILED! =>
  msg: "Failed to connect to the host via ssh:
        Unable to agree on a key exchange algorithm"
```

The fix options, in order of preference:

**Option 1 — Update Paramiko** (usually resolves algorithm mismatch):
```bash
pip install --upgrade paramiko --break-system-packages
```

**Option 2 — Force specific algorithms via `ssh_args`** in `ansible.cfg`:
```ini
[ssh_connection]
ssh_args = -o KexAlgorithms=+diffie-hellman-group14-sha1 \
           -o HostKeyAlgorithms=+ssh-rsa
```

**Option 3 — Switch to OpenSSH for specific devices** (covered in section 16.3).

---

## 16.2 — Configuring SSH in `ansible.cfg`

The `[ssh_connection]` section of `ansible.cfg` controls SSH behavior for both Paramiko and OpenSSH connections. The `[persistent_connection]` section controls the persistent connection that `network_cli` maintains between tasks.

```ini
# ansible.cfg — SSH and persistent connection settings

[defaults]
# Paramiko is the default for network_cli — no explicit setting needed
# To force OpenSSH for a specific group, set in group_vars:
# ansible_connection: ansible.netcommon.network_cli
# ansible_ssh_common_args: (OpenSSH args)

host_key_checking = False    # Lab only — must be True in production

[ssh_connection]
# These args are passed to the underlying SSH implementation
# For Paramiko, most of these are ignored — they apply to OpenSSH connections
ssh_args = -o ControlMaster=no \
           -o ControlPath=none \
           -o StrictHostKeyChecking=no \
           -o UserKnownHostsFile=/dev/null \
           -o ConnectTimeout=10 \
           -o ServerAliveInterval=30 \
           -o ServerAliveCountMax=3

# For network_cli (Paramiko) specifically:
[persistent_connection]
connect_timeout = 30       # Seconds to wait for initial SSH connection
command_timeout = 60       # Seconds to wait for a command to return output
connect_retry_timeout = 15 # Seconds between connection retry attempts
```

### The Settings That Actually Matter for Network Devices

**`connect_timeout`** — How long to wait for the TCP + SSH handshake to complete. Default is 30 seconds. Increase to 60 for slow-booting NOS images or devices over high-latency WAN links.

**`command_timeout`** — How long to wait for a command to return its prompt. Default is 60 seconds. Increase for commands that take longer — `show tech-support`, large BGP table dumps, configuration saves on flash-heavy devices.

**`connect_retry_timeout`** — How long to keep retrying a failed connection before giving up. Useful when devices restart during a playbook run.

---

## 16.3 — `host_key_checking` and `known_hosts` Management

### The Problem in the Lab

Every time Containerlab redeploys a topology, new container instances are created. New containers generate new SSH host keys. But `~/.ssh/known_hosts` on the Ubuntu VM still has the old host keys from the previous deployment. The result:

```
fatal: [wan-r1]: FAILED! =>
  msg: "Failed to connect to the host via ssh:
        WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
        IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!"
```

This is SSH's man-in-the-middle protection doing exactly what it's designed to do — refusing to connect to a host whose key changed unexpectedly. In production, this is correct and should be investigated. In a lab where containers are regularly torn down and rebuilt, it's an operational friction point.

### The Lab Solution: Disable Host Key Checking

```ini
# ansible.cfg
[defaults]
host_key_checking = False
```

This tells Paramiko to skip host key verification entirely — connect regardless of whether the key matches or is new. Combined with:

```ini
[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
```

The `/dev/null` for `UserKnownHostsFile` means known hosts are never written to disk — no accumulation of stale entries.

### ### 🔴 Danger

> `host_key_checking = False` is **only** appropriate for an isolated lab environment. In production or any environment with real devices, host key checking must be enabled. A changed host key on a production device is a security event that warrants investigation — either the device was replaced, its SSH keys were regenerated, or (worst case) someone has intercepted the connection. Never disable host key checking in production `ansible.cfg`. If production and lab share an `ansible.cfg`, use separate config files: set the lab config via `ANSIBLE_CONFIG=lab_ansible.cfg ansible-playbook ...` or maintain separate project directories.

### Manually Clearing Stale Host Keys

When I need to clear specific entries without disabling checking entirely:

```bash
# Remove a specific host's key from known_hosts
ssh-keygen -R 172.16.0.11

# Remove all Containerlab management network entries at once
# (assuming lab uses 172.16.0.0/24 for management)
grep -v "172.16.0\." ~/.ssh/known_hosts > /tmp/known_hosts_clean
mv /tmp/known_hosts_clean ~/.ssh/known_hosts

# Or use a script that runs before each Containerlab deploy
cat > ~/scripts/clear_lab_known_hosts.sh << 'EOF'
#!/bin/bash
# Clear known_hosts entries for the lab management subnet
# Run after: sudo clab destroy && sudo clab deploy
sed -i '/172\.16\.0\./d' ~/.ssh/known_hosts
echo "Cleared Containerlab host keys from known_hosts"
EOF
chmod +x ~/scripts/clear_lab_known_hosts.sh
```

### Pre-Populating Known Hosts (Production Pattern)

In production where host key checking must be enabled, I pre-populate `known_hosts` using `ssh-keyscan` during device onboarding:

```bash
# Scan a device and add its host key to known_hosts
ssh-keyscan -H 192.168.1.1 >> ~/.ssh/known_hosts

# Scan multiple devices and add all at once
for ip in 192.168.1.{1..20}; do
    ssh-keyscan -H $ip 2>/dev/null
done >> ~/.ssh/known_hosts

# Or use an Ansible task during device onboarding
- name: "Onboard | Add device host key to known_hosts"
  ansible.builtin.known_hosts:
    name: "{{ ansible_host }}"
    state: present
    key: "{{ lookup('pipe', 'ssh-keyscan -H ' + ansible_host) }}"
  delegate_to: localhost
```

---

## 16.4 — SSH Troubleshooting Methodology

When a playbook fails with an SSH-related error, I work through these steps in order. The methodology applies regardless of device vendor or NOS.

### Step 1 — Confirm Basic TCP Reachability

Before debugging SSH, confirm the device is reachable at the TCP level:

```bash
# Test TCP connectivity to port 22
nc -zv 172.16.0.11 22
# Expected: Connection to 172.16.0.11 22 port [tcp/ssh] succeeded!
# If this fails: routing/firewall issue, not SSH

# Or use nmap
nmap -p 22 172.16.0.11
# Expected: 22/tcp open  ssh
```

If TCP port 22 is unreachable, the problem is network connectivity — routing, firewall rules, or the device not listening on port 22. No amount of SSH configuration will fix a TCP connectivity problem.

### Step 2 — Test Manual SSH Connection

Try connecting manually with the same credentials Ansible would use:

```bash
# Basic SSH attempt — shows the full handshake and error
ssh -v ansible@172.16.0.11

# Force the exact settings Ansible uses
ssh -v \
    -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=/dev/null \
    -o ConnectTimeout=10 \
    ansible@172.16.0.11
```

The `-v` flag shows exactly where the connection fails — key exchange, authentication, or after login. Match the error to the categories below.

### Step 3 — Run Ansible with Maximum Verbosity

```bash
ansible wan-r1 -m ansible.netcommon.net_ping -vvv 2>&1 | head -100
```

The `-vvv` output for a network_cli connection shows:
```
<wan-r1> SSH: EXEC ssh ... -o ... ansible@172.16.0.11
<wan-r1> SSH: ESTABLISH SSH CONNECTION FOR USER: ansible
<wan-r1> Paramiko: CONNECT: wan-r1:22 AS ansible
<wan-r1> Paramiko: KEY EXCHANGE COMPLETE
<wan-r1> Paramiko: AUTHENTICATION complete
<wan-r1> network_cli: TERMINAL PROMPT: wan-r1>
<wan-r1> network_cli: ENABLE: sending enable command
<wan-r1> network_cli: TERMINAL PROMPT: wan-r1#
```

A failure shows which step stopped. Key exchange failure means cipher/algorithm mismatch. Authentication failure means credentials. Prompt detection failure means the terminal handler can't find the expected prompt pattern.

### Common SSH Error Patterns and Fixes

**Error: "Unable to negotiate with ... no matching key exchange method"**
```bash
# The device only offers older kex algorithms Paramiko doesn't support by default
# Diagnosis:
ssh -vv ansible@172.16.0.11 2>&1 | grep -i "kex\|key exchange"

# Fix option 1 — update Paramiko
pip install --upgrade paramiko --break-system-packages

# Fix option 2 — add legacy algorithms to ssh_args in ansible.cfg
# [ssh_connection]
# ssh_args = -o KexAlgorithms=+diffie-hellman-group14-sha1,diffie-hellman-group1-sha1
```

**Error: "Authentication failed"**
```bash
# Wrong username or password — or the device requires key auth
# Diagnosis:
ssh -v ansible@172.16.0.11
# Look for: "Authentications that can continue: publickey" (key-only device)
# or: "Permission denied, please try again" (wrong password)

# Check what ansible_password resolves to (confirms vault is working)
ansible-inventory --host wan-r1 | grep ansible_password

# Test with explicit password flag
sshpass -p 'ansible123' ssh ansible@172.16.0.11
```

**Error: "REMOTE HOST IDENTIFICATION HAS CHANGED"**
```bash
# Host key mismatch — most common after Containerlab redeploy
# Fix:
ssh-keygen -R 172.16.0.11

# Or for lab use, set in ansible.cfg:
# host_key_checking = False
```

**Error: "Timeout waiting for privilege escalation prompt"**
```bash
# Device isn't responding to the enable command with the expected prompt
# Common causes:
# 1. Wrong ansible_become_password
# 2. Device requires "enable" password but none is configured (just press Enter)
# 3. Prompt pattern doesn't match what network_cli expects

# Test enable manually
ssh ansible@172.16.0.11
# After connecting: type "enable" and observe what prompt appears

# If the enable password is blank:
# ansible_become_password: ""   in group_vars
```

**Error: "Command timeout triggered"**
```bash
# A command took longer than command_timeout (default 60 seconds)
# Common on: show tech-support, large show bgp, write memory on slow flash

# Fix: increase command_timeout in ansible.cfg
# [persistent_connection]
# command_timeout = 120
```

**Error: "Connection closed by remote host"**
```bash
# Device closed the connection — usually a failed login attempt triggering lockout
# or the SSH session limit being reached on the device

# Check if you're locked out:
ssh -v ansible@172.16.0.11
# "Connection closed" immediately = login failure lockout or ACL block

# On IOS — check the SSH session limit
# show ip ssh    ← shows max sessions
# show users     ← shows current sessions (zombie sessions from failed plays)

# Clear zombie sessions on IOS:
# configure terminal
# clear line vty 0
# clear line vty 1
# ... etc
```

**Containerlab-Specific: Slow SSH on First Connection**

Some NOS images in Containerlab are slow to complete the SSH handshake on first connect after boot — the image needs time to generate its host keys or initialize the SSH daemon. The symptom is a timeout even though the device is up:

```bash
# Check if SSH daemon is listening yet
watch -n 2 "nc -zv 172.16.0.11 22 2>&1"
# Wait until you see "succeeded" before running Ansible

# Increase connect_timeout for lab environments
# [persistent_connection]
# connect_timeout = 60    # Give slow NOS images more time
```

### Step 4 — Isolate to a Single Device and Task

Once the error category is identified, isolate the problem:

```bash
# Test connectivity module only — no configuration tasks
ansible wan-r1 -m ansible.netcommon.net_ping -vvv

# Test one specific task from the playbook in ad-hoc mode
ansible wan-r1 -m cisco.ios.ios_command -a "commands='show version'" -vvv

# If all devices fail the same way: global config issue (ansible.cfg, vault)
# If only one device fails: device-specific issue (host_vars, network config, device state)
```

---

## 16.5 — Using Paramiko and Netmiko Directly in Python

Understanding how Paramiko works at the Python level is useful when I need to write automation that falls outside what Ansible modules provide — emergency scripts, one-off bulk operations, or tools that need more control over the SSH session than Ansible exposes.

### Raw Paramiko — What Ansible Uses Internally

```python
#!/usr/bin/env python3
"""
paramiko_example.py — Direct Paramiko connection to a network device
Shows what Ansible's network_cli plugin does under the hood.
"""

import paramiko
import time
import re

def connect_ios(host, username, password, enable_password=None):
    """Open a Paramiko SSH connection to an IOS device."""

    # Create SSH client
    client = paramiko.SSHClient()

    # Disable host key checking (lab only — use AutoAddPolicy or RejectPolicy in production)
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    # Connect
    client.connect(
        hostname=host,
        port=22,
        username=username,
        password=password,
        timeout=30,
        look_for_keys=False,    # Don't try SSH key auth
        allow_agent=False,      # Don't use SSH agent
    )

    # Open an interactive shell channel
    shell = client.invoke_shell(
        term='xterm',
        width=512,
        height=512,
    )

    time.sleep(1)    # Wait for the initial prompt to appear

    # Read and discard the login banner and initial prompt
    if shell.recv_ready():
        shell.recv(65535)

    # Enter enable mode if password provided
    if enable_password:
        shell.send('enable\n')
        time.sleep(0.5)
        shell.send(enable_password + '\n')
        time.sleep(0.5)
        if shell.recv_ready():
            shell.recv(65535)    # Consume the enable prompt

    # Disable pagination
    shell.send('terminal length 0\n')
    time.sleep(0.5)
    if shell.recv_ready():
        shell.recv(65535)

    return client, shell


def send_command(shell, command, wait=1.0):
    """Send a command and return the output."""
    shell.send(command + '\n')
    time.sleep(wait)

    output = ''
    while shell.recv_ready():
        output += shell.recv(65535).decode('utf-8', errors='replace')
        time.sleep(0.1)

    return output


def main():
    host = '172.16.0.11'
    username = 'ansible'
    password = 'ansible123'
    enable_password = 'ansible123'

    print(f"Connecting to {host}...")
    client, shell = connect_ios(host, username, password, enable_password)

    # Run show commands
    version_output = send_command(shell, 'show version')
    print("=== show version ===")
    print(version_output[:500])    # First 500 chars

    interfaces_output = send_command(shell, 'show ip interface brief')
    print("=== show ip interface brief ===")
    print(interfaces_output)

    # Close connection
    client.close()
    print("Connection closed.")


if __name__ == '__main__':
    main()
```

This works — but look at the complexity: manually handling the enable prompt, manually stripping pagination, manually timing the reads with `time.sleep()`, manually decoding bytes. The output contains the command echo, the prompt, and the actual output all mixed together. Parsing it reliably is difficult.

This is exactly the problem Netmiko was built to solve.

### Netmiko — Paramiko with Network Device Intelligence

Netmiko is a library built on top of Paramiko specifically for network device automation. It handles everything raw Paramiko requires manual work for:

```
Raw Paramiko                          Netmiko
────────────────────────────────────  ────────────────────────────────────────
Manual prompt detection               Automatic — knows IOS/NX-OS/JunOS prompts
Manual enable mode handling           Automatic — handles enable and config mode
Manual pagination disabling           Automatic — sends terminal length 0
Manual output buffering               Automatic — reads until prompt, returns clean text
Manual command echo stripping         Automatic — removes command from output
Manual timing/sleep calls             Automatic — uses expect-style prompt detection
No device type awareness              150+ supported device types built in
```

The same operation in Netmiko:

```python
#!/usr/bin/env python3
"""
netmiko_example.py — Netmiko connection to a network device
Compare with paramiko_example.py to see the difference.
"""

from netmiko import ConnectHandler
import json

def main():
    # Device definition — this structure mirrors Ansible's host_vars
    device = {
        'device_type': 'cisco_ios',    # Netmiko device type
        'host': '172.16.0.11',
        'username': 'ansible',
        'password': 'ansible123',
        'secret': 'ansible123',        # Enable password
        'timeout': 30,
        'session_log': 'netmiko_session.log',    # Optional — logs full session
    }

    print(f"Connecting to {device['host']}...")

    # ConnectHandler manages the full connection lifecycle
    with ConnectHandler(**device) as net_connect:

        # Enter enable mode automatically
        net_connect.enable()

        # Run show commands — output is clean, prompt and echo stripped
        version_output = net_connect.send_command('show version')
        print("=== show version (first 300 chars) ===")
        print(version_output[:300])

        interfaces_output = net_connect.send_command(
            'show ip interface brief',
            use_textfsm=True    # Parse with TextFSM — returns list of dicts
        )
        print("\n=== show ip interface brief (parsed) ===")
        for iface in interfaces_output:
            print(f"  {iface['intf']:<25} {iface['ipaddr']:<18} {iface['status']}")

        # Enter config mode and push commands
        config_commands = [
            'interface Loopback99',
            'description Netmiko test interface',
            'ip address 10.99.0.1 255.255.255.255',
            'no shutdown',
        ]
        config_output = net_connect.send_config_set(config_commands)
        print(f"\n=== Config output ===")
        print(config_output)

        # Save configuration
        save_output = net_connect.save_config()
        print(f"\n=== Save output ===")
        print(save_output)

    # Connection closed automatically by context manager
    print("\nConnection closed.")


if __name__ == '__main__':
    main()
```

Install Netmiko:
```bash
pip install netmiko --break-system-packages
```

### When to Use Each Tool

| Task | Best Tool | Reason |
|---|---|---|
| Standard playbook automation | Ansible + network_cli | Idempotent, declarative, version-controlled |
| Bulk show command collection | Netmiko | Cleaner than ad-hoc, easier to parse output |
| Emergency one-off fix | Netmiko | Faster to write than a playbook for a one-time task |
| Custom parsing and logic | Netmiko | Full Python — can use any library for output processing |
| Automation outside Ansible context | Netmiko | When Ansible isn't installed or not appropriate |
| Raw protocol investigation | Paramiko | When I need to control exactly what bytes are sent |
| REST API devices (RESTCONF, YANG) | requests library | Not SSH at all — different tool entirely |

### The Relationship to Ansible

Ansible's `network_cli` plugin is essentially a managed version of Paramiko — it does what the raw Paramiko example above does, but with platform-specific handlers for each NOS, retry logic, persistent connection management, and integration with the Ansible variable and module system.

Netmiko serves a different niche: Python scripts that need network device access without the full Ansible framework. Many network automation workflows use both — Ansible for infrastructure-as-code configuration management, and Netmiko for operational scripts that need flexibility Ansible doesn't provide (real-time output streaming, complex conditional logic based on parsed output, integration with ticketing systems or CMDBs).

---

## 16.6 — Advanced `ssh_args` Tuning

The `ssh_args` setting in `ansible.cfg` passes additional options to the underlying SSH implementation. For `network_cli` connections using Paramiko, most standard OpenSSH options don't apply — but some do via Paramiko's own configuration:

```ini
[ssh_connection]
# These apply to OpenSSH-based connections (Linux hosts, ansible_connection: ssh)
ssh_args = -o ControlMaster=no \
           -o ControlPath=none \
           -o StrictHostKeyChecking=no \
           -o UserKnownHostsFile=/dev/null \
           -o ConnectTimeout=10 \
           -o ServerAliveInterval=30 \
           -o ServerAliveCountMax=3 \
           -o TCPKeepAlive=yes
```

### Per-Host SSH Arguments

For devices that need special SSH settings (older cipher suites, non-standard port):

```yaml
# host_vars/legacy-switch.yml
ansible_host: 10.0.0.1
ansible_port: 2222                              # Non-standard SSH port
ansible_ssh_common_args: >-
  -o KexAlgorithms=+diffie-hellman-group1-sha1
  -o HostKeyAlgorithms=+ssh-rsa
  -o Ciphers=+aes128-cbc,3des-cbc
```

### SSH Key Authentication for Network Devices

Using SSH keys instead of passwords (the production preference):

```bash
# Generate a dedicated key pair for Ansible
ssh-keygen -t ed25519 -C "ansible@automation" -f ~/.ssh/ansible_ed25519 -N ""

# Copy public key to IOS device (requires password auth first)
# On the device:
# configure terminal
# ip ssh pubkey-chain
#   username ansible
#     key-string
#       <paste the content of ~/.ssh/ansible_ed25519.pub>
#     exit
#   exit
# exit
```

```yaml
# group_vars/all.yml
ansible_ssh_private_key_file: ~/.ssh/ansible_ed25519
ansible_password: ""    # Empty — key auth, no password
```

```ini
# ansible.cfg
[defaults]
private_key_file = ~/.ssh/ansible_ed25519
```

SSH key authentication eliminates password exposure in vault files and is significantly more secure — particularly relevant for production environments where the Ansible control node itself may be shared.

---


SSH connectivity is now fully understood — what's happening at the protocol level, how to diagnose failures systematically, and how Paramiko relates to Netmiko and Ansible. Part 17 moves into the full OSPF and BGP automation workflow — configuring, verifying, and validating dynamic routing using resource modules.

