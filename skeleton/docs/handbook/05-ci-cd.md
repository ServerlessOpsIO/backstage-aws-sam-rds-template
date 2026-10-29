# CI/CD pipeline

## Workflow triggers

| Workflow | Automatic trigger | Manual trigger |
| --- | --- | --- |
| `.github/workflows/main.yml` (`Main`) | Push to `main` | `workflow_dispatch` |
| `.github/workflows/branch.yml` (`Branches`) | Push to any branch except `main` | `workflow_dispatch` |

There is no `pull_request` trigger. A push to a feature branch starts its
workflow without needing a PR. Manual dispatch is also supported; select the
intended workflow and ref carefully.

Both workflows run on `ubuntu-latest` and grant jobs `contents: read` and
`id-token: write`. The latter allows OIDC-based AWS role assumption rather
than committing long-lived AWS credentials.

## Build job

The build job executes these steps in order:

1. `ServerlessOpsIO/gha-setup-workspace@v2` sets up the checked-out workspace.
2. `ServerlessOpsIO/gha-assume-aws-credentials@v1` obtains credentials using
   `AWS_CICD_ACCOUNT_ID`.
3. `aws-actions/setup-sam@v3` installs SAM.
4. `sam validate --lint` validates the template with CloudFormation linting.
5. `sam build` prepares the deployable template.
6. `ServerlessOpsIO/gha-store-artifacts@v2`, with `use_aws_sam: true`, stores
   the build output for the deploy job.

There are no unit, integration, or migration steps, and no TechDocs generation.
Linting checks template structure, not whether the deployment will succeed or
whether a change replaces the cluster. Add checks explicitly if your project
requires them.

## Deploy job

`deploy` depends on `build` and only proceeds after it succeeds. It uses the
GitHub environment named **`production`** in both workflows.

1. The workspace action runs with `checkout_artifact: true` to retrieve the
   build artifact.
2. The credential action receives both `AWS_CICD_ACCOUNT_ID` and
   `DEPLOYMENT_ACCOUNT_ID` for build/deployment account access.
3. `ServerlessOpsIO/gha-deploy-aws-sam@v1` deploys with the target account ID
   and serialized workflow environment, variables, and secrets.

The shared actions implement artifact handling, credential assumptions, and
parameter/tag expression resolution. Their internals are external to this
repository. Confirm the expected IAM roles, region, artifact storage, and
stack-name behavior against the shared actions' configuration.

Creating an Aurora cluster and its instances commonly takes several minutes or
more, so the first deployment is slower than later updates. Changeset
confirmation is disabled: whatever the change set contains, including a cluster
replacement, is applied without a manual review step. GitHub environment
protection rules can require deployment approval or restrict which branches
deploy.

See [Configuration](03-configuration.md) for parameter files, tags, secrets,
networking prerequisites, and SAM defaults.

## Feature branch builds and deployments

Branch support means a non-main push executes **both build and deploy**, not
just CI checks. `CodeBranch` is resolved from `$env.GITHUB_REF_SLUG_CS` and
appears in every SSM parameter name. That alone does not isolate a deployment:

- `samconfig.toml` has a component-wide default stack name,
  `${{ values.component_name }}-db`.
- Both workflows select the same GitHub environment, `production`, and the same
  deployment account.

If the shared deployment action uses that stack name, a branch push **updates the
main database stack** and changes `CodeBranch`, which renames all seven SSM
parameters. Consumers reading the `main` paths then fail to find them.
If the action does derive a separate stack per branch, each branch creates its
own cluster, secret, and snapshots, with their cost.

Before enabling branch deployments, establish and verify a platform policy:

1. Give each deployed branch a distinct stack name through the shared deployment
   action's supported configuration, or restrict `branch.yml` from deploying.
2. Use an appropriate account/environment and OIDC trust policy for branch
   workloads, with approval rules that prevent unintended production changes.
3. Consider lower capacity, `RdsEnableHA` off, and short backup retention for
   branch clusters.
4. Confirm the branch slug, final stack name, and SSM paths in the deployment
   output before connecting to the database.
5. Define teardown for branch stacks. There is no branch-deletion cleanup
   workflow, and deleting a stack leaves a final cluster snapshot and the
   retained secret behind.

Without this policy, do not assume a branch push creates a safe preview
environment out of the box.

## Reviewing a deployment

Inspect the GitHub Actions build and deploy logs, then check the deployed
CloudFormation stack's status, events, and resources. Confirm the cluster and
instances are `available`, that the SSM parameters exist under the expected path,
and that a permitted client can connect over TLS. Use
[Observability](06-observability.md) to check connection logs and metrics.
