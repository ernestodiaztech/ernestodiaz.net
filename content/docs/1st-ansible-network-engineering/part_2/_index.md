---
draft: false
title: '2 - Python'
description: "Part 2 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 2
---

{{< badge "Ansible" >}}
{{< badge content="Python" color="purple" >}}
{{< badge content="Linux" color="red" >}}

---

{{< lab-callout type="info" >}}
This is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. Each part will build upon the last. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.
{{< /lab-callout >}}

---

## Python 3

The goal of this part is to get comfortable enough with Python that it never slows me down when working with Ansible

Here's where Python shows up:

- **Variables and data structures in playbooks** are Python dictionaries and lists, just written in YAML syntax
- **Jinja2 filters** (used constantly in templates) are Python expressions
- **Custom filters and plugins** I might write for Ansible are Python scripts
- **`ansible-lint`**, **`yamllint`**, and most Ansible tooling is Python
- **Netmiko and NAPALM**: the libraries that underpin many network automation workflows are Python
- When Ansible throws an error, the traceback is Python. If I can't read a Python traceback, I can't debug effectively

{{< lab-callout type="info" >}}
Ansible modules are Python scripts that run on managed hosts (or locally for network devices). When I call `ios_config` in a playbook, Ansible is executing a Python script behind the scenes. Understanding Python helps me understand what those modules are actually doing.
{{< /lab-callout >}}

---

### Ubuntu 22.04

Ubuntu 22.04 ships with Python 3 already installed. I confirm what I have:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
python3 --version
pip3 --version
which python3
{{< /codeblock >}}

{{< line-explain >}}
Line 2:
: Pip is the package installer Python 3

Line 3:
: Shows the path: /usr/bin/python3
{{< /line-explain >}}

---

### Variables

Variables in Python do not need declaration, just assign.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
hostname = "R1"
mgmt_ip = "192.168.1.1"
vlan_id = 10
is_enabled = True

print(hostname)
print(type(vlan_id))
print(type(mgmt_ip))
{{< /codeblock >}}

{{< line-explain >}}
Lines 1-4:
: Python variables have no type declaration. The type is inferred from the value. Strings use single or double quotes interchangeably. In Ansible, variable names follow the same rules. Letters, numbers, and underscores only, starting with a letter.
{{< /line-explain >}}
 
In Ansible playbooks written in YAML, every variable is essentially a Python variable under the hood. When I write `vlan_id: 10` in a `vars:` block, Ansible stores it as a Python integer. When I write `hostname: "R1"`, it's a Python string. Understanding Python types helps me avoid subtle bugs like a VLAN ID being treated as a string when Ansible expects an integer.

---

### Strings and F-Strings

{{< codeblock file="Old Style" syntax="python" >}}
hostname = "R1"
interface = "GigabitEthernet0/1"
ip_address = "10.0.0.1"

print("Device: " + hostname)
{{< /codeblock >}}

---

{{< codeblock file="Modern Style" syntax="python" >}}
hostname = "R1"
interface = "GigabitEthernet0/1"
ip_address = "10.0.0.1"

print(f"Device: {hostname}, Interface: {interface}, IP: {ip_address}")
{{< /codeblock >}}

---

F-string can include expressions.

{{< codeblock lang="Python" syntax="python" >}}
vlan = 100
print(f"VLAN {vlan} is {'even' if vlan % 2 == 0 else 'odd'}")
{{< /codeblock >}}

F-strings (formatted string literals) start with `f` before the quote. Anything inside `{}` is evaluated as Python.

{{< lab-callout type="info" >}}
Jinja2 templating in Ansible uses `{{ variable_name }}` syntax. Which is conceptually identical to Python f-strings. Once I understand f-strings, Jinja2 templating feels familiar rather than foreign.
{{< /lab-callout >}}

---

### Lists

Lists are ordered collections. In Ansible, it can be lists of VLANs, lists of interfaces, lists of hosts.

{{< codeblock lang="Python" syntax="python" lines="true" >}}
vlans = [10, 20, 30, 40]
interfaces = ["GigabitEthernet0/0", "GigabitEthernet0/1", "GigabitEthernet0/2"]
nameservers = ["8.8.8.8", "8.8.4.4"]

print(vlans[0])
print(interfaces[-1])

vlans.append(50)
print(vlans)

vlans.remove(40)
print(vlans)

if 20 in vlans:
    print("VLAN 20 is in the list")

print(len(vlans))
{{< /codeblock >}}

{{< line-explain >}}
Line 5:
: Python lists are zero-indexed, the first item is at index `[0]`, not `[1]`. This catches people off guard early.

Line 6:
: Negative indexing counts from the end. `[-1]` is always the last item.
{{< /line-explain >}}

---

### Dictionaries

Dictionaries are key-value pairs. This is the most important Python data structure for Ansible: `host_vars`, `group_vars`, facts, and registered task output are all dictionaries.

{{< codeblock lang="Python" syntax="python" lines="true" >}}
router = {
    "hostname": "R1",
    "mgmt_ip": "192.168.1.1",
    "platform": "ios",
    "location": "DataCenter-A",
    "interfaces": ["GigabitEthernet0/0", "GigabitEthernet0/1"]
}

print(router["hostname"])
print(router.get("location"))

print(router.get("serial_number"))
print(router["serial_number"])

router["vendor"] = "Cisco"
router["mgmt_ip"] = "192.168.1.2"

if "platform" in router:
    print(f"Platform is: {router['platform']}")

print(router.keys())
print(router.values())
print(router.items())
{{< /codeblock >}}

{{< line-explain >}}
Line 18:
: Checks whether the key "platform" exists in the dictionary using the `in` operator.

Line 19:
: Prints the value of "platform" using an f-string if the condition evaluates to `True`.

Line 21:
: Prints all dictionary keyssing `.keys()`.

Line 22:
: Prints all dictionary values using `.values()`.

Line 23:
: Prints all key-value pairs using `.items()`.
{{< /line-explain >}}

---

**Dictionary vs YAML Syntax**

In Python I write `{"hostname": "R1", "platform": "ios"}`. In YAML (Ansible) I write the same data as:

{{< codeblock lang="YAML" syntax="yaml" >}}
hostname: R1
platform: ios
{{< /codeblock >}}

They represent identical data structures. When Ansible reads my YAML file, it converts it into Python dictionaries internally. Knowing this makes the relationship between YAML playbooks and Python crystal clear.

---

### Nested Dictionaries

Real device data is nested (dictionaries inside dictionaries). Ansible facts look exactly like this.

{{< codeblock lang="Python" syntax="python" lines="true" >}}
devices = {
    "R1": {
        "mgmt_ip": "192.168.1.1",
        "platform": "ios",
        "interfaces": {
            "GigabitEthernet0/0": {
                "ip": "10.0.0.1",
                "mask": "255.255.255.0",
                "state": "up"
            },
            "GigabitEthernet0/1": {
                "ip": "10.0.1.1",
                "mask": "255.255.255.0",
                "state": "down"
            }
        }
    },
    "R2": {
        "mgmt_ip": "192.168.1.2",
        "platform": "ios",
        "interfaces": {}
    }
}

print(devices["R1"]["mgmt_ip"])
print(devices["R1"]["interfaces"]["GigabitEthernet0/0"]["ip"])

state = devices.get("R1", {}).get("interfaces", {}).get("GigabitEthernet0/0", {}).get("state")
print(state)
{{< /codeblock >}}

{{< line-explain >}}
Line 25:
: Retrieves R1's management IP.

Line 26:
: Retrieves the IP address of R1's `GigabitEthernet0/0`.

Lines 28-29:
: This uses chained `.get()` calls with default empty dictionaries `{}`. Each level safely returns `{}` if the key does not exist. Returns "up" if everything exsists, or returns "none" if any level is missing.
{{< /line-explain >}}

---

### Loops

{{< codeblock lang="Python" syntax="python" lines="true" >}}
vlans = [10, 20, 30, 40, 50]
devices = ["R1", "R2", "SW1", "SW2"]

for vlan in vlans:
    print(f"Configuring VLAN {vlan}")

for index, device in enumerate(devices):
    print(f"{index}: {device}")

router = {"hostname": "R1", "platform": "ios", "mgmt_ip": "192.168.1.1"}

for key, value in router.items():
    print(f"{key}: {value}")

access_ports = [f"GigabitEthernet0/{i}" for i in range(0, 24)]
print(access_ports)
{{< /codeblock >}}

{{< line-explain >}}
Line 7:
: `enumerate()` gives me both the index and the value in a loop (useful when I need to know the position of an item).

Line 15:
: List comprehensions are a compact way to build lists. `range(0, 24)` generates numbers 0 through 23. This single line creates a list of 24 interface names.
{{< /line-explain >}}

---

### Conditionals

{{< codeblock lang="Python" syntax="python" lines="true" >}}
interface_state = "down"
vlan_id = 4094

if interface_state == "up":
    print("Interface is operational")
elif interface_state == "down":
    print("Interface is down — check the cable or config")
else:
    print(f"Unknown state: {interface_state}")

if vlan_id >= 1 and vlan_id <= 4094:
    print(f"VLAN {vlan_id} is a valid VLAN ID")

reserved_vlans = [1, 1002, 1003, 1004, 1005]
if vlan_id in reserved_vlans:
    print("This VLAN is reserved — do not use it")

status = "active" if interface_state == "up" else "inactive"
print(status)
{{< /codeblock >}}

{{< line-explain >}}
Lines 1-2:
: These variables drive the conditional checks that follow.

Lines 4-9:
: If the interface is "up", it prints a success message. If the interface is "down", it prints a troubleshooting message. If the state is anything else, it prints an "Unknown state" message.

Lines 11-12:
: This validates that the VLAN falls within the standard IEEE VLAN range.

Lines 14-16:
: This is a typical validation step in automation workflows.

Lines 18-19:
: Uses a one-line conditional expression. If the interface is "up", "status" becomes "active". Otherwise, it becomes "inactive".
{{< /line-explain >}}

---

### Functions

{{< codeblock lang="Python" syntax="python" lines="true" >}}
def build_description(device_name, interface_purpose, ticket_number):
    return f"{device_name} | {interface_purpose} | Ticket: {ticket_number}"

desc = build_description("R1", "Uplink-to-SW1", "CHG0012345")
print(desc)   

def get_vlan_name(vlan_id, name="unknown"):
    return f"VLAN{vlan_id}-{name}"

print(get_vlan_name(10, "MGMT"))
print(get_vlan_name(20))
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: `def` defines a function. `return` sends a value back to whoever called the function. Functions in Python are how I avoid writing the same logic in multiple places.
{{< /line-explain >}}

---

## JSON and YAML

These two formats are everywhere in network automation. REST APIs return JSON. Ansible playbooks and inventory are YAML. I need to be able to read both formats in Python and convert between them.

---

### JSON in Python

JSON looks almost identical to Python dictionaries. Python's built-in `json` module handles it.

{{< codeblock lang="Python" syntax="python" lines="true" >}}
import json

json_string = '''
{
    "device": "R1",
    "platform": "ios-xe",
    "version": "17.6.1",
    "interfaces": [
        {"name": "GigabitEthernet1", "state": "up", "ip": "10.0.0.1"},
        {"name": "GigabitEthernet2", "state": "down", "ip": "10.0.1.1"}
    ]
}
'''

device_data = json.loads(json_string)

print(device_data["device"])
print(device_data["version"])
print(device_data["interfaces"][0]["name"])

for interface in device_data["interfaces"]:
    status = "✓" if interface["state"] == "up" else "✗"
    print(f"  {status} {interface['name']}: {interface['ip']}")

output = json.dumps(device_data, indent=4)
print(output)
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: `import json` loads Python's built-in JSON module (no installation needed).

Line 15:
: `json.loads()` ("load string") parses a JSON string into a Python dictionary. The `s` matters, `json.load()` (without `s`) reads from a file.

Line 25:
: `json.dumps()` ("dump string") converts a Python dict back to a JSON string. `indent=4` pretty-prints it with 4-space indentation.
{{< /line-explain >}}

---

### Reading and Writing JSON Files

{{< codeblock lang="Python" syntax="python" lines="true" >}}
import json

device_inventory = {
    "R1": {"ip": "192.168.1.1", "platform": "ios"},
    "R2": {"ip": "192.168.1.2", "platform": "ios"},
    "SW1": {"ip": "192.168.1.10", "platform": "nxos"}
}

with open("inventory.json", "w") as f:
    json.dump(device_inventory, f, indent=4)

with open("inventory.json", "r") as f:
    loaded_inventory = json.load(f)

print(loaded_inventory["SW1"]["platform"])
{{< /codeblock >}}

{{< line-explain >}}
Line 9:
: `with open(...) as f:` is the correct way to open files in Python. The `with` block automatically closes the file when done, even if an error occurs. `"w"` means write mode, `"r"` means read mode.

Line 10:
: `json.dump()` (no `s`) writes to a file object. Contrast with `json.dumps()` which writes to a string.
{{< /line-explain >}}

---

### YAML in Python

Python doesn't include a YAML parser in its standard library, so I install `PyYAML`:

{{< codeblock lang="Bash" syntax="bash" >}}
pip3 install pyyaml
{{< /codeblock >}}

{{< codeblock lang="Python" syntax="python" lines="true" >}}
import yaml

yaml_string = """
devices:
  R1:
    mgmt_ip: 192.168.1.1
    platform: ios
    vlans:
      - 10
      - 20
      - 30
  SW1:
    mgmt_ip: 192.168.1.10
    platform: nxos
    vlans:
      - 10
      - 20
"""

data = yaml.safe_load(yaml_string)

print(data["devices"]["R1"]["platform"])
print(data["devices"]["R1"]["vlans"])

with open("group_vars/all.yml", "r") as f:
    vars_data = yaml.safe_load(f)

with open("output.yml", "w") as f:
    yaml.dump(data, f, default_flow_style=False)
{{< /codeblock >}}

{{< line-explain >}}
Line 3-18:
: A structured YAML document.

Line 20:
: Converts the YAML string into a Python dictionary.

Line 22:
: Prints "ios"

Line 23:
: Prints "[10,20,30]"

Line 25:
: Opens an external file, parses it into a Python dictionary, and stores it in `vars_data`.

Lines 28-29:
: Opens `output.yml`, writes the `data` dictionary to YAML format. `default_flow_style=False` ensures block-style YAML.
{{< /line-explain >}}

{{< lab-callout type="warning" >}}
Never use `yaml.load()` without a `Loader` argument in any script, and never use `yaml.load(data, Loader=yaml.FullLoader)` on YAML files from untrusted sources. The safe habit is `yaml.safe_load()` always, everywhere. This same principle applies when I see Ansible warning about unsafe YAML loading.
{{< /lab-callout >}}

---

### JSON vs YAML

| Situation | Format |
|---|---|
| REST API response (Netbox, AWX, PAN-OS API) | JSON |
| Ansible playbooks and inventory | YAML |
| Ansible facts gathered from devices | Python dict (originally JSON) |
| Ansible output with `-v` flag | JSON-like Python dict |
| Jinja2 template variable files | YAML |
| Network device RESTCONF responses | JSON or XML |

---

## How Ansible Uses Python

Understanding this demystifies a lot of Ansible's behavior and error messages.

### The Execution Flow

When I run `ansible-playbook site.yml`, here's what actually happens:

1. Ansible reads my YAML playbook and converts it into Python data structures (dictionaries and lists)
2. For each task, Ansible finds the corresponding Python module file (e.g., `ios_config.py`)
3. For network devices using `network_cli` connection, Ansible runs the module **locally** on the control node (my Ubuntu VM), not on the device itself
4. The module uses Paramiko or SSH to connect to the device, send commands, and receive output
5. The module returns a Python dictionary with keys like `changed`, `failed`, `stdout`, `diff`
6. Ansible evaluates the return dictionary to determine if the task passed or failed, and whether anything changed

---

This is a key difference between network automation and server automation. When Ansible manages a Linux server, it copies the Python module to the server and runs it there. When Ansible manages a network device (a router or switch), the device doesn't run Python (Ansible runs the module locally on the control node and communicates with the device over SSH or NETCONF). This is why network modules require `connection: network_cli` or `connection: netconf` instead of the default `connection: ssh`.

---

### Python Tracebacks

When something goes wrong, Ansible often surfaces a Python traceback. Here's how to read one:

{{< codeblock lang="" copy="false" >}}
TASK [ios_config] *************************************
fatal: [R1]: FAILED! => {
    "msg": "Traceback (most recent call last):\n
    File \"/usr/lib/python3/dist-packages/ansible/...\", line 142, in run\n
        connection.get('show version')\n
    File \"...\", line 87, in get\n
        raise AnsibleConnectionFailure('timed out')\n
    ansible.errors.AnsibleConnectionFailure: timed out"
}
{{< /codeblock >}}

Reading this from bottom to top is the trick. The last line is the actual error: `AnsibleConnectionFailure: timed out`. The lines above it are the call stack showing how the code got there. I almost always only need the last line to understand what went wrong.

{{< lab-callout type="tip" >}}
When I get a cryptic Ansible error, I add `-vvv` to my `ansible-playbook` command. This verbose mode prints the full Python traceback and the exact SSH commands being sent to the device. It's the fastest way to go from "something failed" to "I know exactly why."
{{< /lab-callout >}}

---

## Python Libraries

These three libraries come up constantly in network automation. I don't need to master them now but understanding what they do and how to use them in a basic script makes me a more capable engineer.

---

### Installing Libraries

All three go into my virtual environment. For now, I install them into the system Python just to experiment:

{{< codeblock lang="Bash" syntax="bash" >}}
pip3 install netmiko napalm requests --break-system-packages
{{< /codeblock >}}

{{< lab-callout type="warning" >}}
The `--break-system-packages` flag is needed on Ubuntu 22.04 because pip is restricted from modifying system packages by default. I use this flag only for quick experiments on a VM I control.
{{< /lab-callout >}}

---

### Talking to REST APIs

`requests` is the standard Python HTTP library. I use it to interact with REST APIs (Netbox, AWX, Palo Alto, etc.).

{{< codeblock lang="Python" syntax="python" lines="true" >}}
import requests
import json

response = requests.get("https://httpbin.org/get")

print(response.status_code)
print(response.headers)
print(response.json())

NETBOX_URL = "http://192.168.1.100:8000"
NETBOX_TOKEN = "my_api_token_here"

headers = {
    "Authorization": f"Token {NETBOX_TOKEN}",
    "Content-Type": "application/json"
}

response = requests.get(
    f"{NETBOX_URL}/api/dcim/devices/",
    headers=headers,
    verify=False 
)

if response.status_code == 200:
    devices = response.json()
    print(f"Found {devices['count']} devices in Netbox")
    for device in devices["results"]:
        print(f"  - {device['name']}: {device['primary_ip']['address']}")
else:
    print(f"API call failed: {response.status_code}")
    print(response.text)
{{< /codeblock >}}

{{< line-explain >}}
Line 4:
: `requests.get()` sends an HTTP GET request and returns a response object.

Line 8:
: `.json()` automatically parses the JSON response body into a Python dictionary (equivalent to calling `json.loads(response.text)`).

Line 21:
: `verify=False` disables SSL certificate verification.
{{< /line-explain >}}

---

### Netmiko

Netmiko is a Python library built specifically for SSH connections to network devices. It handles all the quirks of different vendors' SSH implementations.

{{< codeblock lang="Python" syntax="python" lines="true" >}}
from netmiko import ConnectHandler

device = {
    "device_type": "cisco_ios",
    "host": "192.168.1.1",
    "username": "ansible",
    "password": "cisco123",
    "port": 22,
}

connection = ConnectHandler(**device)

output = connection.send_command("show ip interface brief")
print(output)

config_commands = [
    "interface GigabitEthernet0/1",
    "description Uplink-to-Core",
    "no shutdown"
]

connection.send_config_set(config_commands)

# Save the configuration
connection.save_config()

# Always disconnect when done
connection.disconnect()
{{< /codeblock >}}

{{< line-explain >}}
Lines 3-9:
: The device dictionary tells Netmiko exactly what kind of device I'm connecting to. The `device_type` field is critical, it tells Netmiko which SSH behavior patterns to expect. Common values: `cisco_ios`, `cisco_nxos`, `juniper_junos`, `paloalto_panos`.

Line 11:
: `**device` unpacks the dictionary as keyword arguments (equivalent to writing `ConnectHandler(device_type="cisco_ios", host="192.168.1.1", ...)`).

Line 13:
: `send_command()` sends a single show command and returns the output as a string.

Lines 16-22:
: `send_config_set()` enters config mode, sends each command in the list, and exits config mode automatically.
{{< /line-explain >}}

Ansible's `network_cli` connection plugin is built on top of Netmiko. When I configure an Ansible playbook with `connection: network_cli`, Ansible is using Netmiko under the hood to connect to devices. Understanding Netmiko directly helps me troubleshoot connectivity issues in Ansible, because the same parameters apply: `device_type` maps to `ansible_network_os`, `username` maps to `ansible_user`, and so on.

---

### Napalm

NAPALM sits above Netmiko. Instead of sending raw CLI commands, NAPALM provides vendor-neutral Python methods that work the same way across Cisco, Juniper, and Arista.

{{< codeblock lang="Python" syntax="python" lines="true" >}}
from napalm import get_network_driver

driver = get_network_driver("ios")

device = driver(
    hostname="192.168.1.1",
    username="ansible",
    password="cisco123",
    optional_args={"port": 22}
)

device.open()

facts = device.get_facts()
print(facts)

interfaces = device.get_interfaces()
for name, details in interfaces.items():
    print(f"{name}: {'up' if details['is_up'] else 'down'}")

bgp_neighbors = device.get_bgp_neighbors()

device.close()
{{< /codeblock >}}

{{< line-explain >}}
Line 3:
: `get_network_driver("ios")` returns the Cisco IOS driver class. The same code works with `"junos"`, `"eos"` (Arista), or `"nxos"`.

Line 14:
: `get_facts()` returns a standardized dictionary with the same keys regardless of vendor. This is NAPALM's core value: I write the code once and it works everywhere.
{{< /line-explain >}}

---

Part 3 sets up Git for version control.
