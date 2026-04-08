---
draft: false
title: '1. Project Structure'
description: "Part 1 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 1
---

{{< badge "Ansible" >}}
{{< badge content="Git" color="blue" >}}
{{< badge content="Linux" color="red" >}}

---

Setting up the Ansible control node, Python environment, project layout, and local version control.

{{< objectives title="What I Will Be Completing In This Part" >}}
- Provision an Ubuntu 22.04 VM as the dedicated Ansible control node
- Create an isolated Python virtual environment for Ansible
- Install Ansible and verify the installation
- Build the project directory structure for `network-automation-lab`
- Create the `ansible.cfg` project configuration
- Initialise a local Git repository and make the first commit
{{< /objectives >}}

---

## <span class="section-num">01</span> VM Specification

I created a new VM in my Proxmox server with the following specifications. This VM will be dedicated as the Ansible control node.

{{< codeblock lang="Specs" copy=false >}}
OS:        Ubuntu Server 22.04 LTS
Hostname:  ansible-ctrl
CPU:       2 vCPU
RAM:       4 GB
Disk:      40 GB
Network:   1 NIC (bridged to management network)
{{< /codeblock >}}

2 vCPU and 4GB is more than needed for an Ansible control node. I also gave this VM a static IP on the management network and added a DNS entry.

---

## <span class="section-num">02</span> System Setup

After installing the OS I ran a full system update and installed the base dependencies that Ansible and the Python virtual environment will need.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-venv python3-pip git sshpass libffi-dev libssl-dev
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Updates the package list and upgrades all installed packages.

Line 2:
: `python3-venv` provides the `venv` module for creating isolated environments. `git` is for version control. `sshpass` allows password-based SSH connections during initial device bootstrapping. `libffi-dev` and `libssl-dev` are C libraries required to compile `paramiko` and `cryptography`.
{{< /line-explain >}}

{{< lab-callout type="warning" >}}
`sshpass` is a bootstrap tool, not a long-term solution. I'm only using it for the initial connection to devices that don't yet have SSH keys configured. Once I deploy key-based authentication via Ansible in later parts, `sshpass` becomes unnecessary. In a production environment, password-based SSH access should be disabled as soon as keys are in place.
{{< /lab-callout >}}

---

## <span class="section-num">03</span> Python Virtual Environment

Creating a dedicated Python virtual environment is best practice because it prevents version conflicts with system Python packages. It also makes the Ansible installation reproducible, and it lets me pin specific versions of Ansible and its dependencies without affecting other tools on the VM.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
mkdir -p ~/network-automation-lab
cd ~/network-automation-lab
python3 -m venv .venv
source .venv/bin/activate
{{< /codeblock >}}

{{< line-explain >}}
Line 1–2:
: Created the project root directory and moved into it. I will create everything related to this project within this directory (laybooks, roles, inventory, configs, and the virtual environment itself).

Line 3:
: Created the virtual environment in a `.venv` directory.

Line 4:
: Activated the virtual environment. After this, `python3` and `pip` point to the copies inside `.venv/` rather than the system installation.
{{< /line-explain >}}

To avoid forgetting to activate the venv every time I SSH into the control node, I added the activation command to my `~/.bashrc`.

{{< codeblock file="~/.bashrc" >}}
# Auto-activate the Ansible venv when entering the project directory
if [ -d "$HOME/network-automation-lab/.venv" ]; then
    source "$HOME/network-automation-lab/.venv/bin/activate"
    cd ~/network-automation-lab
fi
{{< /codeblock >}}

On a shared machine, I would use `direnv` or a wrapper script instead, that way the venc only activates when explicitly entering the project directory.

---

## <span class="section-num">04</span> Installing Ansible

After activating the virtual environement, I installed Ansible using `pip`. I installed the full `ansible` package because it includes community collections that I'll need for network device modules.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
(.venv) pip install --upgrade pip
(.venv) pip install ansible paramiko
(.venv) pip freeze > requirements.txt
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Upgraded `pip` itself first. The version bundled with Ubuntu 22.04's Python is old and will print deprecation warnings otherwise.

Line 2:
: `ansible` pulls in `ansible-core` plus community collections. `paramiko` is the Python SSH library that Ansible uses as a fallback transport.

Line 3:
: Froze the exact package versions into `requirements.txt`. If I rebuild the control node six months from now, I can recreate the identical environment with `pip install -r requirements.txt`.
{{< /line-explain >}}

{{< lab-callout type="danger" >}}
Never install Ansible with `sudo pip install` outside a virtual environment. This modifies system Python packages and can break OS tools that depend on specific Python library versions (like `apt` itself on Ubuntu). Always use a venv.
{{< /lab-callout >}}

Next, I installed the specific Ansible collections I'll need for the 3 network platforms in this lab.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
(.venv) $ ansible-galaxy collection install cisco.ios
(.venv) $ ansible-galaxy collection install cisco.nxos
(.venv) $ ansible-galaxy collection install paloaltonetworks.panos
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: IOS-XE modules for the Cat9kv nodes.

Line 2:
: NX-OS modules for the N9Kv spine/leaf switches.

Line 3:
: PAN-OS collection for the PA-VM firewall.
{{< /line-explain >}}

To be able to reproduce this evironment more easily, I created a `requirements.yml` file to pin these collections. This makes it so I can reinstall all collections with 1 command: `ansible-galaxy collection install -r requirements.yml`.

{{< codeblock file="requirements.yml" >}}
---
collections:
  - name: cisco.ios
  - name: cisco.nxos
  - name: paloaltonetworks.panos
{{< /codeblock >}}

---

## <span class="section-num">05</span> Directory Structure

I like to setup the directory structure before writing playbooks, that way everything is layed out beforehand.

{{< codeblock lang="Bash" >}}
(.venv) mkdir -p inventory/group_vars inventory/host_vars
(.venv) mkdir -p playbooks
(.venv) mkdir -p roles
(.venv) mkdir -p templates
(.venv) mkdir -p files
(.venv) mkdir -p docs
(.venv) mkdir -p filter_plugins
(.venv) touch README.md
{{< /codeblock >}}

The directory tree should look like this:

{{< dir-tree >}}
<span class="dir">network-automation-lab/</span>
├── <span class="dir">inventory/</span>            <span class="desc"># Device inventory (static for now, dynamic with NetBox later)</span>
│   ├── <span class="dir">group_vars/</span>      <span class="desc"># Variables scoped to groups of devices</span>
│   └── <span class="dir">host_vars/</span>       <span class="desc"># Variables scoped to individual devices</span>
├── <span class="dir">playbooks/</span>            <span class="desc"># Task playbooks</span>
├── <span class="dir">roles/</span>                <span class="desc"># Reusable role packages</span>
├── <span class="dir">templates/</span>            <span class="desc"># Jinja2 templates for config generation</span>
├── <span class="dir">files/</span>                <span class="desc"># Static files to push to devices</span>
├── <span class="dir">filter_plugins/</span>       <span class="desc"># Custom Jinja2 filters (Python)</span>
├── <span class="dir">docs/</span>                 <span class="desc"># Project documentation</span>
├── <span class="file">.venv/</span>                <span class="desc"># Python virtual environment (git-ignored)</span>
├── <span class="file">ansible.cfg</span>           <span class="desc"># Project-level Ansible configuration</span>
├── <span class="file">requirements.txt</span>      <span class="desc"># Pinned Python dependencies</span>
├── <span class="file">requirements.yml</span>      <span class="desc"># Pinned Ansible collections</span>
└── <span class="file">README.md</span>             <span class="desc"># Project overview</span>
{{< /dir-tree >}}

This follows Ansible's recommended directory layout for a single project repo.

---

## <span class="section-num">06</span> Ansible Configuration

While in the project root I created an `ansible.cfg` file. When Ansible runs it searched for this configuration in this order: `ANSIBLE_CONFIG` environment variable, `ansible.cfg` in the current directory, `~/.ansible.cfg`, `/etc/ansible/ansible.cfg`.

{{< codeblock file="ansible.cfg" >}}
[defaults]
inventory         = inventory/
roles_path        = roles/
collections_path  = ~/.ansible/collections
remote_user       = admin
timeout           = 30
forks             = 10
host_key_checking = False
retry_files_enabled = False
stdout_callback   = yaml
callbacks_enabled = ansible.posix.timer

[persistent_connection]
connect_timeout   = 60
command_timeout   = 60
{{< /codeblock >}}

{{< line-explain >}}
Line 2:
: Points Ansible to the `inventory/` directory. It will read all valid inventory files in this directory automatically.

Line 4:
: Keeps collections in the default user location.

Line 5:
: Default SSH user for device connections.

Line 7:
: Number of parallel processes. `10` means Ansible will configure up to 10 devices simultaneously.

Line 8:
: Disables SSH host key checking. Necessary in a lab where devices are frequently rebuilt and their host keys change. In production, this should be `True`.

Line 9:
: Disables `.retry` files that Ansible creates on failed playbook runs.

Line 10:
: Switches playbook output from the default (dense, hard to read) to YAML format.

Line 11:
: Enables the timer callback plugin which displays how long each playbook run takes.

Lines 13–15:
: The `persistent_connection` section controls network device connections. Default 30-second timeouts cause failures on slower devices (especially N9Kv in Containerlab), so I increased both to 60 seconds.
{{< /line-explain >}}

`host_key_checking = False` is to make it easier for this lab project. If this were a real environment I would manage SSH known hosts properly. I could either create a pre-populated `known_hosts` file distributed via configuration management, or by using a host key verification machanism tied to the source of truth.

---

## <span class="section-num">07</span> Git Initialisation

I then initialized a local Git repo at the project root. I will be setting up Gitea later on.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
(.venv) git init
(.venv) git config user.name "Your Name"
(.venv) git config user.email "you@example.com"
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Created the `.git/` directory in the project root. This is now a tracked repository.

Lines 2–3:
: Set my identity for commits.
{{< /line-explain >}}

---

### <span class="section-num">07a</span> .gitignore

Before commiting anything I created the `.gitignore` file. This file will ommit any file or directory from being pushed to the repo.

{{< codeblock file=".gitignore" >}}
# Python virtual environment
.venv/
__pycache__/
*.pyc

# Ansible artifacts
*.retry

# Vault password file (NEVER commit this)
.vault/
.vault_pass
.vault_password

# IDE and editor files
.vscode/
*.swp
*.swo
*~

# OS artifacts
.DS_Store
Thumbs.db
{{< /codeblock >}}

The `.vault/` directory and vault password files are the most important since they contain the encryption key for all my secrets. If this file is committed to Git then the entire secret is compromised.

---

### <span class="section-num">07b</span> First Commit

After creating the directory structure, configuration files, and `.gitingore`, I staged everything and made the first commit.

{{< codeblock lang="Bash" >}}
(.venv) git add -A
(.venv) git status
{{< /codeblock >}}

Running `git status` before commiting is so I can verify that only the intended files are staged. The output should show the project files but not `.venv/`.

{{< codeblock lang="Expected Output" copy=false >}}
On branch main

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   .gitignore
        new file:   README.md
        new file:   ansible.cfg
        new file:   requirements.txt
        new file:   requirements.yml
{{< /codeblock >}}

The empty directories won't appear in `git status` because Git doesn't track empty directories. I could add a `.gitkeep` placeholder file to each but it's not that important.

{{< codeblock lang="Bash" >}}
(.venv) git commit -m "init: control node setup, ansible install, project structure"
{{< /codeblock >}}

- `init` is used for bootstrapping
- `feat` is used for new features
- `fix` is used for bug fixes
- `docs` is used for documentation
- `refractor` is used for restructuring without changing behavior

---

## <span class="section-num">07</span> Verification

I then ran a few checks to verify that everything is working correctly.

**Ansible Version**

{{< codeblock lang="Bash" >}}
(.venv) which python3
/home/nesto/network-automation-lab/.venv/bin/python3

(.venv) which ansible
/home/nesto/network-automation-lab/.venv/bin/ansible
{{< /codeblock >}}

Both paths should point to `.venv/bin/`, if they point to `/usr/bin/` then the virtual environment is not activated.

**Installed Collections**

{{< codeblock lang="Bash" >}}
(.venv) ansible-galaxy collection list | grep -E "cisco|paloalto"
{{< /codeblock >}}

This should show `cisco.ios.`, `cisco.nxos`, and `paloaltonetworks.panos`.

**Git Log**

{{< codeblock lang="Bash" >}}
(.venv) git log --oneline
a1b2c3d init: control node setup, ansible install, project structure
{{< /codeblock >}}

Should only show 1 commit.

---

After this I took a Proxmox snapshot of the VM. That way if anything messes up I can quickly rollback to a known working state of the VM.