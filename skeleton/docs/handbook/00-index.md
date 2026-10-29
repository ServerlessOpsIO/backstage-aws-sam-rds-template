# Project handbook

This handbook describes the starter database stack generated for **${{ values.component_name }}**.
Keep it current as the cluster's configuration, consumers, and workflows change.

- [Features](01-features.md): Aurora PostgreSQL cluster, credentials, SSM parameters, networking, and backups.
- [Creating a component](02-creating-a-component.md): Backstage form pages and generated outputs.
- [Configuration](03-configuration.md): deployment parameters, tags, and prerequisites.
- [Architecture](04-architecture.md): AWS resources, template anatomy, and repository layout.
- [CI/CD pipeline](05-ci-cd.md): build, deployment, and feature branches.
- [Observability](06-observability.md): database logs, metrics, and automatic Datadog integration.
- [Common tasks](07-common-tasks.md): connect, grant access, resize, upgrade, and recover.

## Reading in Backstage

The component's `catalog-info.yaml` contains `backstage.io/techdocs-ref: dir:.`.
TechDocs uses the repository-root `mkdocs.yml` and the Markdown files in
`docs/handbook/`.
Open the component in the catalog and select **Docs**.

The platform must enable the TechDocs frontend and backend and configure either
on-demand documentation generation or an external builder and publisher. The
builder needs MkDocs and `mkdocs-techdocs-core` for the `techdocs-core` plugin.
The generated workflows do not build or publish documentation.
With the Backstage TechDocs CLI and its Docker prerequisites installed, preview
from the generated repository root with `techdocs-cli serve`.

The pages can also be read directly in GitHub. Source paths shown in
backticks are relative to the generated repository root.

## Content review

Before treating this starter handbook as your database's operational reference,
review its network access, credential handling, backup and deletion settings,
deployment assumptions, and runbooks. Template-generated documentation does not
constitute human approval.
