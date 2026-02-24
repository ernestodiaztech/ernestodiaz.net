---
draft: false
title: '3 - Virtualenv'
weight: 3
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 3: Python Virtual Environments (`virtualenv`)

> *In Part 2 I used `--break-system-packages` every time I installed something with pip. That flag exists because Ubuntu 22.04 protects its system Python from being polluted by random packages. The right solution isn't to force past that protection — it's to stop installing things into the system Python altogether. Virtual environments are how I do that. This part sets up the isolated Python environment that everything else in this guide runs inside.*

---

## 3.1 — What a Virtual Environment Is and Why It Matters

When I install a Python package with `pip3 install ansible`, it goes into the system-wide Python installation at `/usr/lib/python3/`. Every Python script on the entire Ubuntu VM now shares that installation. This sounds convenient until it isn't.

Here's the problem: different projects need different versions of the same package.

```
Project A (ansible-network):   needs ansible==9.x, netmiko==4.x
Project B (legacy-scripts):    needs ansible==2.9, netmiko==3.x
System tools (apt, Ubuntu):    need specific Python packages to function
```

If I install everything into the system Python, these projects will constantly conflict. Upgrading Ansible for Project A breaks Project B. Worse, a bad pip install can break Ubuntu's own system tools that depend on Python.

A virtual environment solves this by creating a completely isolated Python installation for each project:

```
/home/ansible/
├── venvs/
│   ├── ansible-network/      ← Isolated Python for this guide's project
│   │   ├── bin/python3       ← Its own Python interpreter
│   │   ├── bin/pip           ← Its own pip
│   │   ├── bin/ansible       ← Ansible installed HERE, not system-wide
│   │   └── lib/              ← All packages installed here
│   └── legacy-scripts/       ← Completely separate environment
│       ├── bin/python3
│       └── lib/
└── projects/
    └── ansible-network/      ← My project code lives here (NOT inside venvs/)
```

When I activate a virtual environment, my shell uses that environment's Python and pip instead of the system ones. When I deactivate it, the system Python comes back. Nothing ever collides.

### ### ℹ️ Info

> The virtual environment directory (`venvs/ansible-network/`) and the project directory (`projects/ansible-network/`) are kept **separate** intentionally. The venv contains the Python runtime and installed packages — it's regeneratable from `requirements.txt` and should never go into Git. The project directory contains my playbooks, inventory, and roles — that's what goes into Git. Mixing them together is a common beginner mistake that creates messy repositories.

### ### 🏢 Real-World Scenario

> In enterprise environments, the Ansible control node is often shared between multiple teams or projects. Team A runs Ansible 9.x for modern IOS-XE automation. Team B maintains legacy scripts that only work on Ansible 2.9 for older devices that haven't been upgraded. Without virtual environments, these teams would be in constant conflict over which version of Ansible is installed. With virtual environments, each team activates their own isolated environment and the two never interfere. I've seen production outages caused by someone running `pip install --upgrade ansible` on a shared control node without understanding this — don't be that person.

---

## 3.2 — `venv` vs `virtualenv` — Understanding the Difference

Both tools create virtual environments, but they come from different places and have slightly different capabilities.

### `venv` — Python's Built-in Tool

`venv` is included with Python 3.3+ and ships with Ubuntu 22.04. No installation required. It's the official, standard way to create virtual environments and covers everything I need for this guide.

```bash
# Check if venv is available
python3 -m venv --help
```

**Limitations of `venv`:**
- Cannot create environments for Python versions other than the one it's called from
- Slightly fewer configuration options than `virtualenv`
- That's it — for most purposes these limitations don't matter

### `virtualenv` — The Third-Party Tool

`virtualenv` is a more mature, feature-rich third-party package that predates Python's built-in `venv`. It's what most older documentation and enterprise tooling references when they say "virtual environment."

```bash
# Install virtualenv
pip3 install virtualenv --break-system-packages

# Check the version
virtualenv --version
```

**Additional capabilities of `virtualenv`:**
- Can create environments for different Python versions (e.g., `virtualenv -p python3.9 myenv`)
- Faster environment creation
- More configuration options
- Required by some tools like `tox` (a testing framework)

### Which One Should I Use?

| Situation | Use |
|---|---|
| New project, single Python version | `venv` — no installation needed, officially supported |
| Need to target a specific Python version | `virtualenv` |
| Working with older team documentation | `virtualenv` — matches what they're describing |
| CI/CD pipelines and testing with `tox` | `virtualenv` |
| This guide's main project | `venv` — simple, built-in, sufficient |

### ### 💡 Tip

> There is also a tool called `pipenv` and another called `poetry` that combine virtual environment management with dependency resolution. I won't use them here, but they're worth knowing about. For network automation work, plain `venv` with a `requirements.txt` file is the industry standard and what I'll see on most Ansible control nodes.

---

## 3.3 — Installing `pip` and `virtualenv` on Ubuntu 22.04

Before creating any environments, I make sure the necessary tools are in place.

### Checking What's Already Installed

```bash
# Check Python version
python3 --version

# Check pip
pip3 --version

# Check if venv module is available
python3 -m venv --help 2>&1 | head -5
```

### Installing pip and venv if Missing

On a fresh Ubuntu 22.04 install, I run:

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv
```

- `python3-pip` — installs pip3, the package installer for Python 3
- `python3-venv` — installs the venv module (sometimes missing on minimal Ubuntu installs)

### Upgrading pip Itself

pip has its own updates. Before creating any environments, I upgrade pip:

```bash
pip3 install --upgrade pip --break-system-packages
```

### ### ⚠️ Warning

> On Ubuntu 22.04, pip will sometimes show a yellow warning: `WARNING: Running pip as the 'root' user can result in broken permissions...` This appears when I run pip with `sudo`. The fix is to never run `pip` with `sudo` — always run it as my regular user, either in a virtual environment (the correct approach from Part 3 onward) or with `--user` flag for one-off system-level installs.

### Installing `virtualenv` (Optional but Useful)

Even though I'll primarily use `venv` for this project, I install `virtualenv` to have both available:

```bash
pip3 install virtualenv --break-system-packages
virtualenv --version
```

---

## 3.4 — Creating a Virtual Environment

Now I create the virtual environment that will house Ansible and all related tools for this guide. I keep all my virtual environments in `~/venvs/` and project code in `~/projects/` — as established in Part 1.

### Creating the Environment with `venv`

```bash
# First, create the venvs directory if it doesn't exist
mkdir -p ~/venvs

# Create the virtual environment
python3 -m venv ~/venvs/ansible-network
```

- `python3 -m venv` — runs the venv module using Python 3
- `~/venvs/ansible-network` — the path where the environment will be created

That's it. Python creates a complete, isolated Python installation at `~/venvs/ansible-network/`.

### What Gets Created

```bash
tree ~/venvs/ansible-network/ -L 3
```

```
/home/ansible/venvs/ansible-network/
├── bin/
│   ├── activate            ← The activation script I'll source
│   ├── activate.fish       ← Activation script for Fish shell
│   ├── pip                 ← pip linked to this environment
│   ├── pip3                ← Same as pip
│   ├── python              ← Python interpreter for this environment
│   └── python3             ← Same as python
├── include/                ← C headers for compiling extensions
├── lib/
│   └── python3.10/
│       └── site-packages/  ← Installed packages go here
└── pyvenv.cfg              ← Environment configuration file
```

**`pyvenv.cfg`** is worth looking at:

```bash
cat ~/venvs/ansible-network/pyvenv.cfg
```

```
home = /usr/bin
include-system-site-packages = false
version = 3.10.12
```

The `include-system-site-packages = false` line is critical — it means this environment is completely isolated from the system Python packages. Nothing installed system-wide bleeds in.

### Creating the Same Environment with `virtualenv` (Alternative)

```bash
# Using virtualenv instead of venv — nearly identical result
virtualenv ~/venvs/ansible-network-v2

# Target a specific Python version with virtualenv
virtualenv -p python3.10 ~/venvs/ansible-network-v3
```

- `-p python3.10` — tells virtualenv which Python binary to use. This is the main advantage over `venv` — I can target any installed Python version.

### ### ℹ️ Info

> Both `venv` and `virtualenv` produce environments that work identically once created. The activation scripts, pip behavior, and package isolation are the same. The difference is only in how the environment is created and what options are available at creation time.

---

## 3.5 — Naming Conventions: Project-Specific vs Generic

As I mentioned, I'll show both approaches and when to use each.

### Approach 1: Name After the Project

```bash
python3 -m venv ~/venvs/ansible-network
python3 -m venv ~/venvs/netbox-scripts
python3 -m venv ~/venvs/legacy-automation
```

**When to use this:** When I have multiple projects with different dependency requirements. Each project gets its own environment. The name makes it immediately obvious which project the environment belongs to.

**Pros:**
- Clear association between environment and project
- Easy to manage when working on multiple projects
- Aligns with the "one virtualenv per project" best practice

**Cons:**
- If I rename the project, the environment name becomes confusing
- Managing many environments requires discipline

### Approach 2: Generic Name

```bash
python3 -m venv ~/venvs/ansible-env
python3 -m venv ~/venvs/python-env
```

**When to use this:** When I have one general-purpose environment for all Ansible work, or on a dedicated Ansible control node that only ever runs one type of workload.

**Pros:**
- Simple — one environment to rule them all
- Common in CI/CD pipelines where the environment name doesn't matter

**Cons:**
- Over time, the environment accumulates packages from multiple projects
- Hard to tell which packages are actually needed vs. installed by accident
- `requirements.txt` becomes bloated

### My Convention for This Guide

For this guide I use the project-specific naming approach:

```bash
python3 -m venv ~/venvs/ansible-network
```

This environment is dedicated to this project. When I start a new, separate project later, it gets its own environment.

### ### 💡 Tip

> In enterprise environments, the naming convention for virtualenvs is often defined by team standards. Common patterns I've seen: `{project}-{env}` (e.g., `network-automation-prod`), `{team}-{tool}` (e.g., `netops-ansible`), or simply the project repository name. Whatever convention I choose, I document it in the project's `README.md` so other engineers know what to create.

---

## 3.6 — Activating and Deactivating a Virtual Environment

Creating the environment doesn't automatically use it. I need to activate it.

### Activating

```bash
source ~/venvs/ansible-network/bin/activate
```

- `source` — runs the activation script in the current shell session (not a subprocess). This is important — without `source`, the environment variables set by `activate` wouldn't affect my current shell.
- After activation, my shell prompt changes to show the environment name:

```bash
# Before activation
ansible@ubuntu:~$

# After activation
(ansible-network) ansible@ubuntu:~$
```

That `(ansible-network)` prefix is my visual confirmation that I'm inside the right environment.

### Confirming the Activation Worked

```bash
# Where is Python coming from now?
which python3
# /home/ansible/venvs/ansible-network/bin/python3  ← virtualenv, not system

# Where is pip coming from?
which pip
# /home/ansible/venvs/ansible-network/bin/pip  ← virtualenv pip

# What version of pip?
pip --version
# pip 24.x.x from /home/ansible/venvs/ansible-network/lib/python3.10/site-packages/pip

# What packages are currently installed? (should be almost empty)
pip list
# Package    Version
# ---------- -------
# pip        24.x.x
# setuptools 68.x.x
```

### Deactivating

```bash
deactivate
```

My prompt returns to normal and Python/pip point back to the system installation.

```bash
# After deactivating
which python3
# /usr/bin/python3  ← back to system Python
```

### ### ⚠️ Warning

> A very common mistake is opening a new terminal tab or reconnecting via SSH and forgetting to reactivate the virtual environment before running Ansible commands. If I type `ansible-playbook` and get "command not found" — it almost always means I forgot to activate the virtualenv. I make activation part of my startup routine: open terminal → `source ~/venvs/ansible-network/bin/activate` → then work.

### Automating Activation (Optional)

I can add the activation command to my `~/.bashrc` so it activates automatically on every new shell:

```bash
echo "source ~/venvs/ansible-network/bin/activate" >> ~/.bashrc
source ~/.bashrc
```

### ### 🪲 Gotcha — Auto-activation in `~/.bashrc`

> Auto-activating in `.bashrc` is convenient for a dedicated Ansible control node where I always want this environment active. But if I use the Ubuntu VM for multiple Python projects, auto-activating one environment means I always have to manually deactivate it before working on another project. For a single-purpose Ansible VM, auto-activation in `.bashrc` is fine. For a general-purpose dev machine, activate manually per session.

---

## 3.7 — Installing the Full Ansible Networking Stack

With the environment activated, I now install everything needed for this guide. Notice I no longer need `--break-system-packages` — I'm inside the virtualenv and pip only touches this isolated environment.

### Upgrading pip First

```bash
# Always upgrade pip inside a fresh virtualenv before installing anything
pip install --upgrade pip
```

### Installing the Core Stack

I'll install everything in a logical order, explaining what each package is and why I need it.

```bash
# Core Ansible
pip install ansible

# Ansible linting and validation tools
pip install ansible-lint
pip install yamllint

# SSH connectivity libraries
pip install paramiko
pip install netmiko

# Vendor-neutral network abstraction
pip install napalm

# HTTP library for REST APIs (Netbox, AWX, PAN-OS)
pip install requests

# YAML parser (used in helper scripts)
pip install pyyaml

# Netbox Python client (for Netbox integration in Part 26)
pip install pynetbox

# Ansible Navigator (modern way to run and inspect playbooks)
pip install ansible-navigator
```

Or I can install everything in one command:

```bash
pip install ansible ansible-lint yamllint paramiko netmiko napalm requests pyyaml pynetbox ansible-navigator
```

**What each package does:**

| Package | Purpose |
|---|---|
| `ansible` | The core automation engine — includes `ansible-playbook`, `ansible`, `ansible-galaxy` |
| `ansible-lint` | Static analysis tool that checks playbooks for best practice violations |
| `yamllint` | YAML syntax and style checker — catches formatting issues before Ansible sees them |
| `paramiko` | Python SSH library — Ansible's default SSH backend for network devices |
| `netmiko` | Multi-vendor SSH library — underpins Ansible's `network_cli` connection plugin |
| `napalm` | Vendor-neutral network abstraction layer — used by `napalm_*` Ansible modules |
| `requests` | HTTP library — used for REST API calls to Netbox, AWX, PAN-OS |
| `pyyaml` | YAML parser — used in helper scripts and by Ansible itself |
| `pynetbox` | Netbox Python client — simplifies Netbox API interactions |
| `ansible-navigator` | Modern TUI for running and inspecting Ansible playbooks |

### Verifying the Installation

```bash
# Verify Ansible is installed and check the version
ansible --version
```

Expected output:
```
ansible [core 2.17.x]
  config file = None
  configured module search path = ['/home/ansible/.ansible/plugins/modules', ...]
  ansible python module location = /home/ansible/venvs/ansible-network/lib/python3.10/site-packages/ansible
  ansible collection location = /home/ansible/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/ansible/venvs/ansible-network/bin/ansible
  python version = 3.10.12 (main, ...) [GCC 11.4.0]
  jinja version = 3.1.x
  libyaml = True
```

The key line to confirm: `executable location` should point into my virtualenv (`/home/ansible/venvs/ansible-network/bin/ansible`), not `/usr/bin/ansible`.

```bash
# Verify other tools
ansible-lint --version
yamllint --version
python3 -c "import netmiko; print(netmiko.__version__)"
python3 -c "import napalm; print(napalm.__version__)"
python3 -c "import paramiko; print(paramiko.__version__)"
```

### ### ℹ️ Info

> `ansible` (the pip package) installs `ansible-core` plus a curated set of collections. `ansible-core` alone is a much smaller install — just the engine with no collections bundled. For this guide I install the full `ansible` package because it includes collections I'll use. In production environments with strict package management, some teams prefer `ansible-core` plus explicitly versioned collections to have full control over what's installed.

### Installing Ansible Collections

Collections are Ansible's plugin/module packages. They're installed separately from pip packages using `ansible-galaxy`:

```bash
# Install network vendor collections
ansible-galaxy collection install cisco.ios
ansible-galaxy collection install cisco.nxos
ansible-galaxy collection install junipernetworks.junos
ansible-galaxy collection install paloaltonetworks.panos

# Install utility collections
ansible-galaxy collection install ansible.netcommon
ansible-galaxy collection install ansible.utils
ansible-galaxy collection install netbox.netbox

# Verify installed collections
ansible-galaxy collection list
```

### ### 💡 Tip

> Collections installed with `ansible-galaxy` go to `~/.ansible/collections/` by default — outside the virtualenv. This is intentional: collections are Ansible-level packages, not Python-level packages. They stay in place even if I recreate the virtualenv. I can change the install path with `--collections-path` or by setting `collections_paths` in `ansible.cfg`. For now, the default location is fine.

### ### 🏢 Real-World Scenario

> In enterprise environments, direct internet access from the Ansible control node is often blocked by a corporate proxy or firewall. In those cases, pip packages are served from an internal PyPI mirror (like Nexus or Artifactory) and Ansible collections come from an internal Ansible Automation Hub rather than the public Galaxy. The commands are the same — I just need to configure pip and ansible-galaxy to point at the internal mirrors. This is something to flag in any enterprise onboarding: "Where is your internal PyPI mirror and do I need proxy settings?"

---

## 3.8 — Freezing Dependencies with `requirements.txt`

Right now my virtual environment has a specific set of packages at specific versions. If I delete the environment, move to a new VM, or hand this project to a colleague, I need a way to recreate it exactly. That's what `requirements.txt` is for.

### Generating `requirements.txt`

```bash
# Make sure the virtualenv is activated
# Then freeze all installed packages and their versions
pip freeze > requirements.txt

# View the result
cat requirements.txt
```

Example output:
```
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
```

**`pip freeze`** captures every single installed package — including dependencies of my dependencies. This guarantees a perfectly reproducible environment.

### ### ℹ️ Info

> The output of `pip freeze` includes packages I didn't explicitly install — like `cffi`, `cryptography`, `MarkupSafe`. These are **transitive dependencies** — packages that `paramiko`, `ansible`, or `requests` depend on. Freezing them ensures that even sub-dependency versions are locked, preventing the scenario where a sub-dependency gets updated and silently breaks something.

### Two Styles of `requirements.txt`

There are two common approaches, and both have their place:

**Style 1: Pinned versions (from `pip freeze`) — Maximum reproducibility**
```
ansible==9.8.0
ansible-lint==24.9.2
netmiko==4.4.0
paramiko==3.5.0
```
Use this for production environments where I need guaranteed stability. Every engineer gets the exact same versions.

**Style 2: Minimum versions — Maximum flexibility**
```
ansible>=9.0.0
ansible-lint>=24.0.0
netmiko>=4.0.0
paramiko>=3.0.0
```
Use this for open-source or shared projects where I want to allow newer compatible versions. Less strict, but more likely to get the latest bug fixes.

**My recommendation for this guide:** Use pinned versions from `pip freeze` for the actual project environment, and keep a separate loose `requirements-dev.txt` for quick new environment setups where exact versions matter less.

### ### 💡 Tip

> I commit `requirements.txt` to Git every time I add or upgrade a package. The git history then shows exactly when each package was added or updated and by whom. This is invaluable for debugging: "Ansible started failing on Thursday — let me check what changed in `requirements.txt` on Thursday."

---

## 3.9 — Recreating an Environment from `requirements.txt`

This is the payoff for maintaining `requirements.txt`. Whether I'm setting up a new VM, onboarding a colleague, or recovering from a broken environment, I can recreate everything in minutes.

### The Full Workflow

```bash
# Step 1: Clone the project repository (covered in Part 4)
git clone git@github.com:myusername/ansible-network.git
cd ansible-network

# Step 2: Create a fresh virtual environment
python3 -m venv ~/venvs/ansible-network

# Step 3: Activate it
source ~/venvs/ansible-network/bin/activate

# Step 4: Upgrade pip
pip install --upgrade pip

# Step 5: Install all packages from requirements.txt
pip install -r requirements.txt

# Step 6: Reinstall Ansible collections
ansible-galaxy collection install -r collections/requirements.yml

# Step 7: Verify
ansible --version
```

The `-r requirements.txt` flag tells pip to read the file and install everything listed in it. The entire environment is rebuilt in one command.

### Managing Collections with `requirements.yml`

Just as `requirements.txt` tracks Python packages, `collections/requirements.yml` tracks Ansible collections:

```bash
mkdir -p ~/projects/ansible-network/collections
```

Create `~/projects/ansible-network/collections/requirements.yml`:

```yaml
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
```

Install from this file:
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### ### ⚠️ Warning

> When recreating an environment from `requirements.txt`, if I get errors about conflicting package versions, the most common cause is that a package in `requirements.txt` has been yanked from PyPI (the author removed it for a critical bug) or my Python version differs from the one used to generate the file. The fix is to relax the version pin for the problematic package: change `==9.8.0` to `>=9.0.0` and let pip resolve a compatible version.

---

## 3.10 — Setting Up VS Code to Use the Virtual Environment

VS Code needs to know which Python interpreter to use — specifically, the one inside my virtual environment. Without this, VS Code's IntelliSense, linting, and the Ansible extension won't see the packages I've installed.

### Selecting the Python Interpreter in VS Code

With VS Code connected to my Ubuntu VM via Remote-SSH (from Part 1):

1. Press `Ctrl+Shift+P` to open the Command Palette
2. Type `Python: Select Interpreter` and press Enter
3. VS Code scans for available Python environments and shows a list
4. I select the one pointing to my virtualenv:
   ```
   Python 3.10.12 ('ansible-network': venv)
   ~/venvs/ansible-network/bin/python3
   ```

If VS Code doesn't detect it automatically, I click **Enter interpreter path...** and type the full path:
```
/home/ansible/venvs/ansible-network/bin/python3
```

### Configuring VS Code Settings for the Project

I add a `.vscode/settings.json` file to my project directory so the interpreter setting is saved with the project and applies automatically when anyone opens it:

```bash
mkdir -p ~/projects/ansible-network/.vscode
```

Create `~/projects/ansible-network/.vscode/settings.json`:

```json
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
```

**Line by line:**
- `python.defaultInterpreterPath` — tells the Python extension which interpreter to use
- `ansible.python.interpreterPath` — tells the Ansible extension which interpreter to use (must match)
- `ansible.validation.enabled` — enables real-time playbook validation in the editor
- `ansible.validation.lint.enabled` — enables ansible-lint integration in VS Code
- `yaml.schemas` — maps my playbook and task files to the Ansible YAML schema for validation
- `editor.tabSize: 2` — YAML best practice is 2-space indentation
- `editor.insertSpaces: true` — uses spaces, not tabs (tabs in YAML are illegal)
- `"[yaml]": { "editor.formatOnSave": true }` — auto-formats YAML files on save

### ### 🔴 Danger

> YAML does **not** allow tab characters for indentation — only spaces. If VS Code is configured to insert tabs (`editor.insertSpaces: false`) in YAML files, every playbook I save will be silently broken. Always verify `editor.insertSpaces` is `true` and `editor.tabSize` is `2` for YAML files. The `indent-rainbow` extension (from Part 1) will highlight incorrect indentation visually.

### Verifying VS Code Sees the Right Environment

After selecting the interpreter, I open a new VS Code terminal (`Ctrl+`` `). The virtual environment should activate automatically. I verify:

```bash
which ansible
# /home/ansible/venvs/ansible-network/bin/ansible  ← correct

which python3
# /home/ansible/venvs/ansible-network/bin/python3  ← correct
```

### ### 💡 Tip

> The `.vscode/settings.json` file should be committed to Git so the entire team gets the same VS Code configuration automatically. However, if some team members have the virtualenv at a different path (e.g., on macOS it would be different), the hardcoded path becomes a problem. A cleaner approach for cross-platform teams is to use a relative path or the `${workspaceFolder}` VS Code variable and keep the virtualenv inside the project directory itself. For this guide's single-user setup, the absolute path is fine.

---

## 3.11 — Best Practices Summary

### One Virtual Environment Per Project

```bash
# ✅ Correct — separate environments for separate projects
~/venvs/ansible-network/       ← This guide's project
~/venvs/legacy-nornir-scripts/ ← Older project using different library versions
~/venvs/netbox-api-tools/      ← Standalone Netbox scripts

# ❌ Wrong — one environment for everything
~/venvs/python-env/            ← All projects crammed in here, dependency chaos
```

### Never Commit the `venv/` Directory to Git

The virtual environment directory is large (often 50–200MB), contains compiled binaries specific to my OS, and is completely regeneratable from `requirements.txt`. It must never go into Git.

I add it to `.gitignore` (covered in detail in Part 4):

```
# .gitignore
venv/
venvs/
.venv/
env/
```

### Always Pin Versions in Production

```bash
# After setting up the environment and confirming everything works:
pip freeze > requirements.txt
git add requirements.txt
git commit -m "Pin Python dependencies for ansible-network environment"
```

### Document the Setup Process in `README.md`

Every project should have a `README.md` that tells a new engineer exactly how to recreate the environment:

```markdown
## Setup

1. Clone the repository:
   git clone git@github.com:myusername/ansible-network.git

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
```

### ### 🏢 Real-World Scenario

> I once joined a team that had no `requirements.txt` and no README setup instructions. The only way to know which packages were needed was to look at what was installed on the existing control node — which had been accumulating packages for three years with no documentation. Setting up a second control node for redundancy took two days of trial and error. A proper `requirements.txt` and README would have made it a 10-minute job. This is the kind of thing that separates amateur automation from professional, maintainable infrastructure-as-code.

---

## 3.12 — Managing the Environment Over Time

### Adding a New Package

```bash
# Activate the environment
source ~/venvs/ansible-network/bin/activate

# Install the new package
pip install some-new-library

# Update requirements.txt immediately
pip freeze > requirements.txt

# Commit the change
git add requirements.txt
git commit -m "Add some-new-library for feature X"
```

### Upgrading Packages

```bash
# Upgrade a specific package
pip install --upgrade ansible

# Check what changed
pip freeze > requirements.txt
git diff requirements.txt    # See exactly what version numbers changed

# Commit if everything still works
git add requirements.txt
git commit -m "Upgrade ansible from 9.7.0 to 9.8.0"
```

### Upgrading All Packages at Once

```bash
# List outdated packages
pip list --outdated

# Upgrade all packages (use with caution)
pip install --upgrade $(pip freeze | cut -d= -f1)

# Test everything still works before freezing
ansible --version
ansible-lint --version

# Then freeze and commit
pip freeze > requirements.txt
```

### ### ⚠️ Warning

> Upgrading all packages at once with no testing is risky. Ansible minor version upgrades occasionally deprecate or change module behavior. My recommendation: upgrade packages one at a time or in small groups, run a test playbook against a Containerlab device after each upgrade, and only freeze `requirements.txt` once I've confirmed nothing broke. In production, this upgrade process should go through a test environment first.

### Deleting and Recreating a Broken Environment

If the virtualenv gets corrupted or I want a clean slate:

```bash
# Deactivate first if active
deactivate

# Delete the entire environment directory
rm -rf ~/venvs/ansible-network

# Recreate from scratch
python3 -m venv ~/venvs/ansible-network
source ~/venvs/ansible-network/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml

# Verify
ansible --version
```

This takes about 2-3 minutes and produces a perfectly clean environment. The virtualenv is truly disposable because everything needed to recreate it is in `requirements.txt`.

---

## 3.13 — Common Gotchas in This Section

### ### 🪲 Gotcha — `ansible-playbook: command not found` after reconnecting SSH

The virtualenv deactivates when the SSH session ends. Every time I reconnect, I must reactivate:

```bash
source ~/venvs/ansible-network/bin/activate
```

The fix is either to add this to `~/.bashrc` (auto-activates on every new shell) or to always start sessions inside a tmux session where the environment was already activated (from Part 1).

### ### 🪲 Gotcha — VS Code terminal doesn't activate the virtualenv

VS Code's integrated terminal sometimes doesn't pick up the `.bashrc` activation. I can fix this by opening VS Code settings and adding:

```json
"terminal.integrated.env.linux": {
    "VIRTUAL_ENV": "/home/ansible/venvs/ansible-network",
    "PATH": "/home/ansible/venvs/ansible-network/bin:${env:PATH}"
}
```

Or simply run `source ~/venvs/ansible-network/bin/activate` manually in the VS Code terminal at the start of each session.

### ### 🪲 Gotcha — `pip freeze` includes packages I don't recognize

`pip freeze` includes transitive dependencies — packages that my packages depend on. This is expected and correct. If I want a cleaner `requirements.txt` with only the packages I explicitly installed, I can use `pip-chill`:

```bash
pip install pip-chill
pip-chill > requirements-direct.txt    # Only direct dependencies
pip freeze > requirements.txt          # Full pinned lockfile (use this for production)
```

### ### 🪲 Gotcha — Ansible collections not found after recreating the environment

Collections are installed to `~/.ansible/collections/`, not inside the virtualenv. When I delete and recreate the virtualenv, the collections are still there. But if I set up on a new machine, I need to reinstall them:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

This is why I maintain both `requirements.txt` (Python packages) and `collections/requirements.yml` (Ansible collections) and document both in the `README.md`.

---


The environment is now fully set up and reproducible. Everything runs inside an isolated, documented, version-controlled Python environment. Part 4 takes this further — I put the project under Git version control so every change to my playbooks, inventory, and configuration is tracked, reversible, and shareable.


