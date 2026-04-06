---
draft: false
title: '2nd Ansible for Network Automation Project'
sidebar:
  exclude: false
weight: 2
---

---

This project treats the network as code. Every device configuration, every IP assignment, every VLAN, and every monitoring target is defined in a source of truth and enforced by Ansible. Changes go through Git approval, which validated everything before deployment, and are monitored for configuration drift. When drift is detected, the platform remediates it automatically (with human approval).

I broke it down into 38 parts, with each part building onto the previous one. This is my 2nd iteration of my 1st Ansible for Network Automation Project.

---

{{< accent-label >}}Network topology{{< /accent-label >}}

---

{{< accent-label >}}Infrastructure Stack{{< /accent-label >}}

Each service runs on its own dedicated VM since I tried to mirror enterprise isolation practices. Every service integrates with NetBox for device data and FreeIPA for authentication.

{{< table >}}
Service | Purpose | Integration
**NetBox** | Source of truth (IPAM, DCIM) | Dynamic inventory for Ansible, service discovery for monitoring
**Gitea** | Self-hosted Git with branch protection | PR pipeline, Oxidized config backend, AWX project source
**AWX** | Automation controller | Job scheduling, RBAC, webhook-triggered deployments
**Zabbix** | Networking monitoring (SNMP) | Auto-discovers devices from Netbox
**Prometheus + Grafana** | Metrics and dashboards | Node exporter on all VMs, NetBox service discovery
**Graylog + OpenSearch** | Centralized log management | Syslog over TLS, NetBox enrichment, alerting
**Oxidized** | Config backup and versioning | Gitea backend, diff-on-change alerts
**FreeIPA** | Centralized identity and SSO | LDAP backend for TACACS+, SSO for all web UIs
**step-ca** | Internal PKI | TLS for syslog, HTTPS endpoints, ACME renewal
**Batfish** | Offline config analysis | Pre-change validation in the CI/CD pipeline
**pyATS / Genie** | Live network validation | Post-change verification, health snapshots
**Ansible EDA** | Event-driven automation | Closed-loop remediation triggered by Graylog
{{< /table >}}

---

{{< accent-label >}}Key Capabilities{{< /accent-label >}}

{{< feature-grid >}}
  {{< feature-card title="Intent-based configuration" mono="true" >}}
  Device configs generated from Jinja2 templates driven by NetBox data. Changing an interface description in NetBox and running the playbook updates the device. The network conforms to the source of truth.
  {{< /feature-card >}}

  {{< feature-card title="Dynamic inventory" mono="true" >}}
  No static hosts file. The NetBox plugin queries the API at runtime and builds the inventory from device records, platform assignments, and IP addresses. Adding a device means registering it in NetBox.
  {{< /feature-card >}}

  {{< feature-card title="Structured IPAM" mono="true" >}}
  A `10.33.0.0/16` supernet where the third octet encodes function (0 = loopbacks, 1 = P2P links, 10+ = VLAN ID). All addressing managed in NetBox IPAM and populated via Ansible.
  {{< /feature-card >}}

  {{< feature-card title="Multi-vault secrets" mono="true" >}}
  Three vault IDs separating trust boundaries: network device credentials, infrastructure service credentials, and PKI material. Pre-commit hooks block plaintext secrets from ever reaching Git.
  {{< /feature-card >}}

  {{< feature-card title="GitOps change pipeline" mono="true" wide="true" >}}
  Feature branch ➦ PR in Gitea ➦ Ansible lint ➦ Batfish analysis ➦ Peer review ➦ Merge ➦ AWX webhook ➦ Playbook run ➦ pyATS validation ➦ Oxidized backup
  {{< /feature-card >}}

  {{< feature-card title="Self-healing loop" mono="true" wide="true" >}}
  Drift detected ➦ Graylog alert ➦ EDA rulebook ➦ AWX workflow ➦ Approval gate ➦ Remediation ➦ Backup + verify
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
      image="/docs/network_engineering/part_1/ansiblebyredhat.jpg"
      desc="Unified dashboard with device availability, interface utilization, and BGP session states."
  >}}

  {{< showcase-card
      title="NetBox: Device inventory"
      desc="All 8 devices with sites, roles, platforms, and primary IPs populated."
  >}}

  {{< showcase-card
      title="AWX: Workflow execution"
      desc="Multi-step workflow with validation, deployment, and approval gates."
  >}}

  {{< showcase-card
      title="Gitea: PR with CI checks"
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