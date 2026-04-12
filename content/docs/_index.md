---
draft: false
title: 'Projects'
breadcrumbs: false
---

{{< projects-page >}}

  {{< projects-hero
      kicker=""
      title="Network & Systems Engineering Portfolio"
  >}}
  {{< /projects-hero >}}

  {{< projects-section label="Latest" >}}
    {{< featured-project
        title="2nd Network Automation with Ansible Project"
        desc="Designing, deploying, and managing enterprise routing, switching, and wireless with Ansible."
        status="progress"
        featured="false"
        image=""
        view_href="/docs/2nd-ansible-network-engineering/"
        github_href="https://github.com/ernestodiaztech"
    >}}
      {{< project-tag >}}Ansible{{< /project-tag >}}
      {{< project-tag >}}Git{{< /project-tag >}}
      {{< project-tag >}}Python{{< /project-tag >}}
      {{< project-tag >}}Networking{{< /project-tag >}}
    {{< /featured-project >}}
  {{< /projects-section >}}

  {{< projects-section label="All Projects" >}}
    {{< project-grid >}}

      {{< project-card
          title="Learning Network Automation with Ansible Project"
          desc="Learning how to design, deploy, and manage enterprise routing, switching, and wireless with Ansible."
          status="completed"
          image=""
          view_href="/docs/infra_mgmt/project-one/"
          github_href="https://github.com/ernestodiaztech/ansible-network"
      >}}
        {{< project-tag >}}Ansible{{< /project-tag >}}
        {{< project-tag >}}Python{{< /project-tag >}}
        {{< project-tag >}}Containerlab{{< /project-tag >}}
        {{< project-tag >}}Jinja2{{< /project-tag >}}
      {{< /project-card >}}

    {{< /project-grid >}}
  {{< /projects-section >}}

{{< /projects-page >}}