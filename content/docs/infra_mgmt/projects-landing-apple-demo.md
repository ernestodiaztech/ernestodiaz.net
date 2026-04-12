---
title: "Projects"
layout: "projects-landing"
---

{{< projects-page >}}

  {{< projects-hero
      kicker="Projects"
      title="Network Automation Portfolio"
      desc="Designing, deploying, and managing enterprise routing, switching, and wireless across distributed environments — all automated with Ansible."
  >}}
    {{< projects-stat value="3" label="Projects" >}}
    {{< projects-stat value="2" label="Active" >}}
    {{< projects-stat value="1" label="Completed" >}}
  {{< /projects-hero >}}

  {{< projects-section label="Featured" >}}
    {{< featured-project
        title="2nd Network Automation with Ansible Project"
        desc="Designing, deploying, and managing enterprise routing, switching, and wireless across distributed environments."
        status="active"
        featured="true"
        image=""
        view_href="/docs/infra_mgmt/project-landing-apple-demo/"
        github_href="https://github.com/yourusername/project-two"
    >}}
      {{< project-tag >}}Ansible{{< /project-tag >}}
      {{< project-tag >}}YAML{{< /project-tag >}}
      {{< project-tag >}}Python{{< /project-tag >}}
      {{< project-tag >}}Networking{{< /project-tag >}}
    {{< /featured-project >}}
  {{< /projects-section >}}

  {{< projects-section label="All Projects" >}}
    {{< project-grid >}}

      {{< project-card
          title="1st Network Automation with Ansible Project"
          desc="Designing, deploying, and managing enterprise routing, switching, and wireless across distributed environments."
          status="completed"
          image=""
          view_href="/docs/infra_mgmt/project-one/"
          github_href="https://github.com/yourusername/project-one"
      >}}
        {{< project-tag >}}Ansible{{< /project-tag >}}
        {{< project-tag >}}YAML{{< /project-tag >}}
      {{< /project-card >}}

      {{< project-card
          title="Infrastructure Management Project"
          desc="Automating infrastructure provisioning and configuration management for scalable network deployments."
          status="progress"
          image=""
          view_href="/docs/infra_mgmt/infrastructure-management/"
          github_href="https://github.com/yourusername/infrastructure-management"
      >}}
        {{< project-tag >}}Ansible{{< /project-tag >}}
        {{< project-tag >}}Python{{< /project-tag >}}
        {{< project-tag >}}Jinja2{{< /project-tag >}}
      {{< /project-card >}}

    {{< /project-grid >}}
  {{< /projects-section >}}

{{< /projects-page >}}
