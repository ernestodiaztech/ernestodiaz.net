---
draft: false
title: '2 - Python'
weight: 2
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.

---

## Python 3

When I first started looking at Ansible, I assumed I could avoid Python entirely (after all, playbooks are just YAML). That assumption breaks down fast. The goal of this part is to get comfortable enough with Python that it never slows me down when working with Ansible

Here's where Python shows up whether I plan for it or not:

- **Variables and data structures in playbooks** are Python dictionaries and lists, just written in YAML syntax
- **Jinja2 filters** (used constantly in templates) are Python expressions
- **Custom filters and plugins** I might write for Ansible are Python scripts
- **`ansible-lint`**, **`yamllint`**, and most Ansible tooling is Python
- **Netmiko and NAPALM**: the libraries that underpin many network automation workflows are Python
- When Ansible throws an error, the traceback is Python. If I can't read a Python traceback, I can't debug effectively

> [!Note]
> Ansible modules are Python scripts that run on managed hosts (or locally for network devices). When I call `ios_config` in a playbook, Ansible is executing a Python script behind the scenes. Understanding Python helps me understand what those modules are actually doing.

---
{{% steps %}}

<h4>Ubuntu 22.04</h4>

Ubuntu 22.04 ships with Python 3 already installed. Before writing a single line of code, I confirm what I have:

```bash
python3 --version
pip3 --version
which python3
```

- `pip3 --version` - Pip is the package installer Python 3
- `which python3` - Shows the path: /usr/bin/python3

---

<h4>Variables</h4>

Variables in Python do not need declaration, just assign.

```python {linenos=table}
hostname = "R1"
mgmt_ip = "192.168.1.1"
vlan_id = 10
is_enabled = True

print(hostname)
print(type(vlan_id))
print(type(mgmt_ip))
```

**Line 1-4:** Python variables have no type declaration. The type is inferred from the value. Strings use single or double quotes interchangeably. In Ansible, variable names follow the same rules. Letters, numbers, and underscores only, starting with a letter.

> [!Note] 
> In Ansible playbooks written in YAML, every variable is essentially a Python variable under the hood. When I write `vlan_id: 10` in a `vars:` block, Ansible stores it as a Python integer. When I write `hostname: "R1"`, it's a Python string. Understanding Python types helps me avoid subtle bugs like a VLAN ID being treated as a string when Ansible expects an integer.

---

<h4>Strings and F-Strings</h4>

Old style.

```python
hostname = "R1"
interface = "GigabitEthernet0/1"
ip_address = "10.0.0.1"

print("Device: " + hostname)
```

Modern style.

```python
hostname = "R1"
interface = "GigabitEthernet0/1"
ip_address = "10.0.0.1"

print(f"Device: {hostname}, Interface: {interface}, IP: {ip_address}")
```

F-string can include expressions.

```python
vlan = 100
print(f"VLAN {vlan} is {'even' if vlan % 2 == 0 else 'odd'}")
```

F-strings (formatted string literals) start with `f` before the quote. Anything inside `{}` is evaluated as Python.

>[!Tip]
> Jinja2 templating in Ansible uses `{{ variable_name }}` syntax. Which is conceptually identical to Python f-strings. Once I understand f-strings, Jinja2 templating feels familiar rather than foreign.

---

<h4>Lists</h4>

Lists are ordered collections. In Ansible, it can be lists of VLANs, lists of interfaces, lists of hosts.

```python  {linenos=table}
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
```

- **Line 6:** Python lists are zero-indexed, the first item is at index `[0]`, not `[1]`. This catches people off guard early.
- **Line 7:** Negative indexing counts from the end. `[-1]` is always the last item.

---

<h4>Dictionaries</h4>

Dictionaries are key-value pairs. This is the most important Python data structure for Ansible: `host_vars`, `group_vars`, facts, and registered task output are all dictionaries.

```python  {linenos=table}
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
```

- **Line 14 vs 16:** I almost always use `.get()` instead of direct key access in scripts. Direct access crashes with a `KeyError` if the key doesn't exist. `.get()` returns `None` (or a default value I specify). Much safer when working with device data that might be incomplete.

> [!Note] Dictionary vs YAML Syntax
> In Python I write `{"hostname": "R1", "platform": "ios"}`. In YAML (Ansible) I write the same data as:
> ```yaml
> hostname: R1
> platform: ios
> ```
> They represent identical data structures. When Ansible reads my YAML file, it converts it into Python dictionaries internally. Knowing this makes the relationship between YAML playbooks and Python crystal clear.

---

<h4>Nested Dictionaries</h4>

Real device data is nested (dictionaries inside dictionaries). Ansible facts look exactly like this.

```python {linenos=table}
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
```

- **Line 26:** Chaining `.get()` calls with empty dict defaults `{}` lets me safely drill into deeply nested data without risking a `KeyError` at any level. This pattern comes up constantly when processing Ansible facts.

---

<h4>Loops</h4>

```python {linenos=table}
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
```

- **Line 9:** `enumerate()` gives me both the index and the value in a loop (useful when I need to know the position of an item).
- **Line 20:** List comprehensions are a compact way to build lists. `range(0, 24)` generates numbers 0 through 23. This single line creates a list of 24 interface names.

---

<h4>Conditionals</h4>

```python {linenos=table}
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
```

- **Line 12:** `and` / `or` / `not` are Python's logical operators. In Ansible's `when:` conditions, I use the same logic.

---

<h4>Functions</h4>

```python {linenos=table}
def build_description(device_name, interface_purpose, ticket_number):
    return f"{device_name} | {interface_purpose} | Ticket: {ticket_number}"

desc = build_description("R1", "Uplink-to-SW1", "CHG0012345")
print(desc)   

def get_vlan_name(vlan_id, name="unknown"):
    return f"VLAN{vlan_id}-{name}"

print(get_vlan_name(10, "MGMT"))
print(get_vlan_name(20))
```

- **Line 3:** `def` defines a function. `return` sends a value back to whoever called the function. Functions in Python are how I avoid writing the same logic in multiple places.

{{% /steps %}}

---

## JSON and YAML

These two formats are everywhere in network automation. REST APIs return JSON. Ansible playbooks and inventory are YAML. I need to be able to read both formats in Python and convert between them.

{{% steps %}}

---

<h4>JSON in Python</h4>

JSON (JavaScript Object Notation) looks almost identical to Python dictionaries. Python's built-in `json` module handles it.

```python {linenos=table}
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
```

- **Line 1:** `import json` loads Python's built-in JSON module (no installation needed).
- **Line 17:** `json.loads()` ("load string") parses a JSON string into a Python dictionary. The `s` matters, `json.load()` (without `s`) reads from a file.
- **Line 30:** `json.dumps()` ("dump string") converts a Python dict back to a JSON string. `indent=4` pretty-prints it with 4-space indentation.

---

<h4>Reading and Writing JSON Files</h4>

```python {linenos=table}
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

print(loaded_inventory["SW1"]["platform"])    # nxos
```

- **Line 10:** `with open(...) as f:` is the correct way to open files in Python. The `with` block automatically closes the file when done, even if an error occurs. `"w"` means write mode, `"r"` means read mode.
- **Line 11:** `json.dump()` (no `s`) writes to a file object. Contrast with `json.dumps()` which writes to a string.

---

<h4>YAML in Python</h4>

Python doesn't include a YAML parser in its standard library, so I install `PyYAML`:

```bash
pip3 install pyyaml
```

```python {linenos=table}
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
```

- **Line 21:** `yaml.safe_load()` is always preferred over `yaml.load()`. The unsafe `yaml.load()` can execute arbitrary Python code embedded in a YAML file. `safe_load()` only parses data, never executes code.

> [!CAUTION]
> Never use `yaml.load()` without a `Loader` argument in any script, and never use `yaml.load(data, Loader=yaml.FullLoader)` on YAML files from untrusted sources. The safe habit is `yaml.safe_load()` always, everywhere. This same principle applies when I see Ansible warning about unsafe YAML loading.

---

<h4>JSON vs YAML</h4>

| Situation | Format |
|---|---|
| REST API response (Netbox, AWX, PAN-OS API) | JSON |
| Ansible playbooks and inventory | YAML |
| Ansible facts gathered from devices | Python dict (originally JSON) |
| Ansible output with `-v` flag | JSON-like Python dict |
| Jinja2 template variable files | YAML |
| Network device RESTCONF responses | JSON or XML |

{{% /steps %}}

---

## How Ansible Uses Python

Understanding this demystifies a lot of Ansible's behavior and error messages.

{{% steps %}}

<h4>The Execution Flow</h4>

When I run `ansible-playbook site.yml`, here's what actually happens:

1. Ansible reads my YAML playbook and converts it into Python data structures (dictionaries and lists)
2. For each task, Ansible finds the corresponding Python module file (e.g., `ios_config.py`)
3. For network devices using `network_cli` connection, Ansible runs the module **locally** on the control node (my Ubuntu VM), not on the device itself
4. The module uses Paramiko or SSH to connect to the device, send commands, and receive output
5. The module returns a Python dictionary with keys like `changed`, `failed`, `stdout`, `diff`
6. Ansible evaluates the return dictionary to determine if the task passed or failed, and whether anything changed


> [!Note]
> This is a key difference between network automation and server automation. When Ansible manages a Linux server, it copies the Python module to the server and runs it there. When Ansible manages a network device (a router or switch), the device doesn't run Python (Ansible runs the module locally on the control node and communicates with the device over SSH or NETCONF). This is why network modules require `connection: network_cli` or `connection: netconf` instead of the default `connection: ssh`.

---

<h4>Python Tracebacks (Reading Error Messages)</h4>

When something goes wrong, Ansible often surfaces a Python traceback. Here's how to read one:

```
TASK [ios_config] *************************************
fatal: [R1]: FAILED! => {
    "msg": "Traceback (most recent call last):\n
    File \"/usr/lib/python3/dist-packages/ansible/...\", line 142, in run\n
        connection.get('show version')\n
    File \"...\", line 87, in get\n
        raise AnsibleConnectionFailure('timed out')\n
    ansible.errors.AnsibleConnectionFailure: timed out"
}
```

Reading this from **bottom to top** is the trick. The last line is the actual error: `AnsibleConnectionFailure: timed out`. The lines above it are the call stack showing how the code got there. I almost always only need the last line to understand what went wrong.

> [!Tip]
> When I get a cryptic Ansible error, I add `-vvv` to my `ansible-playbook` command. This verbose mode prints the full Python traceback and the exact SSH commands being sent to the device. It's the fastest way to go from "something failed" to "I know exactly why."

{{% /steps %}}

---

## Python Libraries

These three libraries come up constantly in network automation. I don't need to master them now but understanding what they do and how to use them in a basic script makes me a more capable engineer.

---

{{% steps %}}

<h4>Installing Libraries</h4>

All three go into my virtual environment (covered in Part 3). For now, I install them into the system Python just to experiment:

```bash
pip3 install netmiko napalm requests --break-system-packages
```

> [!Warning]
> The `--break-system-packages` flag is needed on Ubuntu 22.04 because pip is restricted from modifying system packages by default. I use this flag only for quick experiments on a VM I control. In Part 3, I set up a virtual environment and never need this flag again — virtualenvs are the correct solution.

---

<h4>Talking to REST APIs</h4>

`requests` is the standard Python HTTP library. I use it to interact with REST APIs (Netbox, AWX, Palo Alto, etc.).

```python {linenos=table}
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
```

- **Line 6:** `requests.get()` sends an HTTP GET request and returns a response object.
- **Line 10:** `.json()` automatically parses the JSON response body into a Python dictionary (equivalent to calling `json.loads(response.text)`).
- **Line 24:** `verify=False` disables SSL certificate verification.

---

> [!Note]
> In enterprise environments, the Netbox API is one of the most common things I'll hit with `requests`. Pulling device inventory, IP address data, or VLAN assignments from Netbox via its API is a core part of building dynamic Ansible inventories. The Netbox Ansible collection handles this automatically in later parts, but knowing the raw `requests` call helps me debug when the collection behaves unexpectedly.

---

<h4>Netmiko</h4>

Netmiko is a Python library built specifically for SSH connections to network devices. It handles all the quirks of different vendors' SSH implementations.

```python {linenos=table}
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
```

- **Line 4-10:** The device dictionary tells Netmiko exactly what kind of device I'm connecting to. The `device_type` field is critical, it tells Netmiko which SSH behavior patterns to expect. Common values: `cisco_ios`, `cisco_nxos`, `juniper_junos`, `paloalto_panos`.
- **Line 13:** `**device` unpacks the dictionary as keyword arguments (equivalent to writing `ConnectHandler(device_type="cisco_ios", host="192.168.1.1", ...)`).
- **Line 16:** `send_command()` sends a single show command and returns the output as a string.
- **Line 19-25:** `send_config_set()` enters config mode, sends each command in the list, and exits config mode automatically.

> [!Info]
> Ansible's `network_cli` connection plugin is built on top of Netmiko. When I configure an Ansible playbook with `connection: network_cli`, Ansible is using Netmiko under the hood to connect to devices. Understanding Netmiko directly helps me troubleshoot connectivity issues in Ansible, because the same parameters apply: `device_type` maps to `ansible_network_os`, `username` maps to `ansible_user`, and so on.

---

<h4>Napalm</h4>

NAPALM (Network Automation and Programmability Abstraction Layer with Multivendor support) sits above Netmiko. Instead of sending raw CLI commands, NAPALM provides vendor-neutral Python methods that work the same way across Cisco, Juniper, and Arista.

```python {linenos=table}
from napalm import get_network_driver

# Get the driver for Cisco IOS
driver = get_network_driver("ios")

# Define connection parameters
device = driver(
    hostname="192.168.1.1",
    username="ansible",
    password="cisco123",
    optional_args={"port": 22}
)

# Open the connection
device.open()

# Get facts — same method name works for Cisco, Juniper, Arista
facts = device.get_facts()
print(facts)
# Returns a standardized dictionary regardless of vendor:
# {
#   "hostname": "R1",
#   "fqdn": "R1.lab.local",
#   "vendor": "Cisco",
#   "model": "CSR1000V",
#   "os_version": "17.06.01",
#   "serial_number": "...",
#   "uptime": 86400,
#   "interface_list": ["GigabitEthernet1", "GigabitEthernet2"]
# }

# Get interfaces
interfaces = device.get_interfaces()
for name, details in interfaces.items():
    print(f"{name}: {'up' if details['is_up'] else 'down'}")

# Get BGP neighbors
bgp_neighbors = device.get_bgp_neighbors()

# Close the connection
device.close()
```

- **Line 3:** `get_network_driver("ios")` returns the Cisco IOS driver class. The same code works with `"junos"`, `"eos"` (Arista), or `"nxos"`.
- **Line 17:** `get_facts()` returns a standardized dictionary with the same keys regardless of vendor. This is NAPALM's core value: I write the code once and it works everywhere.

> [!Tip]
> For Ansible work, I won't use Netmiko and NAPALM directly very often — the Ansible collections (`cisco.ios`, `junipernetworks.junos`) wrap this functionality in modules. But I keep these libraries in my toolkit because there are times when I need to do something Ansible doesn't have a module for. In those cases, I can write a quick Python script using Netmiko or NAPALM, or write a custom Ansible module.

{{% /steps %}}

---

## Hands-On Exercises

These exercises run entirely on my Ubuntu VM. I create a working directory and build up a small set of scripts that I'll reference again in later parts.

```bash
mkdir -p ~/projects/python-practice
cd ~/projects/python-practice
```

{{% steps %}}

<h4>Device Inventory Script</h4>

I'll create a script that stores a small device inventory as a Python data structure, then reads, filters, and displays it.

Create the file `~/projects/python-practice/inventory.py`:

```python {linenos=table}
#!/usr/bin/env python3
"""
Exercise 1: Working with a device inventory as a Python data structure.
This mirrors exactly how Ansible stores host_vars internally.
"""

# Device inventory — a list of dictionaries
# Each dictionary represents one device, just like a host_vars file
devices = [
    {
        "hostname": "R1",
        "mgmt_ip": "192.168.1.1",
        "platform": "ios",
        "role": "router",
        "site": "HQ",
        "vlans": [10, 20, 30]
    },
    {
        "hostname": "R2",
        "mgmt_ip": "192.168.1.2",
        "platform": "ios",
        "role": "router",
        "site": "Branch",
        "vlans": [10, 40]
    },
    {
        "hostname": "SW1",
        "mgmt_ip": "192.168.1.10",
        "platform": "nxos",
        "role": "switch",
        "site": "HQ",
        "vlans": [10, 20, 30, 40, 50]
    },
    {
        "hostname": "FW1",
        "mgmt_ip": "192.168.1.20",
        "platform": "panos",
        "role": "firewall",
        "site": "HQ",
        "vlans": [10, 20]
    }
]

# --- Task 1: Print all devices ---
print("=" * 40)
print("All Devices:")
print("=" * 40)
for device in devices:
    print(f"  {device['hostname']:<6} | {device['mgmt_ip']:<16} | {device['platform']:<6} | {device['site']}")

# --- Task 2: Filter by site ---
print("\nHQ Devices only:")
hq_devices = [d for d in devices if d["site"] == "HQ"]
for device in hq_devices:
    print(f"  {device['hostname']}")

# --- Task 3: Filter by platform ---
print("\nIOS Devices only:")
ios_devices = [d for d in devices if d["platform"] == "ios"]
for device in ios_devices:
    print(f"  {device['hostname']}: {device['mgmt_ip']}")

# --- Task 4: Find all unique VLANs across the entire inventory ---
all_vlans = set()
for device in devices:
    all_vlans.update(device["vlans"])
print(f"\nAll VLANs in use: {sorted(all_vlans)}")

# --- Task 5: Write the inventory to a JSON file ---
import json
with open("device_inventory.json", "w") as f:
    json.dump(devices, f, indent=4)
print("\nInventory written to device_inventory.json")
```

- **Line 1:** `#!/usr/bin/env python3` is the shebang line. It tells Linux which interpreter to use if I run the script directly (`./inventory.py`) instead of `python3 inventory.py`.
- **Line 46:** `{device['hostname']:<6}` is a format specifier — `<6` means left-align in a field 6 characters wide. This makes the output columns line up neatly.
- **Line 52:** A list comprehension with a filter condition — equivalent to a `when:` condition in Ansible.
- **Line 60:** `set()` automatically removes duplicates. `update()` adds all items from a list into the set.

Run it:
```bash
python3 inventory.py
```

---

<h4>YAML Reader and JSON Writer</h4>

I'll write a script that reads a YAML file (like an Ansible vars file) and writes the data back out as JSON — the kind of format conversion I'll do constantly.

First, create the YAML file `~/projects/python-practice/network_vars.yml`:

```yaml {linenos=table}
---
site_name: "HQ-DataCenter"
dns_servers:
  - "8.8.8.8"
  - "8.8.4.4"
ntp_servers:
  - "pool.ntp.org"
  - "time.cloudflare.com"
vlans:
  - id: 10
    name: "MGMT"
    subnet: "192.168.10.0/24"
  - id: 20
    name: "USERS"
    subnet: "192.168.20.0/24"
  - id: 30
    name: "SERVERS"
    subnet: "192.168.30.0/24"
banner_motd: |
  *** Authorized Access Only ***
  All activity is monitored and logged.
```

Now create `~/projects/python-practice/yaml_to_json.py`:

```python {linenos=table}
#!/usr/bin/env python3
"""
Exercise 2: Read a YAML vars file and convert to JSON.
This mirrors what Ansible does internally when it reads group_vars files.
"""
import yaml
import json

# Read the YAML file
with open("network_vars.yml", "r") as f:
    data = yaml.safe_load(f)

# Display what was loaded
print(f"Site: {data['site_name']}")
print(f"DNS Servers: {', '.join(data['dns_servers'])}")
print(f"\nVLAN Summary:")
for vlan in data["vlans"]:
    print(f"  VLAN {vlan['id']:>4} | {vlan['name']:<10} | {vlan['subnet']}")

# Write to JSON
with open("network_vars.json", "w") as f:
    json.dump(data, f, indent=4)

print("\nConverted to network_vars.json successfully.")
```

Run it:
```bash
pip3 install pyyaml --break-system-packages
python3 yaml_to_json.py
```

---

<h4>API Test with `requests`</h4>

This exercises the `requests` library against a public test API — no network devices or Netbox instance needed.

Create `~/projects/python-practice/api_test.py`:

```python {linenos=table}
#!/usr/bin/env python3
"""
Exercise 3: Practice using the requests library against a public test API.
httpbin.org is a free HTTP testing service — perfect for learning requests.
"""
import requests
import json

BASE_URL = "https://httpbin.org"

# --- Test 1: GET request ---
print("Test 1: GET Request")
response = requests.get(f"{BASE_URL}/get", params={"device": "R1", "site": "HQ"})
print(f"  Status: {response.status_code}")
data = response.json()
print(f"  My IP as seen by server: {data['origin']}")
print(f"  Query params sent: {data['args']}")

# --- Test 2: POST request (simulating creating a device via API) ---
print("\nTest 2: POST Request")
new_device = {
    "name": "R3",
    "device_type": "CSR1000v",
    "site": "Branch",
    "mgmt_ip": "192.168.1.3"
}

response = requests.post(
    f"{BASE_URL}/post",
    json=new_device,           # Automatically sets Content-Type: application/json
    headers={"Authorization": "Token fake-token-for-testing"}
)

print(f"  Status: {response.status_code}")
echo_data = response.json()
print(f"  Data received by server: {json.dumps(echo_data['json'], indent=2)}")

# --- Test 3: Error handling ---
print("\nTest 3: Error Handling")
response = requests.get(f"{BASE_URL}/status/404")
if response.status_code == 200:
    print("  Success")
elif response.status_code == 404:
    print("  404 Not Found — handle this gracefully")
elif response.status_code == 401:
    print("  401 Unauthorized — check API token")
else:
    print(f"  Unexpected status: {response.status_code}")
```

- **Line 13:** `params={"device": "R1"}` adds query parameters to the URL automatically — `?device=R1&site=HQ`. Much cleaner than building the URL string manually.
- **Line 30:** Passing `json=new_device` to `requests.post()` automatically serializes the dictionary to JSON and sets the correct `Content-Type` header. No need to call `json.dumps()` manually.

Run it:
```bash
pip3 install requests --break-system-packages
python3 api_test.py
```

> [!Tip]
> I keep these exercise scripts in my Git repo (Part 4). When I get to Netbox integration in Part 26, I'll come back to `api_test.py` and swap `httpbin.org` for my actual Netbox URL. The structure of the code — headers, status code checking, JSON parsing — stays exactly the same.

{{% /steps %}}

---

## Reading Error Messages

Errors are inevitable. Here's how I read them without panicking.

```
Traceback (most recent call last):
  File "inventory.py", line 53, in <module>
    print(f"  {device['hostname']:<6} | {device['ip']}")
KeyError: 'ip'
```

I always read from **bottom to top**:

1. **Last line** — the actual error type and message: `KeyError: 'ip'`
2. **Second to last** — the exact line of code that caused it: `device['ip']`
3. **Third to last** — the file and line number: `inventory.py`, line 53

In this case, I tried to access a key `'ip'` that doesn't exist in my dictionary. The key is actually named `'mgmt_ip'`. Fixed.

---

Common Errors

| Error | What It Means | Typical Fix |
|---|---|---|
| `KeyError: 'hostname'` | Dictionary key doesn't exist | Use `.get()` or check the key name |
| `IndentationError` | Wrong whitespace | Fix indentation — Python requires consistency |
| `SyntaxError` | Invalid Python syntax | Check for missing colons, brackets, quotes |
| `ModuleNotFoundError` | Library not installed | `pip3 install <library>` |
| `TypeError` | Wrong data type | Check what type the function expects |
| `FileNotFoundError` | File path is wrong | Check the path with `ls` |

---

Python isn't the destination — it's the vehicle. Now that I'm comfortable with the fundamentals, Part 3 sets up the Python virtual environment that will contain all my Ansible tools, keeping my Ubuntu VM clean and my project dependencies isolated.
