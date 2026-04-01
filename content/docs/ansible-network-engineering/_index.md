---
draft: false
title: 'Ansible for Network Engineering'
sidebar:
  exclude: false
---

---

{{< phase num="1" title="Foundation" >}}
Build the control node, version control, lab topology, core Ansible skills and everything else needed before scaling.
{{< /phase >}}

{{< part-card num="01" title="Project Structure" href="/docs/ansible-network-engineering/part_01/" >}}
Proxmox VM, Python venv, Ansible install, directory layout, local Git init.
{{< /part-card >}}

{{< part-card num="" title="Workstation Setup" href="/docs/ansible-network-engineering/part_00/" >}}
VSCode with Remote-SSH, extensions for Ansible and Jinja2, SSH key setup for Windows and other tools for this project.
{{< /part-card >}}

{{< part-card num="02" title="Gitea" href="/docs/ansible-network-engineering/part_02/" >}}
Deploy Gitea as a container. Create the project repo, push the local Git history. Branch protection rules.
{{< /part-card >}}

{{< part-card num="03" title="Ansible Vault" href="/docs/ansible-network-engineering/part_03/" >}}
Vault setup, encrypted vars file, `.vault/` directory, vault-id pattern, pre-commit hook blocking plaintext secrets.
{{< /part-card >}}

{{< part-card num="04" title="Containerlab" href="/docs/ansible-network-engineering/part_04/" >}}
Containerlab install, vrnetlab image import. Deploy `wan-r1` and `wan-r2` using Cat9kv. Verify console access and management connectivity.
{{< /part-card >}}

{{< part-card num="05" title="Playbooks" href="/docs/ansible-network-engineering/part_05/" >}}
Static inventory. `network_cli` connection, `ios_command`, `ios_config`, `gather_facts`, save config.
{{< /part-card >}}

{{< part-card num="06" title="Roles & Templates" href="/docs/ansible-network-engineering/part_06/" >}}
Role directory structure, Jinja2 templates, defaults vs vars. Build reusable roles.
{{< /part-card >}}