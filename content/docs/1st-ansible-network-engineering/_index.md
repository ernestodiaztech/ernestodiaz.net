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

---

{{< topology1 src="diagrams/topology1.svg" >}}

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
  {{< pill-item >}}GPG-signed commits enforced on protected main branch{{< /pill-item >}}
  {{< pill-item >}}detect-secrets pre-commit hook; credentials blocked before reaching Git{{< /pill-item >}}
  {{< pill-item >}}Ansible collections pinned with SHA256 checksums and private mirror{{< /pill-item >}}
  {{< pill-item >}}Ansible Vault; separate vault IDs per trust boundary{{< /pill-item >}}
  {{< pill-item >}}Branch protection; merge requires lint pipeline to pass{{< /pill-item >}}
  {{< pill-item >}}TACACS+ with privilege levels and command authorisation per role{{< /pill-item >}}
  {{< pill-item >}}SSH hardening, management ACLs, control plane policing on all devices{{< /pill-item >}}
  {{< pill-item >}}Secrets remediation process documented; git-filter-repo purge procedure{{< /pill-item >}}
{{< /pill-grid >}}

---

{{< accent-label >}}Screenshots{{< /accent-label >}}

{{< showcase-grid >}}
  {{< showcase-card
      title="Grafana: Lab Operations Center"
      image="grafana.jpg"
      desc="BGP session states, interface traffic, VM health, and Loki log stream in a single dashboard."
  >}}

  {{< showcase-card
      title="Graylog: Structured Syslog"
      image="graylog.jpg"
      desc="IOS-XE config change event parsed into fields; device_name, event_type, cisco_mnemonic, enriched from NetBox."
  >}}

  {{< showcase-card
      title="Gitea: PR Pipeline Passing"
      image="awx.jpg"
      desc="Yamllint and ansible-lint checks passing on a campus VLAN change PR before merge to main."
  >}}

  {{< showcase-card
      title="AWX: Self-headling Workflow"
      image="gitea.jpg"
      desc="Three-stage remediation workflow: backup → diagnose → notify, triggered by a Graylog config-change alert."
  >}}
{{< /showcase-grid >}}

---

{{< accent-label >}}Build Sequence{{< /accent-label >}}

{{< phase-list >}}
  {{< phase-row num="1" title="Foundation" parts="Parts 01–16" color="#34c759" >}}
  Control node, Ansible fundamentals, Vault secrets management, Containerlab topology, all 7 virtual devices, base IOS-XE / NX-OS / PAN-OS automation, BGP fabric brought up end to end.
  {{< /phase-row >}}

  {{< phase-row num="2" title="Infrastructure Platform" parts="Parts 17–20" color="#2997ff" >}}
  Proxmox VM provisioning, Gitea self-hosted Git, NetBox IPAM and dynamic inventory, AWX automation controller, infrastructure hardening.
  {{< /phase-row >}}

  {{< phase-row num="3" title="Enterprise Network Services" parts="Parts 21–34" color="#ff9f0a" >}}
  Campus switching: VLANs, STP, LAG, port security, 802.1X, QoS. Routing;  OSPF, BGP policy, HSRP, WAN. Firewall policies, NAT, IPsec VPN, AAA, NTP, SNMP.
  {{< /phase-row >}}

  {{< phase-row num="4" title="Observability Stack" parts="Parts 35–41" color="#bf5af2" >}}
  Zabbix SNMP monitoring, Prometheus + Grafana + Loki, Graylog + OpenSearch structured logging, stack integration: NetBox → Zabbix sync, unified dashboard, self-healing workflows.
  {{< /phase-row >}}

  {{< phase-row num="5" title="GitOps, Hardening, and Automation" parts="Parts 42-52" color="#ff375f" >}}
  Oxidized config backup, GitOps pipeline, Batfish pre-change analysis, CI/CD hardening, topology as code, Netdisco discovery, NetBox reconciliation, AWX workflows, Ansible EDA.
  {{< /phase-row >}}
{{< /phase-list >}}

---

{{< accent-label >}}Technologies{{< /accent-label >}}

**NETWORKING**

{{< badge "Cisco IOS-XE" >}}
{{< badge content="Cisco NX-OS" color="red" >}}
{{< badge content="Palo Alto PAN-OS" color="yellow" >}}
{{< badge content="FortiGate" color="blue" >}}
{{< badge content="BGP" color="green" >}}
{{< badge content="OSPF" color="orange" >}}
{{< badge "VLANs" >}}
{{< badge content="STP / RSTP" color="red" >}}
{{< badge content="LACP" color="yellow" >}}
{{< badge content="HSRP" color="blue" >}}
{{< badge content="802.1X" color="green" >}}
{{< badge content="ACLs" color="orange" >}}
{{< badge "NAT" >}}
{{< badge content="IPsec VPN" color="red" >}}
{{< badge content="QoS" color="yellow" >}}
{{< badge content="SNMP" color="blue" >}}

**AUTOMATION**

{{< badge "Ansible" >}}
{{< badge content="AWX" color="red" >}}
{{< badge content="Ansible EDA" color="yellow" >}}
{{< badge content="Jinja2" color="blue" >}}
{{< badge content="Ansible Vault" color="green" >}}
{{< badge content="Python" color="orange" >}}
{{< badge "pybatfish" >}}
{{< badge content="Containerlab" color="red" >}}
{{< badge content="vrnetlab" color="yellow" >}}

**OBSERVABILITY**

{{< badge "Prometheus" >}}
{{< badge content="Grafana" color="red" >}}
{{< badge content="Loki" color="yellow" >}}
{{< badge content="Zabbix" color="blue" >}}
{{< badge content="Graylog" color="green" >}}
{{< badge content="OpenSearch" color="orange" >}}
{{< badge "Oxidized" >}}
{{< badge content="Batfish" color="red" >}}
{{< badge content="Netdisco" color="yellow" >}}
{{< badge content="AlertManager" color="blue" >}}

**INFRASTRUCTURE**

{{< badge "Proxmox" >}}
{{< badge content="Gitea" color="red" >}}
{{< badge content="Gitea Actions" color="yellow" >}}
{{< badge content="Docker" color="blue" >}}
{{< badge content="NetBox" color="green" >}}
{{< badge content="GPG" color="orange" >}}
{{< badge content="detect-secrets" color="brown" >}}
{{< badge "FreeRADIUS" >}}
{{< badge content="TACACS+" color="red" >}}