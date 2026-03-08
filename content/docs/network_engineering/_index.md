---
draft: false
title: 'Ansible Network Automation Lab'
---

{{< cards cols="2">}}
  {{< card link="part_1/" title="1. Foundation" subtitle="I focus on setting up my Ubuntu lab environment using VS Code Remote-SSH, implementing SSH key authentication and basic Linux workflows, and preparing the system with tools, directory structure, and tmux sessions to support reliable Ansible automation work." >}}
  {{< card link="part_2/" title="2. Python" subtitle="I cover core concepts such as variables, lists, dictionaries, loops, functions, JSON/YAML parsing, and common libraries like requests, Netmiko, and NAPALM, focusing on how Python underpins Ansible modules, automation workflows, and troubleshooting." >}}

  {{< card link="part_3/" title="3. Git" subtitle="This part explains how I use Git and GitHub as disciplined version control tools for network automation, not just for basic commits and pushes. It covers building a safe team workflow with repo setup, .gitignore, branching, pull requests, peer review, rollback strategies, and secret-scanning protections, so infrastructure code stays traceable, reviewable, and secure." >}}
  {{< card link="part_4/" title="4. Virtualenv" subtitle="This part explains how I use Python virtual environments to keep my Ansible lab isolated, reproducible, and safe from breaking Ubuntu’s system Python. I cover creating and activating a dedicated environment, installing the full Ansible networking toolchain, freezing dependencies with requirements.txt, managing Ansible collections, and configuring VS Code to use the correct interpreter so the whole project stays clean, portable, and easy to rebuild." >}}

  {{< card link="part_5/" title="5. Ansible Install" subtitle="This part explains what Ansible actually is under the hood, the relationship between ansible, ansible-core, collections, modules, plugins, and the control node architecture so I can move beyond just running playbooks and truly understand how they execute." >}}
  {{< card link="part_6/" title="6. Containerlab" subtitle="This part explains how I use Containerlab to build a fast, reproducible, multi-vendor network lab on my Ubuntu VM using Docker-based network OS containers. It covers installing Docker and Containerlab, designing and deploying an enterprise-style topology with Cisco, NX-OS, Palo Alto, and Linux nodes, creating startup configs, and managing the lab lifecycle so I have a realistic environment ready for Ansible-driven network automation." >}}
{{< /cards >}}