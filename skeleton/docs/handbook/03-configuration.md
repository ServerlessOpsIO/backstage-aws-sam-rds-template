# Configuration

## Stack parameters: `cfn-parameters.json`

This file supplies the deployment action with values for the `Parameters` in
`template.yaml`.

| Parameter | Starter value/source | Meaning |
| --- | --- | --- |
| `Domain` | Selected catalog domain title | Tag value and first SSM path segment. |
| `System` | Selected catalog system title | Tag value and second SSM path segment. |
| `Component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` | Repository-derived component identifier provided through the workflow environment. |
| `CodeBranch` | `$env.GITHUB_REF_SLUG_CS` | Branch slug; the last SSM path segment. |
| `VpcId` | `/org/networking/VpcId` | **SSM parameter name**, resolved by CloudFormation through `AWS::SSM::Parameter::Value<String>`. |
| `VpcSubnets` | `/org/networking/VpcPrivateSubnets` | **SSM parameter name** holding a comma-delimited list of private subnet IDs. |
| `DbEngine` | `${{ values.db_engine }}` | Cluster engine. |
| `DbEngineVersion` | `${{ values.db_engine_version }}` | Major engine version. |
| `DbFamily` | `${{ values.db_engine }}${{ values.db_engine_version }}` | Cluster parameter group family; must match engine and major version. |
| `DbPort` | `${{ values.db_port }}` | Listener port. |
| `DbUsername` | `${{ values.db_username }}` | Admin username. |
| `DbName` | Optional; present only when a name was supplied | Initial database name. |
| `RdsScalingMinCapacity` | `${{ values.min_capacity }}` | Minimum ACUs. |
| `RdsScalingMaxCapacity` | `${{ values.max_capacity }}` | Maximum ACUs. |
| `RdsEnableHA` | `"false"` | `"true"` adds two more cluster instances. |
| `RdsBackupRetentionPeriod` | `"7"` | Days of automated backups. |
| `RdsPreferredMaintenanceWindow` | `"sun:1:00-sun:03:00"` | Weekly maintenance window, UTC. |
| `RdsPreferredBackupWindow` | `"03:01-05:00"` | Daily backup window, UTC; must not overlap maintenance. |

The `$env` expressions are intentionally unquoted in the starter file. It is
deployment-action input, **not strict JSON** for direct consumption by
`json.load`, `jq`, or the SAM CLI. The same applies to expressions in
`cfn-tags.json`. The workflow passes `env_json`, `vars_json`, and `secrets_json`
to `ServerlessOpsIO/gha-deploy-aws-sam@v1` for resolution.
Consult that action's configuration when changing expression syntax or adding
environment-specific values.

For a manual deployment, supply resolved CloudFormation parameter values through
SAM's parameter overrides; do not pass this file as though it were the AWS CLI's
parameter-list format. Never add a password or other credential to this file.

## Stack tags: `cfn-tags.json`

| Tag | Value |
| --- | --- |
| `org:domain` | Selected domain title |
| `org:system` | Selected system title |
| `org:component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` |

Keep these aligned with catalog metadata and your organization's inventory and
cost-allocation conventions. Add tags through the existing deployment action;
CloudFormation propagates stack tags to resources that support them, and
`CopyTagsToSnapshot` carries cluster tags to its snapshots.
These metadata tags are not runtime configuration or access controls.

## Deployment prerequisites

- A target AWS account and deployment region with the required IAM permissions
  to manage RDS, EC2 security groups, Secrets Manager, and SSM parameters.
- SSM parameters `/org/networking/VpcId` and `/org/networking/VpcPrivateSubnets`
  in the deployment account and region. The subnets must span at least two
  availability zones, as required by a DB subnet group.
- Domain and system titles that are valid SSM parameter path segments.
- Aurora PostgreSQL `${{ values.db_engine_version }}` and Serverless v2 available
  in the deployment region.
- GitHub Actions secrets `AWS_CICD_ACCOUNT_ID` and `DEPLOYMENT_ACCOUNT_ID`,
  available to the relevant jobs. The scaffolder only creates
  `DEPLOYMENT_ACCOUNT_ID`; configure the other through your organization's
  repository or environment settings.
- AWS OIDC trust and build/deployment roles expected by the shared
  `gha-assume-aws-credentials` action. Account IDs alone do not grant deployment
  access. Verify role names, trust restrictions, region, and artifact storage
  against the shared action's requirements.
- GitHub environment `production` settings appropriate for **both** workflows,
  including any required approvals and branch restrictions.

## SAM settings

`samconfig.toml` sets the default stack name to `${{ values.component_name }}-db`,
disables changeset confirmation, and requests `CAPABILITY_NAMED_IAM`. It does not
define separate branch environments. Verify the final stack name used by the
shared deployment action before deploying.

`template.yaml` has no `Globals` and no Lambda functions; it is
infrastructure-only. `.cfnlintrc` suppresses cfn-lint check `E3034`.

Because changeset confirmation is disabled and the pipeline deploys on push,
review changes that could replace the cluster before merging. See
[Common tasks](07-common-tasks.md#changes-that-replace-the-cluster).

`CodeBranch` is part of every SSM parameter name. See
[CI/CD pipeline](05-ci-cd.md) before deploying from a branch other than `main`.
