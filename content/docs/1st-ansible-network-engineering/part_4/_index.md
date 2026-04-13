---
draft: false
title: '3 - Git'
description: "Part 3 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 3
---

{{< badge "Ansible" >}}
{{< badge content="Git" color="green" >}}
{{< badge content="Linux" color="red" >}}

---

{{< lab-callout type="info" >}}
This is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. Each part will build upon the last. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.
{{< /lab-callout >}}

---

## Git & GitHub

I already know the basics (add, commit, push). But there's a big difference between using Git and using Git well. Playbooks are infrastructure code. A bad commit that gets pushed and run in production can take down a network segment. A missing `.gitignore` can leak credentials to a public repository.

---

### Why Version Control

Before Git, network changes worked like this: an engineer SSHs into a device, makes changes, types `write mem`, and hopes nothing breaks. If it does break, the rollback is "remember what you changed and undo it manually." The config backup, if it exists, is a text file on a shared drive with a name like `R1-config-final-FINAL-v3-USE-THIS-ONE.txt`.

Ansible without version control isn't much better. I have a playbook that works today. I change it. It breaks something in production. I can't remember exactly what I changed or when.

Version control changes everything:

- **Every change is recorded**: who changed what, when, and why
- **Every change is reversible**: I can roll back to any previous state in seconds
- **Changes are reviewed before they run**: Pull Requests mean a second set of eyes before anything touches production
- **The history is the documentation**: commit messages explain decisions that comments in code never would
- **Collaboration is structured**: multiple engineers can work on the same codebase without overwriting each other

---

## Installing and Configuring Git

{{< subtle-label >}}Installing Git{{< /subtle-label >}}

First, I updated the packages and then install git.

{{< codeblock lang="Bash" syntax="bash" >}}
sudo apt update
sudo apt install -y git
{{< /codeblock >}}

Verify:

{{< codeblock lang="Bash" syntax="bash" >}}
git --version
# git version 2.34.x
{{< /codeblock >}}

---

{{< subtle-label >}}Configuring Git Identity{{< /subtle-label >}}

The first thing I do after installing Git is tell it who I am. Every commit I make is stamped with this name and email, it's how the team knows who made each change.

{{< codeblock lang="Bash" syntax="bash" >}}
git config --global user.name "Ernesto Diaz"
git config --global user.email "myemail@domain.com"
{{< /codeblock >}}

Without this configuration, Git will either refuse to commit or use a default that looks unprofessional in the team's commit history.

---

{{< subtle-label >}}Setting the Default Branch Name{{< /subtle-label >}}

GitHub's default branch name is `main`. Older Git versions default to `master`. I align them to avoid confusion:

{{< codeblock lang="Bash" syntax="bash" >}}
git config --global init.defaultBranch main
{{< /codeblock >}}

---

{{< subtle-label >}}Setting the Default Editor{{< /subtle-label >}}

When Git needs me to write a commit message in an editor (e.g., during a merge), it opens the default editor. I set it to nano since it's what I'm used to:

{{< codeblock lang="Bash" syntax="bash" >}}
git config --global core.editor nano
{{< /codeblock >}}

---

{{< subtle-label >}}Setting Up a Credential Helper{{< /subtle-label >}}

When I push to GitHub over HTTPS (before SSH keys are set up), Git will ask for my credentials. The credential helper caches them so I'm not asked every single time:

{{< codeblock lang="Bash" syntax="bash" >}}
git config --global credential.helper store
{{< /codeblock >}}

{{< lab-callout type="danger" >}}
`credential.helper store` saves credentials in plaintext at `~/.git-credentials`. This is acceptable for a private VM on a home lab network, but not for a shared server or any machine others have access to. On shared machines, use `credential.helper cache` instead, it stores credentials in memory for a configurable timeout (default 15 minutes) and never writes them to disk.
{{< /lab-callout >}}

---

### Verifying the Configuration

{{< codeblock lang="Bash" syntax="bash" >}}
git config --global --list
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
user.name=First Last
user.email=myemail@company.com
init.defaultBranch=main
core.editor=nano
credential.helper=store
{{< /codeblock >}}

All global Git settings are stored in `~/.gitconfig`:

{{< codeblock lang="Bash" syntax="bash" >}}
cat ~/.gitconfig
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
[user]
    name = First Last
    email = myemail@company.com
[init]
    defaultBranch = main
[core]
    editor = nano
[credential]
    helper = store
{{< /codeblock >}}

---

## Connecting VM to GitHub via SSH

I want Git operations to work without typing a password every time. The cleanest way is SSH key authentication between my Ubuntu VM and GitHub. The same concept as SSH keys for server access, but this time the "server" is GitHub.

---

{{< subtle-label >}}Generating an SSH Key{{< /subtle-label >}}

This key is specifically for GitHub. I generate it on the Ubuntu VM itself:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
ssh-keygen -t ed25519 -C "myemail@company.com" -f ~/.ssh/github_key
{{< /codeblock >}}

{{< line-explain >}}
-t ed25519:
: Ed25519 is the modern, recommended key type for GitHub

-C "myemail@company.com:
: the comment becomes a label in GitHub's SSH key list, helping me identify which key is which

-f ~/.ssh/github_key:
: saves the key pair as `github_key` (private) and `github_key.pub` (public)
{{< /line-explain >}}

When prompted for a passphrase, I set one.

---

{{< subtle-label >}}Setting Correct Permissions{{< /subtle-label >}}

{{< codeblock lang="Bash" syntax="bash" >}}
chmod 600 ~/.ssh/github_key
chmod 644 ~/.ssh/github_key.pub
{{< /codeblock >}}

---

{{< subtle-label >}}Configuring SSH to Use This Key for Github{{< /subtle-label >}}

I edit create `~/.ssh/config` on the Ubuntu VM to tell SSH which key to use when connecting to GitHub:

{{< codeblock lang="Bash" syntax="bash" >}}
nano ~/.ssh/config
{{< /codeblock >}}

I add the following:

{{< codeblock file="/.ssh/config" syntax="ini" >}}
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_key
    IdentitiesOnly yes
{{< /codeblock >}}

I set correct permissions on the config file:

{{< codeblock lang="Bash" syntax="bash" >}}
chmod 600 ~/.ssh/config
{{< /codeblock >}}

---

{{< subtle-label >}}Adding the SSH Agent{{< /subtle-label >}}

Since my key has a passphrase, I'd have to type it on every Git push without an SSH agent. The agent holds the decrypted key in memory for the session:

To start the SSH agent:

{{< codeblock lang="Bash" syntax="bash" >}}
eval "$(ssh-agent -s)"
{{< /codeblock >}}

Add my GitHub key to the agent (will prompt for passphrase once):

{{< codeblock lang="Bash" syntax="bash" >}}
ssh-add ~/.ssh/github_key
{{< /codeblock >}}

Verify it's loaded

{{< codeblock lang="Bash" syntax="bash" >}}
ssh-add -l
{{< /codeblock >}}

To make this automatic on every new shell session, I add it to `~/.bashrc`:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
cat >> ~/.bashrc << 'EOF'

# Start SSH agent and add GitHub key if not already running
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval "$(ssh-agent -s)" > /dev/null
    ssh-add ~/.ssh/github_key 2>/dev/null
fi
EOF

source ~/.bashrc
{{< /codeblock >}}

{{< line-explain >}}
Line 4:
: checks if the SSH agent is already running. Without this check, a new agent process would start every time I open a terminal, leaking memory over time.
{{< /line-explain >}}

---

{{< subtle-label >}}Adding the Public Key to Github{{< /subtle-label >}}

Now I copy the public key and add it to my GitHub account.

{{< codeblock lang="Bash" syntax="bash" >}}
cat ~/.ssh/github_key.pub
{{< /codeblock >}}

The output looks like:

{{< codeblock lang="Bash" syntax="bash" >}}
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... myemail@company.com
{{< /codeblock >}}

I copy this entire line, then:

1. Go to **GitHub.com** → click my profile picture → **Settings**
2. In the left sidebar, click **SSH and GPG keys**
3. Click **New SSH key**
4. **Title:** `ansible-ubuntu-vm`
5. **Key type:** Authentication Key
6. **Key:** paste the public key
7. Click **Add SSH key**

### Testing the Connection

{{< codeblock lang="Bash" syntax="bash" >}}
ssh -T git@github.com
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
Hi myusername! You've successfully authenticated, but GitHub does not provide shell access.
{{< /codeblock >}}

That message confirms SSH authentication to GitHub is working. The "does not provide shell access" part is normal. GitHub only accepts Git operations over SSH, not interactive shell sessions.

---

## Initializing the Project Repository

Now I set up the Git repository for the Ansible project I'll build throughout this lab.

---

### Creating the Project Directory Structure

Create the project directory

{{< codeblock lang="Bash" syntax="bash" >}}
mkdir -p ~/projects/ansible-network
cd ~/projects/ansible-network
{{< /codeblock >}}

Create the initial directory structure

{{< codeblock lang="Bash" syntax="bash" >}}
mkdir -p {playbooks,inventory/{group_vars,host_vars},roles,collections,templates,files,vars}
{{< /codeblock >}}

Verify the structure

{{< codeblock lang="Bash" syntax="bash" >}}
tree .
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
ansible-network/
├── collections/
│   └── requirements.yml
├── files/
├── inventory/
│   ├── group_vars/
│   ├── host_vars/
│   └── hosts.yml
├── playbooks/
├── roles/
├── templates/
└── vars/
{{< /codeblock >}}

---

{{< subtle-label >}}Initializing Git{{< /subtle-label >}}

{{< codeblock lang="Bash" syntax="bash" >}}
cd ~/projects/ansible-network
git init
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
Initialized empty Git repository in /home/ansible/projects/ansible-network/.git/
{{< /codeblock >}}

Git creates a hidden `.git/` directory that contains the entire history of the repository. I never manually edit anything inside `.git/`.

Check the current state of the repository:

{{< codeblock lang="Bash" syntax="bash" >}}
git status
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        collections/
        inventory/
        playbooks/

nothing added to commit but untracked files present
{{< /codeblock >}}

"Untracked files" means Git sees these files exist but isn't tracking changes to them yet. Nothing is version-controlled until I explicitly `git add` it. This is by design, Git never assumes I want to track a file. I choose what gets tracked.

---

### Creating the .gitignore File

Before I make a single commit, I create the `.gitignore` file. This is non-negotiable since it prevents secrets, temporary files, and regeneratable artifacts from ever entering the repository.

{{< codeblock lang="Bash" syntax="bash" >}}
nano ~/projects/ansible-network/.gitignore
{{< /codeblock >}}

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
venv/
.venv/
env/
venvs/
__pycache__/
*.py[cod]
*.pyc
*.pyo
*.pyd
.Python

*.retry

.ansible/
fact_cache/

.vault_pass
.vault_password
vault_pass.txt
*.vault_pass

.env
.env.*
*.env

secrets.yml
secrets.yaml
secrets/

*.pem
*.key
id_rsa
id_ed25519
*_key
!*_key.pub

.vscode/extensions.json
.history/

*.swp
*.swo
*~

*.log
*.tmp
*.temp
/tmp/

.pytest_cache/
.coverage
htmlcov/
.tox/

tower_export/
awx_export/

# host_vars/
# group_vars/
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: The venv is outside this project directory (~/venvs/) but adding these patterns protects against accidental venv creation inside the project.

Line 12:
: Retry files are created when a playbook fails mid-run. They contain hostnames and should never be committed.

Line 15:
: Ansible fact cache can contain sensitive device information.

Lines 17-20:
: Secrets, never commit these.

Lines 22-24:
: Environment variable files containing tokens/passwords.

Lines 26-28:
: Any file named secrets.

Lines 30-35:
: SSH private keys, just in case.

Line 37:
: VS Code, individual user settings are not committed.

Lines 40-42:
: VIM swap files.

Lines 44-47:
: Logs and temporary files.

Lines 49-52:
: Test and cover reports.

Lines 54-55:
: AWX / Tower export files. These can contain credentials if exported carelessly.
{{< /line-explain >}}

{{< lab-callout type="warning" >}}
The `.gitignore` file only prevents untracked files from being added to Git. If I accidentally commit a secret file before adding it to `.gitignore`, the secret is now in the Git history. Even if I delete the file afterward. Git history is permanent. The correct remediation is `git filter-repo` to rewrite history (which destroys all commit SHAs and requires every team member to re-clone), plus immediately rotating the compromised credential. The lesson: set up `.gitignore` before the first commit, every single time.
{{< /lab-callout >}}

---

{{< subtle-label >}}host_vars/ and group_vars/{{< /subtle-label >}}

I left these two lines commented out intentionally:

{{< codeblock lang="" copy="false" >}}
# host_vars/
# group_vars/
{{< /codeblock >}}

If I uncomment them, Git ignores the entire `host_vars/` and `group_vars/` directories, which means my variable files never get committed. That's too aggressive. Most content in these directories (VLAN lists, interface names, routing configs) is not sensitive and should be in version control.

The correct approach is to use Ansible Vault to encrypt only the sensitive values within those files. I commit the encrypted vault files. Git stores the ciphertext, which is safe. The vault password itself goes in `.vault_pass`, which is in `.gitignore`.

---

## The Basic Git Workflow

With the `.gitignore` in place, I'm ready to start tracking files. The core Git workflow I use daily is: **modify → stage → commit → push**.

---

{{< subtle-label >}}Three States of a File in Git{{< /subtle-label >}}

{{< codeblock lang="" copy="false" >}}
Working Directory  →  Staging Area (Index)  →  Repository (.git/)
   (modified)           (git add)               (git commit)
{{< /codeblock >}}

- **Working Directory** - files on my filesystem as I edit them
- **Staging Area** - files I've marked as ready to be included in the next commit
- **Repository** - committed snapshots permanently stored in Git history

Understanding the staging area is the key to Git. It lets me commit only specific changes even if I've modified multiple files. I stage exactly what I want in each commit.

---

{{< subtle-label >}}First Commit{{< /subtle-label >}}

{{< codeblock lang="Bash" syntax="bash" >}}
cd ~/projects/ansible-network
{{< /codeblock >}}

1. Check current state

{{< codeblock lang="Bash" syntax="bash" >}}
git status
{{< /codeblock >}}

2. Stage the .gitignore file first

{{< codeblock lang="Bash" syntax="bash" >}}
git add .gitignore
git status
{{< /codeblock >}}

3. Commit it

{{< codeblock lang="Bash" syntax="bash" >}}
git commit -m "Initial commit: add .gitignore for Ansible project"
{{< /codeblock >}}

4. Stage the collections requirements file

{{< codeblock lang="Bash" syntax="bash" >}}
git add collections/requirements.yml
git commit -m "Add Ansible collections requirements file"
{{< /codeblock >}}

5. Stage the entire project structure at once

{{< codeblock lang="Bash" syntax="bash" >}}
git add .
git status
git commit -m "Add initial project directory structure"
{{< /codeblock >}}


- `git add .` - stages all untracked and modified files in the current directory and below. The `.` means "here and everything beneath."
- `git add .gitignore` - stages only that specific file. Precise staging like this produces cleaner commits.

---

{{< subtle-label >}}Viewing What Changed Before Staging{{< /subtle-label >}}

See unstaged changes (what I've modified but not yet staged)

{{< codeblock lang="Bash" syntax="bash" >}}
git diff
{{< /codeblock >}}

See staged changes (what will go into the next commit)

{{< codeblock lang="Bash" syntax="bash" >}}
git diff --staged
{{< /codeblock >}}

See changes in a specific file

{{< codeblock lang="Bash" syntax="bash" >}}
git diff playbooks/site.yml
{{< /codeblock >}}

`git diff` output reads like this:

{{< codeblock lang="Bash" syntax="diff" >}}
diff --git a/playbooks/site.yml b/playbooks/site.yml
index a1b2c3d..d4e5f6g 100644
--- a/playbooks/site.yml
+++ b/playbooks/site.yml
@@ -5,6 +5,8 @@
   hosts: cisco_ios
   gather_facts: false
+  connection: network_cli
+  become: false
   tasks:
{{< /codeblock >}}

- Lines starting with `+` are additions (shown in green in VS Code)
- Lines starting with `-` are deletions (shown in red)
- The `@@` line shows which line numbers are affected

---

{{< subtle-label >}}Unstaging a File{{< /subtle-label >}}

If I staged something by mistake:

{{< codeblock lang="Bash" syntax="bash" >}}
git restore --staged filename.yml
{{< /codeblock >}}

The file goes back to "modified but unstaged". The change is still there, it just won't be in the next commit.

---

{{< subtle-label >}}Discarding Changes Entirely{{< /subtle-label >}}

Discard all changes to a file since the last commit (CANNOT BE UNDONE):

{{< codeblock lang="Bash" syntax="bash" >}}
git restore filename.yml
{{< /codeblock >}}

{{< lab-callout type="warning" >}}
`git restore filename.yml` permanently discards uncommitted changes to that file. There is no undo. This is one of the few Git operations that loses work irreversibly. I always run `git diff filename.yml` first to confirm what I'm about to lose before running restore.
{{< /lab-callout >}}

---

{{< subtle-label >}}Writing Meaningful Commit Messages{{< /subtle-label >}}

A commit message is a letter to my future self and my teammates explaining why a change was made. The code shows what changed and the commit message explains why.

---

{{< subtle-label >}}Standard Format{{< /subtle-label >}}

{{< codeblock lang="" copy="false" >}}
<type>(<scope>): <short summary — 50 chars or less>

<body — optional, wrap at 72 chars>
Explain the motivation for the change. What problem does this solve?
What was the previous behavior, and why was it wrong?

<footer — optional>
Refs: CHG0012345
Closes: #42
{{< /codeblock >}}

---

### Commit Types

| Type | When to Use |
|---|---|
| `feat` | A new playbook, role, or feature |
| `fix` | A bug fix in an existing playbook |
| `refactor` | Restructuring code without changing behavior |
| `docs` | README, comments, or documentation changes |
| `chore` | Maintenance tasks (updating requirements, .gitignore) |
| `test` | Adding or updating test playbooks |
| `revert` | Reverting a previous commit |

---

{{< subtle-label >}}Good vs Bad Commit Messages{{< /subtle-label >}}


{{< codeblock file="Bad" syntax="bash" >}}
git commit -m "fix"
git commit -m "update playbook"
git commit -m "changes"
git commit -m "WIP"
{{< /codeblock >}}

{{< codeblock file="Good" syntax="bash" >}}
git commit -m "fix(ios): correct interface description task to use ios_config not raw"
git commit -m "feat(nxos): add VLAN provisioning playbook for datacenter fabric"
git commit -m "chore: update ansible from 9.7.0 to 9.8.0, pin in requirements.txt"
git commit -m "fix(bgp): add missing neighbor activate under address-family for R3"
{{< /codeblock >}}

---

{{< subtle-label >}}Commit Message with a Body{{< /subtle-label >}}

{{< codeblock lang="Bash" syntax="bash" >}}
git commit
{{< /codeblock >}}


This opens nano for a multi-line message

{{< codeblock file="In the editor" syntax="ini" >}}
fix(ospf): increase dead interval to prevent flapping on WAN links

The OSPF dead interval was set to the default 40 seconds. WAN links
between HQ and Branch sites experience periodic latency spikes that
cause hello packets to be dropped, triggering OSPF neighbor drops
and route reconvergence.

Increased dead interval to 120 seconds and hello interval to 30
seconds on all WAN-facing interfaces. Verified no change to LAN
interfaces (still using defaults).

Tested against: R1, R2, R3 in staging environment.
Change window: CHG0019823
Refs: #87
{{< /codeblock >}}

---

## Connecting to GitHub and Pushing

#### Creating the Remote Repository on GitHub

1. Log in to **GitHub.com**
2. Click the **+** in the top right → **New repository**
3. Configure:
   - **Repository name:** `ansible-network`
   - **Description:** `Network automation playbooks and roles for Cisco IOS/NX-OS, Juniper, and Palo Alto`
   - **Visibility:** Private (always private for infrastructure code)
   - **Do NOT initialize** with README, .gitignore, or license (I already have these locally)
4. Click **Create repository**

GitHub shows the "Quick setup" page with instructions. I'll use the SSH URL.

---

#### Connecting the Local Repository to GitHub

{{< codeblock lang="Bash" syntax="bash" >}}
cd ~/projects/ansible-network
{{< /codeblock >}}

Add GitHub as the remote repository (named "origin" by convention)

{{< codeblock lang="Bash" syntax="bash" >}}
git remote add origin git@github.com:myusername/ansible-network.git
{{< /codeblock >}}

Verify the remote was added

{{< codeblock lang="Bash" syntax="bash" >}}
git remote -v
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
origin  git@github.com:myusername/ansible-network.git (fetch)
origin  git@github.com:myusername/ansible-network.git (push)
{{< /codeblock >}}

`origin` is the conventional name for the primary remote. I can name it anything, but `origin` is what every tool expects.

---

{{< subtle-label >}}Pushing for the First Time{{< /subtle-label >}}

Push the main branch to GitHub and set it as the upstream tracking branch

{{< codeblock lang="Bash" syntax="bash" >}}
git push -u origin main
{{< /codeblock >}}

`-u origin main` - sets the upstream tracking relationship. After this first push, I can just type `git push` and Git knows where to push to.

{{< codeblock lang="Expected Output" copy="false" >}}
Enumerating objects: 12, done.
Counting objects: 100% (12/12), done.
Writing objects: 100% (12/12), 1.23 KiB | 1.23 MiB/s, done.
To git@github.com:myusername/ansible-network.git
 * [new branch]      main -> main
Branch 'main' set up to track remote branch 'main' from 'origin'.
{{< /codeblock >}}

I can now go to `github.com/myusername/ansible-network` and see my files online.

---

## Git Workflow

These are the commands I run every day.

1. Start the session (activate the virtualenv and navigate to the lab)

{{< codeblock lang="Bash" syntax="bash" >}}
source ~/venvs/ansible-network/bin/activate
cd ~/projects/ansible-network
{{< /codeblock >}}

2. Pull the latest changes from GitHub before starting work

{{< codeblock lang="Bash" syntax="bash" >}}
git pull
{{< /codeblock >}}

3. Do my work (edit playbooks, add roles, update inventory)

4. Check what changed

{{< codeblock lang="Bash" syntax="bash" >}}
git status
git diff
{{< /codeblock >}}

5. Stage specific files

{{< codeblock lang="Bash" syntax="bash" >}}
git add playbooks/deploy_vlans.yml
{{< /codeblock >}}

6. Commit with a meaningful message

{{< codeblock lang="Bash" syntax="bash" >}}
git commit -m "feat(vlans): add VLAN deployment playbook for IOS access switches"
{{< /codeblock >}}

7. Push to GitHub

{{< codeblock lang="Bash" syntax="bash" >}}
git push
{{< /codeblock >}}

`git pull` is actually two operations in one: `git fetch` (download changes from GitHub) followed by `git merge` (apply them to my local branch). For solo work, `git pull` is fine. On a team, some engineers prefer `git pull --rebase` which replays my local commits on top of the fetched commits, keeping a cleaner linear history.

---

## Branching Strategy

On a team with a formal review process, nobody pushes directly to `main`. Changes go through branches and Pull Requests. Here's the lightweight strategy I use, tailored specifically for network automation work.

---

{{< subtle-label >}}Branch Structure{{< /subtle-label >}}

{{< codeblock lang="" copy="false" >}}
main          # Production-ready code only. Protected branch.
│             # All changes enter via Pull Request and peer review
├── feature/add-bgp-role
├── feature/nxos-vlan-playbook
├── fix/ospf-dead-interval-wan
├── hotfix/acl-blocking-monitoring
└── chore/update-ansible-9.8
{{< /codeblock >}}

{{< subtle-label >}}Branch Types{{< /subtle-label >}}

| Branch Type | Naming Convention | Purpose |
|---|---|---|
| Feature | `feature/short-description` | New playbooks, roles, or capabilities |
| Fix | `fix/short-description` | Bug fixes in existing playbooks |
| Hotfix | `hotfix/short-description` | Urgent fixes that need to bypass normal review timelines |
| Chore | `chore/short-description` | Dependency updates, documentation, `.gitignore` changes |
| Test | `test/short-description` | Experimental changes, not intended for merge |

---

{{< subtle-label >}}Creating and Working on a Branch{{< /subtle-label >}}

I always start from an up-to-date main

{{< codeblock lang="Bash" syntax="bash" >}}
git checkout main
git pull
{{< /codeblock >}}

Create a new branch and switch to it in one command

{{< codeblock lang="Bash" syntax="bash" >}}
git checkout -b feature/add-ios-vlan-playbook
{{< /codeblock >}}

Verify which branch I'm on

{{< codeblock lang="Bash" syntax="bash" >}}
git branch
# * feature/add-ios-vlan-playbook
#   main
{{< /codeblock >}}

- `git checkout -b` - creates the branch AND switches to it. Without `-b`, I'd switch to an existing branch.
- The `*` in `git branch` output marks the current branch.

I then do my work on this branch

---

Stage and commit as normal. Commits go to the feature branch, not main

{{< codeblock lang="Bash" syntax="bash" >}}
git add playbooks/deploy_vlans.yml
git commit -m "feat(vlans): add IOS VLAN deployment playbook with trunk/access support"
{{< /codeblock >}}

Push the feature branch to GitHub

{{< codeblock lang="Bash" syntax="bash" >}}
git push -u origin feature/add-ios-vlan-playbook
{{< /codeblock >}}

---

{{< subtle-label >}}Switching Between Branches{{< /subtle-label >}}

Switch back to main (e.g., to pull updates or start a different task)

{{< codeblock lang="Bash" syntax="bash" >}}
git checkout main
{{< /codeblock >}}

Switch back to my feature branch

{{< codeblock lang="Bash" syntax="bash" >}}
git checkout feature/add-ios-vlan-playbook
{{< /codeblock >}}

List all branches (local and remote)

{{< codeblock lang="Bash" syntax="bash" >}}
git branch -a
{{< /codeblock >}}


Before switching branches, I always commit or stash my work-in-progress. Git will refuse to switch branches if I have uncommitted changes that conflict with the target branch. If I'm mid-task and need to switch:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
git stash
git checkout main
# ... do what I need to do on main ...
git checkout feature/add-ios-vlan-playbook
git stash pop
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Temporarily shelve uncommitted changes

Line 5:
: Restore my shelved changes
{{< /line-explain >}}



---

{{< subtle-label >}}Cleaning Up{{< /subtle-label >}}

Switch back to main and pull the merged changes

{{< codeblock lang="Bash" syntax="bash" >}}
git checkout main
git pull
{{< /codeblock >}}

Delete the feature branch locally (it's been merged, no longer needed)

{{< codeblock lang="Bash" syntax="bash" >}}
git branch -d feature/add-ios-vlan-playbook
{{< /codeblock >}}

Delete the remote branch on GitHub

{{< codeblock lang="Bash" syntax="bash" >}}
git push origin --delete feature/add-ios-vlan-playbook
{{< /codeblock >}}

---

## Peer Review Process

A Pull Request (PR) is a formal request to merge a branch into `main`. On a team with a formal review process, no code reaches `main` without at least one approval.

{{< subtle-label >}}Creating a Pull Request on GitHub{{< /subtle-label >}}

After pushing a feature branch:

1. Go to the repository on **GitHub.com**
2. GitHub usually shows a yellow banner: "feature/add-ios-vlan-playbook had recent pushes" → click **Compare & pull request**
3. Fill in the PR template:

**PR Title:**

{{< codeblock lang="" copy="false" >}}
feat(vlans): Add IOS VLAN deployment playbook with trunk/access support
{{< /codeblock >}}


**PR Description:**

{{< codeblock lang="Markdown" syntax="markdown" >}}
## Summary
Adds a new playbook `playbooks/deploy_vlans.yml` for deploying VLAN
configurations to Cisco IOS access layer switches.

## Changes
- New playbook: `playbooks/deploy_vlans.yml`
- New vars file: `inventory/group_vars/cisco_ios_access.yml`
- Updated: `collections/requirements.yml` (added cisco.ios 8.0.1)

## Testing
- Tested against 3x CSR1000v nodes in Containerlab
- Ran with `--check` first, then applied
- Verified VLAN database and trunk configurations post-apply

## How to Test
`ansible-playbook playbooks/deploy_vlans.yml --limit lab_switches --check`
`ansible-playbook playbooks/deploy_vlans.yml --limit lab_switches`
{{< /codeblock >}}

---

{{< subtle-label >}}Checklist{{< /subtle-label >}}

- [x] ansible-lint passes with no violations
- [x] yamllint passes with no violations
- [x] No secrets or credentials in this PR
- [x] requirements.txt updated if new Python packages added
- [x] collections/requirements.yml updated if new collections added

4. Assign a **Reviewer** at least one other engineer
5. Click **Create pull request**

{{< lab-callout type="info" >}}
I add a PR template to the repository so every PR automatically gets the checklist structure. I create the file `.github/pull_request_template.md` in the repository root and GitHub will use it for every new PR automatically. This standardizes what every reviewer expects to see and makes the checklist a habit rather than an afterthought.
{{< /lab-callout >}}

---

{{< subtle-label >}}What the Reviewer Does{{< /subtle-label >}}

The reviewer:
- Reads through every changed file in the **Files changed** tab
- Looks for logic errors, missing error handling, hardcoded values that should be variables
- Verifies no secrets are present
- Checks that the playbook follows the project's naming and structure conventions
- Leaves inline comments on specific lines if something needs changing
- Either Approves, Requests changes, or Comments without a verdict

---

{{< subtle-label >}}Responding to Review Feedback{{< /subtle-label >}}

Make the requested changed on my feature branch

{{< codeblock lang="Bash" syntax="bash" >}}
git add playbooks/deploy_vlans.yml
git commit -m "fix: address review feedback — parameterize VLAN range, add error handling"
git push
{{< /codeblock >}}

The PR on GitHub automatically updates with the new commit. The reviewer can see the changes and re-review. Once approved, the PR is merged.

---

### Branch Protection Rules

For a team with formal review, I configure branch protection on `main` in GitHub:

1. Go to repository → **Settings** → **Branches**
2. Click **Add branch protection rule**
3. **Branch name pattern:** `main`
4. Enable:
   - [x] **Require a pull request before merging**
   - [x] **Require approvals** set to `1` minimum (or `2` for higher-risk repos)
   - [x] **Dismiss stale pull request approvals when new commits are pushed**
   - [x] **Require status checks to pass before merging** (for CI/CD in Part 32)
   - [x] **Restrict who can push to matching branches** (only senior engineers or automation service accounts)
5. Click **Create**

---

## git log and git diff

{{< subtle-label >}}Viewing Commit History{{< /subtle-label >}}

Basic log:

```bash
git log
```

Compact one-line format (my most-used):

```bash
git log --oneline
```

Compact with branch graph:

```bash
git log --oneline --graph --all
```

Last 5 commits:

```bash
git log -5 --oneline
```

Commits by a specific author:

```bash
git log --author="First Last" --oneline
```

Commits that touched a specific file:

```bash
git log --oneline -- playbooks/deploy_vlans.yml
```

Commits in a date range:

```bash
git log --oneline --after="2024-01-01" --before="2024-12-31"
```

Search commit messages for a keyword:

```bash
git log --grep="bgp" --oneline
```

Sample `git log --oneline --graph --all` output:
```
* a3f2b1c (HEAD -> main, origin/main) fix(bgp): add missing neighbor activate for R3
* 9d4e5f2 feat(vlans): add IOS VLAN deployment playbook
* 7c8b3a1 chore: update ansible from 9.7.0 to 9.8.0
* 4f1d9e8 feat(ospf): add multi-area OSPF role for IOS
* 2a3c7b0 Initial commit: add .gitignore for Ansible project
```

---

{{< subtle-label >}}Inspecting a Specific Commit{{< /subtle-label >}}

Show the full diff of a specific commit:

```bash
git show a3f2b1c
```

Show just the files that changed in a commit:

```bash
git show a3f2b1c --stat
```

Show what changed in a specific file in a specific commit:

```bash
git show a3f2b1c -- playbooks/bgp.yml
```

---

{{< subtle-label >}}Comparing States{{< /subtle-label >}}

What changed between two commits:

```bash
git diff 4f1d9e8 a3f2b1c
```

What changed between a commit and the current working state:

```bash
git diff a3f2b1c
```

What changed between two branches:

```bash
git diff main feature/add-ios-vlan-playbook
```

What changed in a specific file between two branches:

```bash
git diff main feature/add-ios-vlan-playbook -- playbooks/deploy_vlans.yml
```

{{< subtle-label >}}Reverting a Bad Commit{{< /subtle-label >}}

If a commit that was already pushed to `main` turns out to be wrong:

```bash
git revert a3f2b1c
git push
```

>[!Caution]
> Never use `git reset --hard` or `git push --force` on the `main` branch on a shared repository. These commands rewrite history, which invalidates every team member's local copy of the repository and can cause data loss. `git revert` is always the safe way to undo a change that's already been pushed. Reserve `git reset` for cleaning up commits that have NOT yet been pushed.

---

## Security Best Practices

{{< subtle-label >}}What Never Goes Into Git{{< /subtle-label >}}

```
- Passwords (device passwords, API tokens, RADIUS secrets)
- SSH private keys
- Ansible Vault passwords (.vault_pass)
- .env files with credentials
- AWS/cloud credentials
- Private IP addressing schemes of production networks (debatable, but cautious teams exclude this)
- Anything that would give an attacker a foothold if the repo were made public
```

---

{{< subtle-label >}}Scanning for Accidentlly Committed Secrets{{< /subtle-label >}}

Before pushing, I can scan for secrets using `git-secrets` or `trufflehog`:

Install trufflehog (a secrets scanner)

```bash
pip install trufflehog
```

Scan the entire repository history for secrets

```bash
trufflehog git file://. --only-verified
```

{{< subtle-label >}}Setting Up a Pre-commit Hook to Block Secret Commits{{< /subtle-label >}}

A pre-commit hook runs automatically before every `git commit` and can block the commit if it finds problems:

Install pre-commit framework

```bash
pip install pre-commit
```

Create .pre-commit-config.yaml in the project root

```bash {linenos=table,hl_lines=[6,7,8,9,14,20]}
cat > ~/projects/ansible-network/.pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: check-yaml
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-merge-conflict

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets 
        args: ['--baseline', '.secrets.baseline']

  - repo: https://github.com/ansible/ansible-lint
    rev: v24.9.2
    hooks:
      - id: ansible-lint
EOF
```

- `id: check-yaml` - Validates YAML syntax
- `id:end-of-file-fixer` - Ensures files end with newline
- `id:trailing-whitespace` - Removes trailing whitespace
- `id: check-merge-conflict` - Blocks commits with merge conflic markers
- `id: detect-secrets` - Scans for hardcoded secrets
- `id: ansible-link` - Runs ansible-lint before every commit

Install the hooks (runs once — creates hooks in .git/hooks/)

```bash
pre-commit install
```

- `pre-commit install` - installs the hooks into `.git/hooks/pre-commit`. Now every `git commit` automatically runs these checks first.
- If any check fails, the commit is blocked until I fix the issue.

Test it manually

```bash
pre-commit run --all-files
```

>[!Tip]
> The `.pre-commit-config.yaml` file should be committed to the repository. This means every engineer who clones the repo and runs `pre-commit install` gets the same hooks. Combined with branch protection rules on GitHub, this creates two layers of defense: hooks block bad commits locally, and GitHub blocks merges that haven't passed review. Neither layer alone is sufficient.

---

### If a Secret Was Already Committed

This is the procedure if I accidentally committed a credential:

1. **Rotate the credential immediately** - assume it's compromised. Change the password, revoke the API token, generate a new SSH key. Do this first, before anything else.
2. **Remove it from history** using `git filter-repo` (not `git filter-branch` which is deprecated):
   ```bash
   pip install git-filter-repo
   git filter-repo --path secrets.yml --invert-paths
   git push --force-with-lease origin main
   ```
3. **Notify the team** - everyone must re-clone the repository because the history has changed.
4. **Add the file to `.gitignore`** immediately.
5. **Conduct a post-incident review** - how did this happen and what process change prevents it next time?

---

Every change I make from this point forward goes through Git. Playbooks, inventory files, variable files, roles, templates. All of it is version-controlled, reviewed, and traceable.


