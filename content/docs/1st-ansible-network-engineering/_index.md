---
draft: false
title: 'Learning Ansible for Network Automation'
sidebar:
  exclude: false
weight: 1
---

---

This project is mostly me learning, a lot of parts in this project are more for me to be able to reference if needed. I also learn best by teaching, and repetition. Documenting my projects kills 2 birds with 1 stone.

This project treats the network as code. Device configurations, VLAN assignments, BGP policies, monitoring targets, and topology definitions are all version controlled, validated through a PR pipeline, and deployed automatically on merge.

This project is documented across 46 parts, starting from a bare control node and working through the full automation platform.

---

{{< accent-label >}}Network topology{{< /accent-label >}}

All devices run as Containerlab nodes using vrnetlab on a dedicated Proxmox VM. The topology file is version controller, validated by a 3 stage pipeline, and deployed via a diff-aware Gitea Actions job that adds or removes only what changed.

![topology](2nd-topology.png)

----

{{< accent-label >}}Infrastructure Stack{{< /accent-label >}}

Each service runs as a Docker container or its own VM. Everything is self-hosted on Proxmox.

{{< table >}}
Service | Purpose | Integration
**NetBox** | Source of truth | Dynamic Ansible inventory, Zabbix and Prometheus auto-registration, Graylog IP enrichment
**Gitea** | Self hosted Git with branch protection | GPG-signed commits enforced on main, Oxidized cofig backend, Gitea Actions runner
**AWX** | Automation controller (RBAC, job history, approval gates) | Webhook triggered by Graylog alerts, Netbox change events, and Gitea merges
**Zabbix** | Network monitoring via SNMP polling | Hosts auto-registered from Netbox; interface, BGP, and CPU/memory triggers
**Prometheus + Grafana** | Metrics scraping and dashboards | SNMP Exporter for devices, Node Exporter for VMs
**Graylog + OpenSearch** | Structured syslog (parsing, routing, alerting) | Pipeline rules parse IOS/NX-OS/PAN-OS formates; NetBox lookup table enriches source IPs
**Oxidized** | Config backup with Git diff history | Commits to Gitea on schedule and immediately on Graylog config change detection
**Batfish** | Pre-change network model analysis | Ingests Oxidized configs; interactive reachability queries before opening a PR
**Ansible EDA** | Event driven automation | Real-time event processing; wired into Graylog alart stream
**FreeRADIUS** | 802.1X wired authentication | RADIUS server for campus access switch authentication
**tac_plus** | TACACS+ for AAA nd command authorization | Privilege levels and command sets per device role
**Netdisco** | Layer 2/3 discovery | Reality check against NetBox intent; drift dtection
{{< /table >}}

---

{{< accent-label >}}Key Capabilities{{< /accent-label >}}

{{< feature-grid >}}
  {{< feature-card title="GitOps Pipeline" mono="true" >}}
  Pull requests trigger yamllint and ansible-lint gates, merged is blocked if either fails. Marging to main auto-deploys via a self-hosted Gitea Actions runner. Scoped pipelines mean a campus VLAN change only triggers the campus workflow, not a full fabric run.
  {{< /feature-card >}}

  {{< feature-card title="Shelf-healing workflows" mono="true" >}}
  Graylog detects config change syslog events, fires a webhook to AWX. An automated three-stage workflow backs up the device, collects BGP summary and interface states, and posts a full diagnosis report to Slack.
  {{< /feature-card >}}

  {{< feature-card title="Pre-change analysis" mono="true" >}}
  Before opening a PR, an engineer runs a Python script that loads the current Oxidized configs into Batfish and traces a traffic flow end to end. The output shows the full hop-by-hop forwarding path and which ACL line would block the traffic.
  {{< /feature-card >}}

  {{< feature-card title="NetBox as source of truth" mono="true" >}}
  No static inventory files. NetBox drives dynamic Ansible inventory, Zabbix host registration, Prometheus scrape targets, and Graylog IP enrichment. Adding a device to NetBox propagates to all monitoring tools automatically via webhook-triggered AWX jobs.
  {{< /feature-card >}}

  {{< feature-card title="Topology as code" mono="true" mono="true" >}}
  The Containerlab topology file is version-controlled alongside playbooks. PRs trigger a three-stage validation: yamllint, a structural Python validator checking every node and link, then `containerlab graph`. Merges deploy with a diff-aware job that adds or removes only what changed
  {{< /feature-card >}}

  {{< feature-card title="Pipeline hardening" mono="true" mono="true" >}}
  GPG-signed commits enforced on the main branch. detect-secrets pre-commit hooks block credentials before they reach Gitea. Ansible collections pinned with exact versions and SHA256 checksums, served from a private Gitea package registry mirror.
  {{< /feature-card >}}

  {{< feature-card title="GitOps Change Pipeline" mono="true" wide="true" >}}
  Feature branch ➦ Pull request ➦ yamllint ➦ ansible-lint ➦ Batfish query ➦ Merge to main ➦ Gitea Actions deploy ➦ Oxidized backup
  {{< /feature-card >}}

  {{< feature-card title="Self-healing loop" mono="true" wide="true" >}}
  Config change on device ➦ Graylog syslog event ➦ Alert fires ➦ AWX webhook ➦ Stage 1: Backup ➦ Stage 2: Diagnose ➦ Stage 3: Slack report
  {{< /feature-card >}}

  {{< feature-card title="Topology Change Pipeline" mono="true" wide="true" >}}
  Edit topology-lab.yml ➦ Pull request ➦ yamllint ➦ Python validator ➦ containerlab graph ➦ Merge ➦ Diff-aware deploy ➦ Node reachability verify
  {{< /feature-card >}}
{{< /feature-grid >}}

---

{{< accent-label >}}Security Practices{{< /accent-label >}}

{{< pill-grid >}}
  {{< pill-item >}}Ansible Vault with multiple vault IDs per trust boundary{{< /pill-item >}}
  {{< pill-item >}}Pre-commit hooks blocking secrets, unencrypted vaults{{< /pill-item >}}
  {{< pill-item >}}Branch protection with required PR approval{{< /pill-item >}}
  {{< pill-item >}}Passphrase-protected Ed25519 SSH keys with `ssh-agent`{{< /pill-item >}}
  {{< pill-item >}}TACACS+ with FreeIPA LDAP backend{{< /pill-item >}}
  {{< pill-item >}}Internal CA (`step-ca`) with ACME certificate automation{{< /pill-item >}}
  {{< pill-item >}}Signed commits, secrets scanning, SBOM generation{{< /pill-item >}}
  {{< pill-item >}}SSH hardening, control plane policing, management ACLs{{< /pill-item >}}
{{< /pill-grid >}}

---

{{< accent-label >}}Screenshots{{< /accent-label >}}

{{< showcase-grid >}}
  {{< showcase-card
      title="Grafana: Fabric health"
      image="grafana.jpg"
      desc="Unified dashboard with device availability, interface utilization, and BGP session states."
  >}}

  {{< showcase-card
      title="NetBox: Device inventory"
      image="netbox.jpg"
      desc="All 8 devices with sites, roles, platforms, and primary IPs populated."
  >}}

  {{< showcase-card
      title="AWX: Workflow execution"
      image="awx.jpg"
      desc="Multi-step workflow with validation, deployment, and approval gates."
  >}}

  {{< showcase-card
      title="Gitea: PR with CI checks"
      image="gitea.jpg"
      desc="Pull request showing lint, syntax, and pre-change analysis passing."
  >}}
{{< /showcase-grid >}}

---

{{< accent-label >}}Build Sequence{{< /accent-label >}}

{{< phase-list >}}
  {{< phase-row num="1" title="Foundation" parts="Parts 01–06" color="#34c759" >}}
  Control node, Gitea, Ansible Vault, Containerlab with first devices, playbook fundamentals, Jinja2 deep dive, reusable roles.
  {{< /phase-row >}}

  {{< phase-row num="2" title="The Network Fabric" parts="Parts 07–15" color="#2997ff" >}}
  NetBox as source of truth, IPAM plan, multi-vendor automation (IOS-XE, NX-OS, PAN-OS), BGP fabric, VM provisioning, monitoring, config backup.
  {{< /phase-row >}}

  {{< phase-row num="3" title="Network Services" parts="Parts 16–25" color="#ff9f0a" >}}
  BGP/OSPF production patterns, FreeIPA, TACACS+, security hardening, internal PKI, DNS/NTP, centralized logging, traffic flow visibility, unified observability.
  {{< /phase-row >}}

  {{< phase-row num="4" title="Automation Platform" parts="Parts 26–34" color="#bf5af2" >}}
  AWX controller, GitOps pipeline, Batfish pre-change analysis, pyATS live validation, Molecule role testing, CI/CD hardening, drift detection, NetBox reconciliation.
  {{< /phase-row >}}

  {{< phase-row num="5" title="Advanced Automation" parts="Parts 35–38" color="#ff375f" >}}
  AWX workflows with approval gates, Event-Driven Ansible, closed-loop self-healing, full-stack rebuild from scratch.
  {{< /phase-row >}}
{{< /phase-list >}}

---

{{< accent-label >}}Technologies{{< /accent-label >}}

**AUTOMATION**

{{< badge "Ansible" >}}
{{< badge content="Ansible Vault" color="red" >}}
{{< badge content="AWX" color="yellow" >}}
{{< badge content="Ansible EDA" color="blue" >}}
{{< badge content="Jinja2" color="green" >}}
{{< badge content="Molecole" color="orange" >}}

**NETWORK PLATFORMS**

{{< badge "Cisco IOS-XE" >}}
{{< badge content="Cisco NX-OS" color="red" >}}
{{< badge content="Palo Alto PAN-OS" color="yellow" >}}
{{< badge content="BGP" color="blue" >}}
{{< badge content="OSPF" color="green" >}}
{{< badge content="TACACS+" color="orange" >}}

**INFRASTRUCTURE**

{{< badge "NetBox" >}}
{{< badge content="Containerlab" color="red" >}}
{{< badge content="Proxmox VE" color="yellow" >}}
{{< badge content="Docker" color="blue" >}}
{{< badge content="FreeIPA" color="green" >}}
{{< badge content="step-ca" color="orange" >}}

**OBSERVABILITY**

{{< badge "Zabbix" >}}
{{< badge content="Prometheus" color="red" >}}
{{< badge content="Grafana" color="yellow" >}}
{{< badge content="Graylog" color="blue" >}}
{{< badge content="OpenSearch" color="green" >}}
{{< badge content="ntopng" color="orange" >}}
{{< badge content="Oxidized" color="brown" >}}

**DEVOPS**

{{< badge "Git" >}}
{{< badge content="Gitea" color="red" >}}
{{< badge content="GitOps" color="yellow" >}}
{{< badge content="CI/CD" color="blue" >}}
{{< badge content="Batfish" color="green" >}}
{{< badge content="pyATS" color="orange" >}}
{{< badge content="Netdisco" color="brown" >}}

**LANGUAGES**

{{< badge "Python" >}}
{{< badge content="YAML" color="red" >}}
{{< badge content="Jinja2" color="yellow" >}}
{{< badge content="Bash" color="blue" >}}
{{< badge content="REST APIs" color="green" >}}