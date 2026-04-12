---
draft: false
title: '3. Ansible Vault'
description: "Part 3 of my Ansible learning geared towards Network Engineering."
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
{{< badge content="Git" color="blue" >}}
{{< badge content="Linux" color="red" >}}

---

Here I'll setup Ansible Vault which will include multiple vault IDs, encrypted variable files, the plaintext/vault naming pattern, and a pre-commit framework to catch secrets before they reach Git.

{{< objectives title="What I Will Be Completing In This Part" >}}
- Design a multi-vault ID strategy separating network credentials from service credentials
- Create the `.vault/` directory with per-ID password files
- Configure `ansible.cfg` to reference vault password files automatically
- Create encrypted variable files using the plaintext/vault naming pattern
- Install the `pre-commit` framework and configure hooks to block plaintext secrets
- Verify the entire secrets workflow end to end
{{< /objectives >}}

Ansible Vault is Ansible's built-in mechanism for encrypting sensitve data so they can be stored in Git without exposing them in plaintext. The encrypted files are committed alongside playbooks and roles, which means the entire project is version controlled. The secrets are only readable by someone who has the vault password.

If I were to not use Vault, I would need to keep credentials out of the repo entirely which breaks reproducibility.

---

## Vault ID

I set up multiple vault IDs instead of just using a single vault password. Vault IDs let me use different encryption passwords for different categories of secrets.

For this project I defined 3 vault IDs:

{{< kv title="" >}}
network | Device credentials: SSH passwords, enable secrets, SNMP strings
services       | Infrastructure service credentials: DB passwords, API tokens, admin accounts
pki   |  Certificates and private keys
{{< /kv >}}

Further explanation. The `network` vault holds credentials that Ansible uses to connect to and configure network devices. The `services` vault holds credentials for infrastucture platforms (e.g. Gitea, Netbox, Graylog). The `pki` vault will hold TLS certificates and private keys when step-ca is deployed.

---

## Directory Structure

I created a `.vault/` directory in the project root tohold the vault password files.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
cd ~/network-automation-lab
mkdir -p .vault
chmod 700 .vault
{{< /codeblock >}}

{{< line-explain >}}
Line 3:
: Set permissions to `700` that way only the owner can read, write, or list the contents of this directory.
{{< /line-explain >}}

---

## Password Files

I created one password file per vault ID. Each file contains a single line which is the vault password for that ID.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
(.venv) $ openssl rand -base64 32 > .vault.network.pass
(.venv) $ openssl rand -base64 32 > .vault/services.pass
(.venv) $ openssl rand -base64 32 > .vault/pki.pass
{{< /codeblock >}}

Then locked down permissions:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
chmod 600 .vault/*.pass
{{< /codeblock >}}

Each file should report exactly 1 line:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ wc -l .vault/*.pass
{{< /codeblock >}}

---

## Configuring Ansible.CFG

Next, I updated the `ansible.cfg` file to tell Ansible where to find the vault password files for each vault ID. This eliminates having to type `--vault-id` flags.

I added the following under the `[defaults]` section:

{{< codeblock file="ansible.cfg" syntax="cfg" >}}
vault_identity_list = network@.vault/network.pass, services@.vault/services.pass, pki@.vault/pki.pass
{{< /codeblock >}}

The `[defaults]` section afterwards:

{{< codeblock file="ansible.cfg" syntax="ini" >}}
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
vault_identity_list = network@.vault/network.pass, services@.vault/services.pass, pki@.vault/pki.pass

[persistent_connection]
connect_timeout   = 60
command_timeout   = 60
{{< /codeblock >}}

The file is committed to Git and containers the paths to the vault password files, but not the passwords themselves.

---

## Encrypted Variables

Next, I created encrypted variable files for my network device credentials. These files will be added to the `inventory/group_vars/` directory.

I then created the encrypted vault file for the `ios` group:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-vault create --encrypt-vault-id network inventory/group_vars/ios/vault.yml
{{< /codeblock >}}

{{< line-explain >}}
create:
: Opens a new file in the default editor and encrypts the contents on save.

--encrypt-vault-id network:
: Specifies which vault ID to use for encrypting the file.

{{< /line-explain >}}

Running this command will open my default editor, which is nano.

{{< codeblock file="inventory/group_vars/ios/vault.yml" syntax="ini" >}}
---
vault_ansible_user: admin
vault_ansible_password: superpassword2026
vault_ansible_become_password: supersecretpassword2026
{{< /codeblock >}}

After saving the file will be fully encrypted.

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ cat inventory/group_vars/ios/vault.yml
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
ANSIBLE_VAULT;1.2;AES256;network
33363237326239373033313861363034373166333039626436646432613835383733613738313930
6232363866663332333866396638613136383337626138650a373431303830333164393835396534
{{< /codeblock >}}

I did the same for NX-OS and PAN-OS groups:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ mkdir -p inventory/group_vars/nxos
(.venv) $ ansible-vault create --encrypt-vault-id network inventory/group_vars/nxos/vault.yml

(.venv) $ mkdir -p inventory/group_vars/panos
(.venv) $ ansible-vault create --encrypt-vault-id network inventory/group_vars/panos/vault.yml
{{< /codeblock >}}

Then created a services vault file for infrastructure credentials:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ mkdir -p inventory/group_vars/all
(.venv) $ ansible-vault create --encrypt-vault-id network inventory/group_vars/all/vault.yml
{{< /codeblock >}}

The `all` group is a built-in Ansible group that includes every host in the inventory.

---

## Plaintext/Vault Variable Pattern

This is the most important convention in the entire secrets management setup. I used the Ansible best practice of separating vault encrypted variables from the plaintext variables that reference them.

In each group's directory there are 2 files.

The encrypted `vault.yml` defines variables with a `vault_` prefix:

{{< codeblock file="inventory/group_vars/ios/vault.yml" syntax="yaml" >}}
---
vault_ansible_user: admin
vault_ansible_password: superpassword123!
vault_ansible_become_password: supersecretpassword123!
{{< /codeblock >}}

The plaintext `vars.yml` references those vault variables using Jinja2:

{{< codeblock file="inventory/group_vars/ios/vars.yml" syntax="yaml" lines="true" >}}
---
ansible_user: "{{ vault_ansible_user }}"
ansible_password: "{{ vault_ansible_password }}"
ansible_become_password: "{{ vault_ansible_become_password }}"
ansible_connection: ansible.netcommon.network_cli
ansible_network_os: cisco.ios.ios
ansible_become: true
ansible_become_method: enable
{{< /codeblock >}}

{{< line-explain >}}
Line2 2-4:
: The connection variables are set to Jinja2 expressions that pull their values from the vault encrypted variables.

Lines 5-8:
: Non-sensitive connection settings live in the plaintext file alongside the vault references.
{{< /line-explain >}}

I created the same for NX-OS and PAN-OS groups.

{{< codeblock file="inventory/group_vars/nxos/vars.yml" syntax="yaml" >}}
---
ansible_user: "{{ vault_ansible_user }}"
ansible_password: "{{ vault_ansible_password }}"
ansible_become_password: "{{ vault_ansible_become_password }}"
ansible_connection: ansible.netcommon.network_cli
ansible_network_os: cisco.nxos.nxos
ansible_become: true
ansible_become_method: enable
{{< /codeblock >}}

{{< codeblock file="inventory/group_vars/panos/vars.yml" syntax="yaml" >}}
---
ansible_user: "{{ vault_ansible_user }}"
ansible_password: "{{ vault_ansible_password }}"
ansible_connection: local
ansible_network_os: paloaltonetworks.panos.panos
{{< /codeblock >}}

{{< line-explain >}}
Line 4:
: PAN-OS uses `local` connection and not `network_cli`. The PAN-OS Ansible collection communicates with the firewall over its XML API (HTTPS).
{{< /line-explain >}}

---

## Encrypt, Decrypt, and Edit

These are useful commands that I use daily while working with vault encrypted files.

---

**Editing an existing vault file:**

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-vault edit inventory/group_vars/ios/vault.yml
{{< /codeblock >}}

This decrypts the file, opens it in the editor, and re-encrypts it on save.

---

**Viewing contents without editing:**

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-vault view inventory/group_vars/ios/vault.yml
{{< /codeblock >}}

This prints the decrypted contents to stdout without opening an editor.

---

**Encrypting an existing plaintext file:**

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-vault encrypt --encrypt-vault-id network somefile.yml
{{< /codeblock >}}

This encrypts a file in place. The original plaintext is overwritten with the encrypted version.

---

**Encrypting a single string:**

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-vault encrypt_string --encrypt-vault-id services 'MySecretToken' --name 'vault_netbox_api_token'
{{< /codeblock >}}

---

## Pre-Commit

I then installed the `pre-commit` framework and configured hooks that prevent plaintext secrets from being committed to Git. This runs checks automatically before every `git commit`. If the check fails then the commit blocked and an error message appears.

`pre-commit` supports hundreds of community maintained hooks. I'll extend its usage by using it with YAML lint and Ansible syntax validation.

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ pip install pre-commit
(.venv) $ pip freeze > requirements.txt
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Installed `pre-commit`.

Line 2:
: Updated `requirements.txt` to include `pre-commit`.
{{< /line-explain >}}

Then I created the `.pre-commit-config.yml` configuration file:

{{< codeblock file=".pre-commit-config.yaml" syntax="yaml" lines="true" >}}
---
repos:
  # General file hygiene
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: detect-private-key
      - id: no-commit-to-branch
        args: ['--branch', 'main']

  # Block unencrypted Ansible Vault files
  - repo: local
    hooks:
      - id: check-ansible-vault
        name: Check for unencrypted vault files
        entry: bash -c 'for f in "$@"; do if echo "$f" | grep -q "vault\.yml$"; then head -1 "$f" | grep -q "^\$ANSIBLE_VAULT" || { echo "ERROR: $f is not encrypted!"; exit 1; }; fi; done'
        language: system
        files: vault\.yml$
        types: [file]

      - id: check-no-plaintext-passwords
        name: Check for plaintext password patterns
        entry: bash -c 'grep -rlnP "(?i)(password|secret|token|community)\s*[:=]\s*(?!\s*[\"{]\{)" "$@" | grep -v "vault\.yml" | grep -v "\.pre-commit" && { echo "ERROR: Possible plaintext secret detected!"; exit 1; } || exit 0'
        language: system
        files: \.(yml|yaml)$
        types: [file]
{{< /codeblock >}}

{{< line-explain >}}
Lines 4-14:
: The `pre-commit-hooks` repo provides common checks.

Lines 17-24:
: A custom local hook that checks every staged file named `vault.yml`. It reads the first line and verifies it starts with `$ANSIBLE_VAULT`. If the header is missing the file is plaintext and the commit is blocked.

Lines 26:32
: A second custom hook that scans all staged YAML files (ecluding `vault.yml` files and the pre-commit config itself) for patterns that look like plaintext secrets.
{{< /line-explain >}}

I then installed the hooks into the local Git repo:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ pre-commit install
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
pre-commit installed at .git/hooks/pre-commit
{{< /codeblock >}}

{{< lab-callout type="warning" >}}
`pre-commit install` modifies the `.git/hooks/pre-commit` file, which is local to this clone and not tracked by Git. This means the hooks need to be reinstalled after cloning the repo on a new machine. Running `pre-commit install` should be part of the setup steps whenever the repo is cloned fresh.
{{< /lab-callout >}}

I ran the hooks against all existing files to make sure nothing in the current project triggers a failure:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ pre-commit run --all-files
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
Trim Trailing Whitespace.............................Passed
Fix End of Files.....................................Passed
Check Yaml...........................................Passed
Check for added large files..........................Passed
Detect Private Key...................................Passed
Don't commit to branch...............................Passed
Check for unencrypted vault files....................Passed
Check for plaintext password patterns................Passed
{{< /codeblock >}}

---

## Testing Hooks

I tested the hooks just to see what errors I would get and familiarlize myself with them.

**1. Unencrypted vault file**

I created a plaintext file named `vault.yml` to simulate a decrypted vault file being staged:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ echo "vault_test_password: plaintext_oops" > /tmp/test-vault.yml
(.venv) $ cp /tmp/test-vault.yml inventory/group_vars/ios/vault.yml
(.venv) $ git add inventory/group_vars/ios/vault.yml
(.venv) $ git commit -m "test: should be blocked"
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
Check for unencrypted vault files....................Failed
- hook id: check-ansible-vault
- exit code: 1

ERROR: inventory/group_vars/ios/vault.yml is not encrypted!
{{< /codeblock >}}

The commit was blocked. I restored the encrypted file:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git checkout -- inventory/group_vars/ios.vault.yml
{{< /codeblock >}}

---

**2. Plaintext password in a YAML file**

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ echo "snmp_community: public123" > test-secret.yml
(.venv) $ git add test-secret.yml
(.venv) $ git commit -m "test: should be blocked"
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
Check for plaintext password patterns................Failed
- hook id: check-no-plaintext-passwords
- exit code: 1

ERROR: Possible plaintext secret detected!
{{< /codeblock >}}

Blocked again. I cleaned up:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git reset HEAD test-secret.yml
(.venv) $ rm test-secret.yml
{{< /codeblock >}}

---

**3. Direct commit to main**

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git checkout main
(.venv) $ echo "test" >> README.md
(.venv) $ git add README.md
(.venv) $ git commit -m "test: should be blocked"
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
Don't commit to branch...............................Failed
- hook id: no-commit-to-branch
- exit code: 1
{{< /codeblock >}}

I then reset.

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git checkout -- README.md
{{< /codeblock >}}

---

## Commit & Push

After testing everything I committed the vault setup and pre-commit configurationg using the feature branch workflow.

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git checkout -b feat/vault-and-precommit
(.venv) $ git add -A
(.venv) $ git status
{{< /codeblock >}}

Then reviewed the status output to see if anything needed to be corrected:

{{< codeblock lang="Expected Output" copy="false" >}}
new file:   .pre-commit-config.yaml
modified:   ansible.cfg
modified:   requirements.txt
new file:   inventory/group_vars/all/vault.yml
new file:   inventory/group_vars/ios/vars.yml
new file:   inventory/group_vars/ios/vault.yml
new file:   inventory/group_vars/nxos/vars.yml
new file:   inventory/group_vars/nxos/vault.yml
new file:   inventory/group_vars/panos/vars.yml
new file:   inventory/group_vars/panos/vault.yml
{{< /codeblock >}}

Then committed and pushed"

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git commit -m "feat: add ansible vault with multi-ID and pre-commit hooks"
(.venv) $ git push -u origin feat/vault-and-precommit
{{< /codeblock >}}

Then went to Gitea to review the Pull Request and approve it then merged it with `main`.

After merging:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git checkout main
(.venv) $ git pull origin main
{{< /codeblock >}}

---

Secrets are now a solved problem for this project. 