---
draft: false
title: '4 - Virtualenv'
toc: true
description: "Part 4 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 4
---

{{< badge "Ansible" >}}
{{< badge content="VS Code" color="blue" >}}
{{< badge content="Linux" color="red" >}}

---

{{< lab-callout type="info" >}}
This is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. Each part will build upon the last. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.
{{< /lab-callout >}}

---


## Python Virtual Environments

In Part 2 I used `--break-system-packages` every time I installed something with pip. That flag exists because Ubuntu 22.04 protects its system Python from being polluted by random packages. The right solution isn't to force past that protection, it's to stop installing things into the system Python altogether. Virtual environments are how I do that.

---

{{< subtle-label >}}virtual Environment{{< /subtle-label >}}

When I install a Python package with `pip3 install ansible`, it goes into the system-wide Python installation at `/usr/lib/python3/`. Every Python script on the entire Ubuntu VM now shares that installation.

Here's the problem: different projects need different versions of the same package.

{{< codeblock lang="" copy="false" >}}
Project A (ansible-network):   needs ansible==9.x, netmiko==4.x
Project B (legacy-scripts):    needs ansible==2.9, netmiko==3.x
System tools (apt, Ubuntu):    need specific Python packages to function
{{< /codeblock >}}

If I install everything into the system Python, these projects will constantly conflict. Upgrading Ansible for Project A breaks Project B. A bad pip install can break Ubuntu's own system tools that depend on Python.

A virtual environment solves this by creating a completely isolated Python installation for each project:

{{< codeblock lang="" syntax="ini" lines="true" copy="false" >}}
/home/ansible/
├── venvs/
│   ├── ansible-network/ 
│   │   ├── bin/python3  
│   │   ├── bin/pip  
│   │   ├── bin/ansible 
│   │   └── lib/  
│   └── legacy-scripts/  
│       ├── bin/python3
│       └── lib/
└── projects/
    └── ansible-network/
{{< /codeblock >}}

{{< line-explain >}}
Line 3:
: Isolated Python for this guide's project

Line 4:
: Its own Python interpreter

Line 5:
: Its own pip

Line 6:
: Ansible installed HERE, not system-wide

Line 7:
: All packages installed here

Line 8:
: Completely separate environment

Line 12:
: My project code lives here (not inside venvs/)
{{< /line-explain >}}


When I activate a virtual environment, my shell uses that environment's Python and pip instead of the system ones. When I deactivate it, the system Python comes back.

---

## venv vs virtualenv

Both tools create virtual environments, but they come from different places and have slightly different capabilities.

---

{{< subtle-label >}}VENV - Python's Built-in Tool{{< /subtle-label >}}

`venv` is included with Python 3.3+ and ships with Ubuntu 22.04. No installation required. It's the official, standard way to create virtual environments and covers everything I need.

How to check if venv is available:

{{< codeblock lang="Bash" syntax="bash" >}}
python3 -m venv --help
{{< /codeblock >}}


**Limitations of `venv`:**
- Cannot create environments for Python versions other than the one it's called from
- Slightly fewer configuration options than `virtualenv`

---

{{< subtle-label >}}virtualenv - The Third-Party Tool{{< /subtle-label >}}

`virtualenv` is a more mature, feature-rich third-party package that predates Python's built-in `venv`. It's what most older documentation and enterprise tooling references when they say "virtual environment."

How to install and check version:

{{< codeblock lang="Bash" syntax="bash" >}}
pip3 install virtualenv --break-system-packages
virtualenv --version
{{< /codeblock >}}

**Additional capabilities of `virtualenv`:**
- Can create environments for different Python versions (e.g., `virtualenv -p python3.9 myenv`)
- Faster environment creation
- More configuration options
- Required by some tools like `tox` (a testing framework)

{{< subtle-label >}}Comparison{{< /subtle-label >}}

| Situation | Use |
|---|---|
| New project, single Python version | `venv` - no installation needed, officially supported |
| Need to target a specific Python version | `virtualenv` |
| Working with older team documentation | `virtualenv` - matches what they're describing |
| CI/CD pipelines and testing with `tox` | `virtualenv` |
| This guide's main project | `venv` - simple, built-in, sufficient |

---

## Installing `pip` and `virtualenv`

Before creating any environments, I make sure the necessary tools are in place.

---

{{< subtle-label >}}Pre-Check{{< /subtle-label >}}

Check Python version:

{{< codeblock lang="Bash" syntax="bash" >}}
python3 --version
{{< /codeblock >}}

Check Pip version:

{{< codeblock lang="Bash" syntax="bash" >}}
pip3 --version
{{< /codeblock >}}


Check if venv module is available:

{{< codeblock lang="Bash" syntax="bash" >}}
python3 -m venv --help 2>&1 | head -5
{{< /codeblock >}}

---

{{< subtle-label >}}Installing pip and venv{{< /subtle-label >}}

On a fresh Ubuntu 22.04 install, I run:

{{< codeblock lang="Bash" syntax="bash" >}}
sudo apt update
sudo apt install -y python3-pip python3-venv
{{< /codeblock >}}

{{< line-explain >}}
python3-pip:
: installs pip3, the package installer for Python 3

python3-venv:
: installs the venv module (sometimes missing on minimal Ubuntu installs)
{{< /line-explain >}}

---

{{< subtle-label >}}Upgrading pip{{< /subtle-label >}}

pip has its own updates. Before creating any environments, I upgrade pip:

{{< codeblock lang="Bash" syntax="bash" >}}
pip3 install --upgrade pip --break-system-packages
{{< /codeblock >}}

---

{{< subtle-label >}}Installing virtualenv{{< /subtle-label >}}

Even though I'll primarily use `venv` for this project, I install `virtualenv` to have both available:

{{< codeblock lang="Bash" syntax="bash" >}}
pip3 install virtualenv --break-system-packages
virtualenv --version
{{< /codeblock >}}

---

## Creating a Virtual Environment

Now I create the virtual environment that will house Ansible and all related tools for this lab. I keep all my virtual environments in `~/venvs/` and project code in `~/projects/`.

---

{{< subtle-label >}}Creating the Environment with venv{{< /subtle-label >}}

First, I create the venvs directory:

{{< codeblock lang="Bash" syntax="bash" >}}
mkdir -p ~/venvs
{{< /codeblock >}}

Then I create the virtual environment:

{{< codeblock lang="Bash" syntax="bash" >}}
python3 -m venv ~/venvs/ansible-network
{{< /codeblock >}}

{{< line-explain >}}
python3 -m venv:
: runs the venv module using Python 3

~/venvs/ansible-network:
: the path where the environment will be created
{{< /line-explain >}}

---

{{< subtle-label >}}What Gets Created{{< /subtle-label >}}

{{< codeblock lang="Bash" syntax="bash" >}}
tree ~/venvs/ansible-network/ -L 3
{{< /codeblock >}}

{{< codeblock lang="Expected Output" syntax="ini" copy="false" lines="true" >}}
/home/ansible/venvs/ansible-network/
├── bin/
│   ├── activate 
│   ├── activate.fish
│   ├── pip
│   ├── pip3
│   ├── python
│   └── python3
├── include/
├── lib/
│   └── python3.10/
│       └── site-packages/
└── pyvenv.cfg
{{< /codeblock >}}

{{< line-explain >}}
Line 3:
: The activation script I'll source

Line 4:
: Activation script for Fish shell

Line 5:
: pip linked to this environment

Line 6:
: Same as pip

Line 7:
: Python interpreter for this environment

Line 8:
: Same as python

Line 9:
: C headers for compiling extensions

Line 12:
: Installed packages go here

Line 13:
: Environment configuration file
{{< /line-explain >}}


**`pyvenv.cfg`** is worth looking at:

{{< codeblock lang="Bash" syntax="bash" >}}
cat ~/venvs/ansible-network/pyvenv.cfg
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
home = /usr/bin
include-system-site-packages = false
version = 3.10.12
{{< /codeblock >}}

The `include-system-site-packages = false` line means this environment is completely isolated from the system Python packages. Nothing installed system-wide bleeds in.

---

{{< subtle-label >}}Creating the Same Environment with virtualenv{{< /subtle-label >}}

Use virtualenv instead of venv:

{{< codeblock lang="Bash" syntax="bash" >}}
virtualenv ~/venvs/ansible-network-v2
{{< /codeblock >}}


Target a sepcific Python version with virtualenv:

{{< codeblock lang="Bash" syntax="bash" >}}
virtualenv -p python3.10 ~/venvs/ansible-network-v3
{{< /codeblock >}}


`-p python3.10` - tells virtualenv which Python binary to use. This is the main advantage over `venv` since I can target any installed Python version.

{{< lab-callout type="info" >}}
Both `venv` and `virtualenv` produce environments that work identically once created. The activation scripts, pip behavior, and package isolation are the same. The difference is only in how the environment is created and what options are available at creation time.
{{< /lab-callout >}}

---

## Naming Conventions

Project-Specific vs Generic naming conventions.

---

{{< subtle-label >}}Approach 1: Name After the Project{{< /subtle-label >}}

{{< codeblock lang="Expected Output" copy="false" >}}
python3 -m venv ~/venvs/ansible-network
python3 -m venv ~/venvs/netbox-scripts
python3 -m venv ~/venvs/legacy-automation
{{< /codeblock >}}

**When to use this:** When I have multiple projects with different dependency requirements. Each project gets its own environment. The name makes it immediately obvious which project the environment belongs to.

**Pros:**
- Clear association between environment and project
- Easy to manage when working on multiple projects
- Aligns with the "one virtualenv per project" best practice

**Cons:**
- If I rename the project, the environment name becomes confusing
- Managing many environments requires discipline

---

{{< subtle-label >}}Approach 2: Generic Name{{< /subtle-label >}}

{{< codeblock lang="Expected Output" copy="false" >}}
python3 -m venv ~/venvs/ansible-env
python3 -m venv ~/venvs/python-env
{{< /codeblock >}}

**When to use this:** When I have one general-purpose environment for all Ansible work, or on a dedicated Ansible control node that only ever runs one type of workload.

**Pros:**
- Simple, one environment to rule them all
- Common in CI/CD pipelines where the environment name doesn't matter

**Cons:**
- Over time, the environment accumulates packages from multiple projects
- Hard to tell which packages are actually needed vs. installed by accident
- `requirements.txt` becomes bloated

---

{{< subtle-label >}}Convention for this Project{{< /subtle-label >}}

For this project I use the project-specific naming approach:

{{< codeblock lang="Expected Output" copy="false" >}}
python3 -m venv ~/venvs/ansible-network
{{< /codeblock >}}

This environment is dedicated to this project. When I start a new, separate project later, it gets its own environment.

---

## Activating and Deactivating

Creating the environment doesn't automatically use it. I need to activate it.

---

{{< subtle-label >}}Activating{{< /subtle-label >}}

{{< codeblock lang="Bash" syntax="bash" >}}
source ~/venvs/ansible-network/bin/activate
{{< /codeblock >}}

- `source` - runs the activation script in the current shell session (not a subprocess). The environment variables set by `activate` wouldn't affect my current shell.
- After activation, my shell prompt changes to show the environment name:

{{< codeblock lang="Expected Output" copy="false" >}}
# Before activation
ansible@ubuntu:~$

# After activation
(ansible-network) ansible@ubuntu:~$
{{< /codeblock >}}

That `(ansible-network)` prefix is my visual confirmation that I'm inside the right environment.

---

{{< subtle-label >}}Confirmation{{< /subtle-label >}}

Check where Python is coming from:

{{< codeblock lang="Bash" syntax="bash" >}}
which python3
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
# /home/ansible/venvs/ansible-network/bin/python3
{{< /codeblock >}}

Check where pip coming from:

{{< codeblock lang="Bash" syntax="bash" >}}
which pip
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
# /home/ansible/venvs/ansible-network/bin/pip
{{< /codeblock >}}

Check whhat packages are currently installed:

{{< codeblock lang="Bash" syntax="bash" >}}
pip list
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
# Package    Version
# ---------- -------
# pip        24.x.x
# setuptools 68.x.x
{{< /codeblock >}}

---

{{< subtle-label >}}Deactivating{{< /subtle-label >}}

{{< codeblock lang="Bash" syntax="bash" >}}
deactivate
{{< /codeblock >}}


My prompt returns to normal and Python/pip point back to the system installation.

{{< codeblock lang="Bash" syntax="bash" >}}
which python3
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
# /usr/bin/python3
{{< /codeblock >}}

---

{{< subtle-label >}}Automating Activation{{< /subtle-label >}}

I can add the activation command to my `~/.bashrc` so it activates automatically on every new shell:

{{< codeblock lang="Bash" syntax="bash" >}}
echo "source ~/venvs/ansible-network/bin/activate" >> ~/.bashrc
source ~/.bashrc
{{< /codeblock >}}

{{< lab-callout type="info" >}}
Auto-activating in `.bashrc` is convenient for a dedicated Ansible control node where I always want this environment active. But if I use the Ubuntu VM for multiple Python projects, auto-activating one environment means I always have to manually deactivate it before working on another project. For a single-purpose Ansible VM, auto-activation in `.bashrc` is fine. For a general-purpose dev machine, activate manually per session.
{{< /lab-callout >}}

---

## Installing the Full Ansible Networking Stack

With the environment activated, I now install everything needed for this project. I no longer need `--break-system-packages` since I'm inside the virtualenv and pip only touches this isolated environment.

---

{{< subtle-label >}}Upgrading PIP{{< /subtle-label >}}

I always upgrade pip inside a fresh virtualenv before installing anything.

{{< codeblock lang="Bash" syntax="bash" >}}
pip install --upgrade pip
{{< /codeblock >}}

---

{{< subtle-label >}}Installing the Core Stack{{< /subtle-label >}}

I'll install everything in a logical order, explaining what each package is and why I need it.

Core Ansible:

{{< codeblock lang="Bash" syntax="bash" >}}
pip install ansible
{{< /codeblock >}}


Ansible linting and validation tools.

{{< codeblock lang="Bash" syntax="bash" >}}
pip install ansible-lint
pip install yamllint
{{< /codeblock >}}


SSH connectivity libraries:

{{< codeblock lang="Bash" syntax="bash" >}}
pip install paramiko
pip install netmiko
{{< /codeblock >}}

Vendor-neutral network abstraction:

{{< codeblock lang="Bash" syntax="bash" >}}
pip install napalm
{{< /codeblock >}}

HTTP library for REST APIs (Netbox, AWX, PAN-OS):

{{< codeblock lang="Bash" syntax="bash" >}}
pip install requests
{{< /codeblock >}}

YAML parser (used in helper scripts):

{{< codeblock lang="Bash" syntax="bash" >}}
pip install pyyaml
{{< /codeblock >}}

Netbox Python client (for Netbox integration):

{{< codeblock lang="Bash" syntax="bash" >}}
pip install pynetbox
{{< /codeblock >}}

Ansible Navigator (modern way to run and inspect playbooks):

{{< codeblock lang="Bash" syntax="bash" >}}
pip install ansible-navigator
{{< /codeblock >}}

Or I can install everything in one command:

{{< codeblock lang="Bash" syntax="bash" >}}
pip install ansible ansible-lint yamllint paramiko netmiko napalm requests pyyaml pynetbox ansible-navigator
{{< /codeblock >}}

---

{{< subtle-label >}}What each package does{{< /subtle-label >}}

| Package | Purpose |
|---|---|
| `ansible` | The core automation engine — includes `ansible-playbook`, `ansible`, `ansible-galaxy` |
| `ansible-lint` | Static analysis tool that checks playbooks for best practice violations |
| `yamllint` | YAML syntax and style checker to catche formatting issues before Ansible sees them |
| `paramiko` | Python SSH library. Ansible's default SSH backend for network devices |
| `netmiko` | Multi-vendor SSH library (underpins Ansible's `network_cli` connection plugin) |
| `napalm` | Vendor-neutral network abstraction layer (used by `napalm_*` Ansible modules) |
| `requests` | HTTP library (used for REST API calls to Netbox, AWX, PAN-OS) |
| `pyyaml` | YAML parser (used in helper scripts and by Ansible itself) |
| `pynetbox` | Netbox Python client (simplifies Netbox API interactions) |
| `ansible-navigator` | Modern TUI for running and inspecting Ansible playbooks |

---

{{< subtle-label >}}Verifying the Installation{{< /subtle-label >}}

I can verify Ansible is installed and check the version with:

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

The key line to confirm: `executable location` should point into my virtualenv (`/home/ansible/venvs/ansible-network/bin/ansible`), not `/usr/bin/ansible`.

Verify the other tools:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-lint --version
yamllint --version
python3 -c "import netmiko; print(netmiko.__version__)"
python3 -c "import napalm; print(napalm.__version__)"
python3 -c "import paramiko; print(paramiko.__version__)"
{{< /codeblock >}}

---

{{< subtle-label >}}Installing Ansible Collections{{< /subtle-label >}}

Collections are Ansible's plugin/module packages. They're installed separately from pip packages using `ansible-galaxy`:


Install network vendor collections:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install cisco.ios
ansible-galaxy collection install cisco.nxos
ansible-galaxy collection install junipernetworks.junos
ansible-galaxy collection install paloaltonetworks.panos
{{< /codeblock >}}


Install utility collections:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install ansible.netcommon
ansible-galaxy collection install ansible.utils
ansible-galaxy collection install netbox.netbox
{{< /codeblock >}}

Verify installed collections

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection list
{{< /codeblock >}}

---

## Freezing Dependencies with requirements.txt

Right now my virtual environment has a specific set of packages at specific versions. If I delete the environment, move to a new VM, or hand this project to a colleague, I need a way to recreate it exactly. That's what `requirements.txt` is for.

---

{{< subtle-label >}}Generating requirements.txt{{< /subtle-label >}}

First, I make sure the virtualenv is activated, and then freeze all installed packages and their  versions.

{{< codeblock lang="Bash" syntax="bash" >}}
pip freeze > requirements.txt
{{< /codeblock >}}

View the results:

{{< codeblock file="requirements.txt" syntax="ini" >}}
ansible==9.8.0
ansible-core==2.17.8
ansible-lint==24.9.2
ansible-navigator==24.8.0
cffi==1.17.1
cryptography==43.0.3
Jinja2==3.1.4
MarkupSafe==2.1.5
napalm==5.0.0
netmiko==4.4.0
nornir==3.4.0
packaging==24.1
paramiko==3.5.0
pynetbox==7.4.0
PyNaCl==1.5.0
pyyaml==6.0.2
requests==2.32.3
resolvelib==1.0.1
setuptools==68.2.0
urllib3==2.2.3
yamllint==1.35.1
...
{{< /codeblock >}}

**`pip freeze`** captures every single installed package including dependencies of my dependencies. This guarantees a perfectly reproducible environment.

{{< lab-callout type="info" >}}
The output of `pip freeze` includes packages I didn't explicitly install like `cffi`, `cryptography`, `MarkupSafe`. These are **transitive dependencies** packages that `paramiko`, `ansible`, or `requests` depend on. Freezing them ensures that even sub-dependency versions are locked, preventing the scenario where a sub-dependency gets updated and silently breaks something.
{{< /lab-callout >}}

---

{{< subtle-label >}}Two Styles of requirements.txt{{< /subtle-label >}}

There are two common approaches, and both have their place:

**Style 1: Pinned versions for Maximum reproducibility**

{{< codeblock lang="" copy="false" >}}
ansible==9.8.0
ansible-lint==24.9.2
netmiko==4.4.0
paramiko==3.5.0
{{< /codeblock >}}

Use this for production environments where I need guaranteed stability. Every engineer gets the exact same versions.

**Style 2: Minimum versions for Maximum flexibility**

{{< codeblock lang="" copy="false" >}}
ansible>=9.0.0
ansible-lint>=24.0.0
netmiko>=4.0.0
paramiko>=3.0.0
{{< /codeblock >}}

Use this for open-source or shared projects where I want to allow newer compatible versions. Less strict, but more likely to get the latest bug fixes.

For this project I use pinned versions from `pip freeze` for the actual project environment, and keep a separate loose `requirements-dev.txt` for quick new environment setups where exact versions matter less.

{{< lab-callout type="tip" >}}
I commit `requirements.txt` to Git every time I add or upgrade a package. The git history then shows exactly when each package was added or updated and by whom. This is helps for debugging: "Ansible started failing on Thursday let me check what changed in `requirements.txt` on Thursday."
{{< /lab-callout >}}

---

{{< subtle-label >}}Recreating an Environment{{< /subtle-label >}}

This is the payoff for maintaining `requirements.txt`. Whether I'm setting up a new VM, onboarding a colleague, or recovering from a broken environment, I can recreate everything in minutes.

Step 1: Clone the project repository:

{{< codeblock lang="Bash" syntax="bash" >}}
git clone git@github.com:ernestodiaztech/ansible-network.git
cd ansible-network
{{< /codeblock >}}

Step 2: Create a fresh virtual environment:

{{< codeblock lang="Bash" syntax="bash" >}}
python3 -m venv ~/venvs/ansible-network
{{< /codeblock >}}

Step 3: Activate it:

{{< codeblock lang="Bash" syntax="bash" >}}
source ~/venvs/ansible-network/bin/activate
{{< /codeblock >}}

Step 4: Upgrade pip:

{{< codeblock lang="Bash" syntax="bash" >}}
pip install --upgrade pip
{{< /codeblock >}}

Step 5: Install all packages from requirements.txt:

{{< codeblock lang="Bash" syntax="bash" >}}
pip install -r requirements.txt
{{< /codeblock >}}

Step 6: Reinstall Ansible collections:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install -r collections/requirements.yml
{{< /codeblock >}}

Step 7: Verify

{{< codeblock lang="Bash" syntax="bash" >}}
ansible --version
{{< /codeblock >}}

The `-r requirements.txt` flag tells pip to read the file and install everything listed in it. The entire environment is rebuilt in one command.

---

{{< subtle-label >}}Managing Collections{{< /subtle-label >}}

Just as `requirements.txt` tracks Python packages, `collections/requirements.yml` tracks Ansible collections:

{{< codeblock lang="Bash" syntax="bash" >}}
mkdir -p ~/projects/ansible-network/collections
{{< /codeblock >}}

Create `~/projects/ansible-network/collections/requirements.yml`:

{{< codeblock lang="YAML" syntax="yaml" >}}
---
collections:
  - name: cisco.ios
    version: ">=8.0.0"
  - name: cisco.nxos
    version: ">=5.0.0"
  - name: junipernetworks.junos
    version: ">=8.0.0"
  - name: paloaltonetworks.panos
    version: ">=2.0.0"
  - name: ansible.netcommon
    version: ">=6.0.0"
  - name: ansible.utils
    version: ">=4.0.0"
  - name: netbox.netbox
    version: ">=3.0.0"
{{< /codeblock >}}

Install from this file:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-galaxy collection install -r collections/requirements.yml
{{< /codeblock >}}

---

## Setting Up VS Code

VS Code needs to know which Python interpreter to use, specifically the one inside my virtual environment. Without this, VS Code's IntelliSense, linting, and the Ansible extension won't see the packages I've installed.

---

{{< subtle-label >}}Selecting the Python Interpreter{{< /subtle-label >}}

With VS Code connected to my Ubuntu VM via Remote-SSH:

1. Press `Ctrl+Shift+P` to open the Command Palette
2. Type `Python: Select Interpreter` and press Enter
3. VS Code scans for available Python environments and shows a list
4. I select the one pointing to my virtualenv:

{{< codeblock lang="" copy="false" >}}
   Python 3.10.12 ('ansible-network': venv)
   ~/venvs/ansible-network/bin/python3
{{< /codeblock >}}

If VS Code doesn't detect it automatically, I click **Enter interpreter path...** and type the full path:

{{< codeblock lang="" copy="false" >}}
/home/ansible/venvs/ansible-network/bin/python3
{{< /codeblock >}}

---

{{< subtle-label >}}Configuring VS Code Settings{{< /subtle-label >}}

I add a `.vscode/settings.json` file to my project directory so the interpreter setting is saved with the project and applies automatically when anyone opens it:

{{< codeblock lang="Bash" syntax="bash" >}}
mkdir -p ~/projects/ansible-network/.vscode
{{< /codeblock >}}

Create `~/projects/ansible-network/.vscode/settings.json`:

{{< codeblock file="settings.json" syntax="json" >}}
{
    "python.defaultInterpreterPath": "/home/ansible/venvs/ansible-network/bin/python3",
    "ansible.python.interpreterPath": "/home/ansible/venvs/ansible-network/bin/python3",
    "ansible.validation.enabled": true,
    "ansible.validation.lint.enabled": true,
    "yaml.schemas": {
        "https://raw.githubusercontent.com/ansible/ansible-lint/main/src/ansiblelint/schemas/ansible.json#/$defs/playbook": "playbooks/*.yml",
        "https://raw.githubusercontent.com/ansible/ansible-lint/main/src/ansiblelint/schemas/ansible.json#/$defs/tasks": "roles/*/tasks/*.yml"
    },
    "editor.tabSize": 2,
    "editor.insertSpaces": true,
    "[yaml]": {
        "editor.tabSize": 2,
        "editor.insertSpaces": true,
        "editor.formatOnSave": true
    }
}
{{< /codeblock >}}

{{< line-explain >}}
python.defaultInterpreterPath:
: tells the Python extension which interpreter to use

ansible.python.interpreterPath:
: tells the Ansible extension which interpreter to use (must match)

ansible.validation.enabled:
: enables real-time playbook validation in the editor

ansible.validation.lint.enabled:
: enables ansible-lint integration in VS Code

yaml.schemas:
: maps my playbook and task files to the Ansible YAML schema for validation

editor.tabSize: 2:
: YAML best practice is 2-space indentation

editor.insertSpaces: true:
: uses spaces, not tabs (tabs in YAML are illegal)

"[yaml]": { "editor.formatOnSave": true }:
: auto-formats YAML files on save
{{< /line-explain >}}

{{< lab-callout type="warning" >}}
YAML does **not** allow tab characters for indentation, only spaces. If VS Code is configured to insert tabs (`editor.insertSpaces: false`) in YAML files, every playbook I save will be silently broken. Always verify `editor.insertSpaces` is `true` and `editor.tabSize` is `2` for YAML files. The `indent-rainbow` extension (from Part 1) will highlight incorrect indentation visually.
{{< /lab-callout >}}

---

{{< subtle-label >}}Verifying VS Code Sees the Right Environment{{< /subtle-label >}}

After selecting the interpreter, I open a new VS Code terminal (`Ctrl+`` `). The virtual environment should activate automatically. I verify:

{{< codeblock lang="Bash" syntax="bash" >}}
which ansible
which python3
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
# /home/ansible/venvs/ansible-network/bin/ansible
# /home/ansible/venvs/ansible-network/bin/python3
{{< /codeblock >}}

The `.vscode/settings.json` file should be committed to Git so the entire team gets the same VS Code configuration automatically. However, if some team members have the virtualenv at a different path (e.g., on macOS it would be different), the hardcoded path becomes a problem. A cleaner approach for cross-platform teams is to use a relative path or the `${workspaceFolder}` VS Code variable and keep the virtualenv inside the project directory itself. For this guide's single-user setup, the absolute path is fine.

---

## Best Practices

{{< subtle-label >}}One Virtual Environment Per Project{{< /subtle-label >}}

**Correct - separate environments for separate projects**

{{< codeblock lang="" copy="false" >}}
~/venvs/ansible-network/       ← This guide's project
~/venvs/legacy-nornir-scripts/ ← Older project using different library versions
~/venvs/netbox-api-tools/      ← Standalone Netbox scripts
{{< /codeblock >}}

**Wrong - one environment for everything**

{{< codeblock lang="" copy="false" >}}
~/venvs/python-env/   ← All projects crammed in here, dependency chaos
{{< /codeblock >}}

---

{{< subtle-label >}}Never Commit the venv Directory to Git{{< /subtle-label >}}

The virtual environment directory is large (often 50–200MB), contains compiled binaries specific to my OS, and is completely regeneratable from `requirements.txt`. It must never go into Git.

I add it to `.gitignore`:

{{< codeblock file=".gitignore" syntax="ini" >}}
# .gitignore
venv/
venvs/
.venv/
env/
{{< /codeblock >}}

---

{{< subtle-label >}}Pin Versions{{< /subtle-label >}}

After setting up the environment and confirming everything works:

{{< codeblock lang="Bash" syntax="bash" >}}
pip freeze > requirements.txt
git add requirements.txt
git commit -m "Pin Python dependencies for ansible-network environment"
{{< /codeblock >}}

---

{{< subtle-label >}}Document the Setup Process in README.md{{< /subtle-label >}}

Every project should have a `README.md` that tells a new engineer exactly how to recreate the environment:

{{< codeblock lang="markdown" syntax="markdown" >}}
## Setup

1. Clone the repository:
   git clone git@github.com:ernestodiaztech/ansible-network.git

2. Create and activate a virtual environment:
   python3 -m venv ~/venvs/ansible-network
   source ~/venvs/ansible-network/bin/activate

3. Install Python dependencies:
   pip install --upgrade pip
   pip install -r requirements.txt

4. Install Ansible collections:
   ansible-galaxy collection install -r collections/requirements.yml

5. Verify:
   ansible --version
{{< /codeblock >}}

---

## Managing the Environment Over Time

{{< subtle-label >}}Adding a New Package{{< /subtle-label >}}

Activate the environment:

{{< codeblock lang="Bash" syntax="bash" >}}
source ~/venvs/ansible-network/bin/activate
{{< /codeblock >}}

Install the new package:

{{< codeblock lang="Bash" syntax="bash" >}}
pip install some-new-library
{{< /codeblock >}}

Update requirements.txt immediately:

{{< codeblock lang="Bash" syntax="bash" >}}
pip freeze > requirements.txt
{{< /codeblock >}}

Commit the change

{{< codeblock lang="Bash" syntax="bash" >}}
git add requirements.txt
git commit -m "Add some-new-library for feature X"
{{< /codeblock >}}

---

{{< subtle-label >}}Upgrading Packages{{< /subtle-label >}}

Upgrade a specific package:

{{< codeblock lang="Bash" syntax="bash" >}}
pip install --upgrade ansible
{{< /codeblock >}}

Check what changed:

{{< codeblock lang="Bash" syntax="bash" >}}
pip freeze > requirements.txt
git diff requirements.txt
{{< /codeblock >}}

Commit if everything still works:

{{< codeblock lang="Bash" syntax="bash" >}}
git add requirements.txt
git commit -m "Upgrade ansible from 9.7.0 to 9.8.0"
{{< /codeblock >}}

---

{{< subtle-label >}}Upgrading All Packages at Once{{< /subtle-label >}}

List outdated packages:

{{< codeblock lang="Bash" syntax="bash" >}}
pip list --outdated
{{< /codeblock >}}

Upgrade all packages (use with caution):

{{< codeblock lang="Bash" syntax="bash" >}}
pip install --upgrade $(pip freeze | cut -d= -f1)
{{< /codeblock >}}

Test everything still works before freezing:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible --version
ansible-lint --version
{{< /codeblock >}}

Then freeze and commit:

{{< codeblock lang="Bash" syntax="bash" >}}
pip freeze > requirements.txt
{{< /codeblock >}}

---

{{< subtle-label >}}Deleting and Recreating a Broken Environment{{< /subtle-label >}}

If the virtualenv gets corrupted or I want a clean slate:

Deactivate first if active:

{{< codeblock lang="Bash" syntax="bash" >}}
deactivate
{{< /codeblock >}}

Delete the entire environment directory:

{{< codeblock lang="Bash" syntax="bash" >}}
rm -rf ~/venvs/ansible-network
{{< /codeblock >}}

Recreate from scratch:

{{< codeblock lang="Bash" syntax="bash" >}}
python3 -m venv ~/venvs/ansible-network
source ~/venvs/ansible-network/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml
{{< /codeblock >}}

Verify:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible --version
{{< /codeblock >}}

This takes about 2-3 minutes and produces a perfectly clean environment. The virtualenv is truly disposable because everything needed to recreate it is in `requirements.txt`.

---

The environment is now fully set up and reproducible. Everything runs inside an isolated, documented, version-controlled Python environment.