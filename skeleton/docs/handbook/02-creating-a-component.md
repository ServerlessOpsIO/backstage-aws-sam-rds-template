# Creating a component via Backstage

Choose **Create** in Backstage and select **AWS RDS**. The platform needs
GitHub publishing, catalog registration, the template's custom form fields, and
catalog entities for the choices below.

## Page 1: RDS Cluster Information

Paths below are relative to the **newly generated repository root**, not the
template source repository. Unlike some other templates, this one passes each
catalog selection's **`metadata.title`** into generated files, not its entity
name; Account ID uses the title as the AWS account ID.

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Component name | Unique component and repository name; lowercased by the template. | GitHub repository name; Component `metadata.name` and the `github.com/project-slug` annotation in `catalog-info.yaml`; `stack_name` (`<name>-db`) in `samconfig.toml`; the top-level `Description` in `template.yaml`; headings in `README.md` and `docs/handbook/00-index.md`; `site_name` in `mkdocs.yml`. `Component` in `cfn-parameters.json` and `org:component` in `cfn-tags.json` resolve from the repository-name workflow variable rather than a literal form value. |
| Description | Describes the repository, component, and SAM application. | GitHub repository description; Component `metadata.description` in `catalog-info.yaml`; top-level `Description` in `template.yaml`; introductory text in `README.md`. |
| Owner | Owning group selected from your groups. | Component `spec.owner` in `catalog-info.yaml` uses `group:<selected-group-title>`; GitHub repository Settings → Custom properties → `owner` contains the group's entity name. |
| Domain | Catalog domain, used for deployment metadata, tagging, and SSM paths. | `Domain` in `cfn-parameters.json`; `org:domain` in `cfn-tags.json`; the first segment of every SSM parameter name; GitHub Settings → Custom properties → `domain`. There is no direct domain relationship in `catalog-info.yaml`. |
| System | Catalog system filtered by the chosen domain. | Component `spec.system` in `catalog-info.yaml`; `System` in `cfn-parameters.json`; `org:system` in `cfn-tags.json`; the second segment of every SSM parameter name; the `README.md` heading; GitHub Settings → Custom properties → `system`. |
| Environment | Catalog environment filtered by the chosen system; narrows the cloud-account picker. | Scaffolder selection only: the chosen environment is not written to a generated file. Both workflow files independently set `jobs.deploy.environment: production`. |
| Account ID | Catalog cloud-account resource related to the environment; its `metadata.title` must contain the AWS account ID. | GitHub Settings → Secrets and variables → Actions → repository secret `DEPLOYMENT_ACCOUNT_ID`; both workflow files reference it. GitHub shows the secret's name but not its value; confirm the source entity's title before creation. |

Because owner and system references in `catalog-info.yaml` are built from
titles, they resolve only when each title equals the entity's name. Domain and
system titles also become SSM parameter path segments, which allow only letters,
numbers, and `.`, `-`, `_`, and `/`. A title containing spaces fails deployment.

Hidden defaults are lifecycle `production`, namespace `default`, and GitHub
organization `ServerlessOpsIO`. Namespace is collected but not written to
generated files. The GitHub organization appears in the `github.com/project-slug`
annotation; the publish step itself hard-codes `ServerlessOpsIO`, so the two
must stay in agreement.

## Page 2: RDS Cluster Configuration

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| DB Engine | Database engine; only `aurora-postgresql` is offered. | `DbEngine` and the prefix of `DbFamily` in `cfn-parameters.json`. |
| DB Engine Version | Major engine version; only `18` is offered. | `DbEngineVersion` and the suffix of `DbFamily` (for example `aurora-postgresql18`) in `cfn-parameters.json`. |
| DB Name | Optional initial database name. Use a PostgreSQL identifier: start with a letter, then letters, digits, or underscores. | When supplied, `DbName` in `cfn-parameters.json`, the `DbName` parameter in `template.yaml`, and `RdsDbCluster.Properties.DatabaseName`. When omitted, all three are absent. |
| DB Username | Admin (master) username; defaults to `postgres`. | `DbUsername` in `cfn-parameters.json`; the secret's `username` field and the `RdsDbAdminUsername` SSM parameter after deployment. |
| DB Port | Listener port; defaults to `5432`. | `DbPort` in `cfn-parameters.json`; the `RdsDbClusterPort` SSM parameter after deployment. |
| Min Capacity | Minimum Serverless v2 capacity in ACUs; defaults to `0`, which allows auto-pause. | `RdsScalingMinCapacity` in `cfn-parameters.json`. |
| Max Capacity | Maximum Serverless v2 capacity in ACUs; defaults to `8`. Must be at least the minimum. | `RdsScalingMaxCapacity` in `cfn-parameters.json`. |

The form does not validate the database name, the capacity range, or that the
minimum is not greater than the maximum. Errors surface during deployment.

Backup retention, backup and maintenance windows, and high availability are not
on the form. They are fixed starter values in `cfn-parameters.json`; see
[Configuration](03-configuration.md).

These values are copied into files at creation time, not maintained by a single
live configuration source. Several of them, including the engine, username,
and initial database name, **replace the cluster** if changed after it is
deployed. Read [Common tasks](07-common-tasks.md#changes-that-replace-the-cluster)
before editing them.

## Review and create

Review the form summary, especially the repository name, account, and database
settings, before submitting. The scaffolder then:

1. Fetches the selected domain, system, owner, and account catalog entities.
2. Renders the project skeleton, including the SAM template and this handbook.
3. Renders the GitHub Actions workflows.
4. Publishes a **public** repository with `main` as the default branch, sets
   `DEPLOYMENT_ACCOUNT_ID`, and sets the `system`, `domain`, and `owner` custom
   properties.
5. Registers the generated Component in Backstage.

The template defines no output links; find the repository in GitHub and the
component in the catalog. Default branch protection is disabled and the
required approving review count is zero in the publisher configuration; apply
your review policy before development.

The initial commit to `main` starts the `Main` workflow, which deploys the
cluster. Confirm the [deployment prerequisites](03-configuration.md#deployment-prerequisites)
are in place, or expect the first run to fail.

## After creation

- Confirm catalog ownership and system relationships resolve.
- Check the first `Main` workflow run and the deployed CloudFormation stack.
- Review [branch deployment isolation](05-ci-cd.md) before pushing additional branches.
- Enable deletion protection and review backup settings before storing real data.
- Grant consumers access using [Common tasks](07-common-tasks.md), and request
  human review of the generated handbook.
