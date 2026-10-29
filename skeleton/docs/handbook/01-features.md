# Features

## Aurora PostgreSQL Serverless v2 cluster

`template.yaml` defines an `${{ values.db_engine }}` cluster (`RdsDbCluster`) at
engine version `${{ values.db_engine_version }}` listening on port
`${{ values.db_port }}`. The version is a major version; AWS selects the minor
version when the cluster is created.

{% if values.db_name %}The cluster creates an initial database named
`${{ values.db_name }}` through `DatabaseName`.{% else %}No initial database
name was supplied, so the template omits `DatabaseName`. Connect to the
PostgreSQL default `postgres` database and create application databases
yourself.{% endif %}

| Feature | Starter behavior |
| --- | --- |
| Instances | `RdsDbInstance0` with instance class `db.serverless`. When `RdsEnableHA` is `"true"`, `RdsDbInstance1` and `RdsDbInstance2` are added, giving one writer and two readers across the cluster. The starter value is `"false"`. |
| Scaling | Serverless v2 capacity between `${{ values.min_capacity }}` and `${{ values.max_capacity }}` Aurora capacity units (ACUs). A minimum of `0` allows the cluster to auto-pause when idle; the first connection after a pause waits while it resumes. `SecondsUntilAutoPause` is commented out, so the AWS default applies. |
| Parameter group | `RdsDbClusterParameterGroup` uses family `DbFamily` and sets `rds.force_ssl`, `log_connections`, and `log_disconnections`. |
| Log exports | `postgresql` logs are exported to CloudWatch Logs. See [Observability](06-observability.md). |

Instance settings not shown in the template, such as availability zone, promotion
tier, and minor-version upgrade behavior, use AWS defaults.

## SSM parameters as the consumer contract

Consumers discover the database through SSM Parameter Store rather than stack
outputs. The template defines no `Outputs`. Each name is built from
`/${Domain}/${System}/${Component}/${CodeBranch}/`; for the `main` branch this
resolves to `/${{ values.domain_name }}/${{ values.system_name }}/${{ values.component_name }}/main/`.

| Parameter suffix | Value |
| --- | --- |
| `RdsDbClusterName` | Cluster identifier. |
| `RdsDbClusterHost` | Writer (read/write) endpoint hostname. |
| `RdsDbClusterROHost` | Reader endpoint hostname. |
| `RdsDbClusterPort` | Endpoint port. |
| `RdsDbAdminUsername` | Admin (master) username. |
| `RdsDbAdminPassword` | **ARN of the Secrets Manager secret**, not the password itself. |
| `RdsSecurityGroupId` | ID of the cluster's security group. |

All are `String` parameters. Treat these names as a public interface: changing
`Domain`, `System`, `Component`, or `CodeBranch` renames every parameter and breaks
consumers that read the old paths.

The starter has descriptions that need review: `RdsDbClusterROHostSsmParam` is
described as "DB endpoint port" and `RdsDbClusterPortSsmParam` as "Hostname of
read-only DB endpoint"; the values are correct, the descriptions are swapped.
`RdsDbAdminPasswordSsmParam` is described as the admin password but stores the
secret ARN. Correct these when adapting the stack.

## Admin credentials in Secrets Manager

`RdsDbAdminPassword` is a Secrets Manager secret generated with a 64-character
password, no punctuation, and a JSON body of `username` and `password`. The
cluster's `MasterUserPassword` is resolved from it by a CloudFormation dynamic
reference at deployment time, so the password is not stored in the template or
parameter files.

- The secret contains only `username` and `password`. It is not attached to the
  cluster, so it has no `host`, `port`, or `engine` fields.
- Rotation is not configured. See [Common tasks](07-common-tasks.md#rotate-the-admin-password).
- The secret has `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain`; deleting
  the stack leaves it behind.

These are the cluster's **admin** credentials. Create separate least-privilege
database roles for applications rather than sharing them.

## Networking and access

The cluster is placed in the account's private subnets, read from the SSM
parameter `/org/networking/VpcPrivateSubnets`, inside the VPC from
`/org/networking/VpcId`. `RdsSecurityGroup` is created in that VPC with **no
ingress rules**, so nothing can connect until access is granted explicitly.
The group's ID is published to `RdsSecurityGroupId` so consumers can add an
ingress rule from their own security group. See
[Common tasks](07-common-tasks.md#grant-a-service-access).

The stack does not create a bastion host, VPN, or RDS Proxy, and it does not
enable IAM database authentication.

## Encryption and TLS

Storage is encrypted with the AWS-managed `alias/aws/rds` KMS key.
`rds.force_ssl` rejects unencrypted client connections, so clients must use TLS.
Use `sslmode=verify-full` with the Amazon RDS CA bundle to verify the server.

## Backups, maintenance, and deletion safety

| Setting | Starter value |
| --- | --- |
| Backup retention | 7 days (`RdsBackupRetentionPeriod`) |
| Backup window (UTC) | `03:01-05:00` (`RdsPreferredBackupWindow`) |
| Maintenance window (UTC) | `sun:1:00-sun:03:00` (`RdsPreferredMaintenanceWindow`) |
| Cluster deletion / replacement | `DeletionPolicy: Snapshot` and `UpdateReplacePolicy: Snapshot` |
| Deletion protection | `false` |
| Tags on snapshots | `CopyTagsToSnapshot: true` |

The snapshot policies take a final snapshot when CloudFormation deletes or
replaces the cluster. A replacement cluster starts **empty**; data must be
restored from that snapshot manually. Deletion protection is off, so a stack
deletion or a replacing update is not blocked. Enable it before production use.
`UseLatestRestorableTime` only affects point-in-time restores and has no effect
on a newly created cluster.

## CI/CD and feature branches

GitHub Actions supplies separate main and non-main branch workflows. Both
validate and build with SAM and deploy after a successful build. There are no
unit tests to run. Branch deployments need deliberate stack isolation; both
workflows currently select the `production` GitHub environment.
See [CI/CD pipeline](05-ci-cd.md) before pushing a feature branch.
