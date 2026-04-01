---
draft: false
title: 'Workstation Setup'
description: ""
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 1
---

{{< badge "Ansible" >}}
{{< badge content="Linux" color="red" >}}
{{< badge content="Windows" color="yellow" >}}

---

I will be editing files using VSCode on my Windows desktop, connected via the Remote-SSH extension. This gives me the full IDE experience (syntax highlighting, linting, file explorer, integrated terminal) while keeping all files on the control node where Ansible actually runs.

---

## <span class="section-num">01</span> SSH Key - Windows

First, I setup SSH key authentication from my Windows pc to `ansible-ctrl`.

I generated a key pair with PowerShell.

{{< codeblock lang="powershell" syntax="powershell" copy=true >}}
ssh-keygen -t ed25519 -C "Windows-desktop > ansible-ctrl" -f $env:USERPROFILE\.ssh\ansible-ctrl
{{< /codeblock >}}

I left the passphrase empty for convenience since this is on my personal pc.

---

I then copied the public key to `ansible-ctrl`

{{< codeblock lang="powershell" syntax="powershell" copy=true >}}
type $env:USERPROFILE\.ssh\ansible-ctrl.pub | ssh nesto@ansible-ctrl "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Reads the public key and pipes it over SSH to append it to the `authorized_keys` file on `ansible-ctrl`
{{< /line-explain >}}

Then I tested the key.

{{< codeblock lang="powershell" syntax="powershell" copy=true >}}
ssh -i $env:USERPROFILE\.ssh\ansible-ctrl nesto@ansible-ctrl
{{< /codeblock >}}

It should connect without prompting for a password.

---

### <span class="section-num">01a</span> Windows SSH Config

Next, I updated the SSH config file on my Windows pc so that VSCode and the command line know how to connect to lab hosts without specifying the full details every time.

{{< codeblock file="C:\Users\Nesto\.ssh\config" >}}
Host ansible-ctrl
    HostName      ansible-ctrl
    User          user
    IdentityFile  ~/.ssh/ansible-ctrl

Host clab
    HostName      clab
    User          user
    IdentityFile  ~/.ssh/ansible-ctrl

Host gitea
    HostName      gitea
    User          user
    IdentityFile  ~/.ssh/ansible-ctrl

Host netbox
    HostName      netbox
    User          user
    IdentityFile  ~/.ssh/ansible-ctrl
{{< /codeblock >}}

Now I should be able to connect without a password.

---

### <span class="section-num">02</span> VSCode Setup

I then installed the Remote-SSH Extension. This extension connects VSCode to a remote machine over SSH, installs a lightweight server component on the remote, and then all file operations, terminal sessions, and extensions run on the remote machine. The Windows VSCode instance is just a frontend.

From within VSCode:

1. Press Ctrl+Shift+X to open the Extensions panel
2. Search for "Remote - SSH"
3. Install the extension by Microsoft

---

Next, I connected to `ansible-ctrl` VM.

1. Press Ctrl+Shift+P to open the Command Palette
2. Type "Remote-SSH: Connect to Host"
3. Select ansible-ctrl from the list
4. A new VCCode window opens, connected to the remote machine
5. When prompted for the platform select Linux

Once connected I opened the project directory.

1. Press Ctrl+Shift+P
2. Type "File: Open Folder"
3. Navigate to /home/nesto/network-automation-lab
4. Click Ok

Every file I open, edit, and save is on `ansible-ctrl`.

---

### <span class="section-num">02a</span> Extensions

{{< tool-card title="Ansible" subtitle="redhat.ansible" >}}
Official Red Hat extension. Provides syntax highlighting, autocompletion, hover documentation, and linting for Ansible playbooks, roles, and inventory files. Also validates YAML structure.
{{< /tool-card >}}

{{< tool-card title="YAML" subtitle="redhat.vscode-yaml" >}}
YAML lanugage support with schema validation. Catches indentation errors and malformed YAML.
{{< /tool-card >}}

{{< tool-card title="Better Jinja" subtitle="samuelcolvin.jinjahtml" >}}
Improved Jinja2 highlighting that understands Jinja2 inside YAML, HTML, and other file types.
{{< /tool-card >}}

{{< tool-card title="GitLens" subtitle="eamodio.gitlens" >}}
Supercharges VSCode's built-in Git support. Shows inline blame annotations (who changed a line and when), commit history, and file diff comparisons.
{{< /tool-card >}}

{{< tool-card title="indent-rainbow" subtitle="oderwat.indent-rainbow" >}}
Colors each indentation level differently. This extension makes indentation errors visually obvious at a glance.
{{< /tool-card >}}

{{< tool-card title="Even Better TOML" subtitle="tamasfe.even-better-toml" >}}
Syntax highlighting for TOML files. Useful if working with Containerlab configs or Python project files that use TOML
{{< /tool-card >}}

{{< tool-card title="Cisco IOS Syntax" subtitle="jamiewoodio.cisco" >}}
Syntax highlighting for Cisco IOS configuration files. Makes the startup configs in `containerlab/configs/` directory and backup files more readable.
{{< /tool-card >}}

{{< tool-card title="Material Icon Theme" subtitle="PKief.material-icon-theme" >}}
Replaces the default file icons with richer ones. YAML files, Jinja2 tempplates, and directories get distinct icons, making the file explorer easier to scan. (This is a local extension installed on the Windows pc)
{{< /tool-card >}}

---

### <span class="section-num">02b</span> Settings

I configured a few settings to make the editing experience a bit easier. These settings are store in a `.vscode/settings.json` file in the project directory and apply only when the project is open.

{{< codeblock file=".vscode/settings.json" syntax="json" lines="true" >}}
{
  // YAML settings
  "yaml.validate": true,
  "yaml.format.enable": true,
  "yaml.format.singleQuote": false,

  // Ansible settings
  "ansible.python.interpreterPath": "/home/nesto/network-automation-lab/.venv/bin/python3",
  "ansible.validation.enabled": true,
  "ansible.validation.lint.enabled": true,

  // File associations
  "files.associations": {
    "*.yml": "ansible",
    "*.yaml": "ansible",
    "*.j2": "jinja",
    "*.cfg": "cisco"
  },

  // Editor settings
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.detectIndentation": false,
  "editor.renderWhitespace": "boundary",
  "editor.rulers": [120],

  // Trim trailing whitespace on save (matches pre-commit hook)
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,

  // Exclude clutter from the file explorer
  "files.exclude": {
    "**/.git": true,
    "**/.venv": true,
    "**/__pycache__": true,
    "**/.vault": true
  }
}
{{< /codeblock >}}

{{< line-explain >}}
Line 8:
: Points the Ansible extension to my project's virtual environment Python interpreter. This ensures the extension can find Ansible, ansible-lint, and the other installed collections for autocompletion and validation.

Lines 13-17:
: File associations tell VSCode which language mode to use for each file type. 

Lines 20-22:
: Forces 2-space indentation with spaces (never tabs). `detectIndentation: false` prevents VSCode from guessing and overriding this setting based on existing file content.

Lines 26-27:
: These match the `trailing-whitespace` and `end-of-file-fixer` precommit hooks.

Lines 30-34:
: Hides directories that shouldn't appear in the file explorer.
{{< /line-explain >}}

---

The Ansible extension's linting feature depends on `ansible-lint` being installed in the Python environment.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
pip install ansible-lint
pip freeze > requirements.txt
{{< /codeblock >}}

The Ansible extension should now automatically start showing linting warnings and errors inline as I type.

---

### <span class="section-num">02c</span> Integrated Terminal

The integrated terminal in VSCode opens a shell on `ansible-ctrl` when connected via Remote-SSH.

Because I setup `~/.bashrc` to auto-activate the virtual environment, the terminal should show the `(.venv)` prompt immediately. If it doesn't it's because the `~/bashrc` file isn't being sourced.

To fix it, I add the activation code to `.bash_profile` as well, or set the terminal profile in VSCode settings:

{{< codeblock file=".vscode/settings.json" >}}
"terminal.integrated.defaultProfile.linux": "bash",
"terminal.integrated.profiles.linux": {
  "bash": {
    "path": "/bin/bash",
    "args": ["--login"]
  }
}
{{< /codeblock >}}

---

### <span class="section-num">03</span> Other Tools

**MobaXterm**

I use this to quickly SSH into a VM if I don't have VSCode running.

**WinSCP**

I use this to quickly transfer files from my Windows PC to any VM.

---

### <span class="section-num">04</span> Workflow

1. Open VSCode
2. Connect to `ansible-ctrl`
3. Create a feature branch: `git checkout -b feat/name-of-change
4. Edit playbooks, roles, etc.
5. Run playbooks from the terminal
6. Review changes in the Source Control panel (Ctrl+Shift+G)
7. Stage, commit, push from terminal
8. Open Gitea, create a PR, review, merge