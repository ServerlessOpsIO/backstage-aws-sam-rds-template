# Common changes and tasks

## Set up a development environment

Install the AWS SAM CLI and the AWS CLI. From the **generated repository root**,
run:

```sh
sam validate --lint
sam build
```

These are the same checks the pipeline runs. They require the deployment tooling
and relevant AWS configuration but do not deploy anything. To connect to the
database you also need a PostgreSQL client such as `psql`, and, for access from
outside the VPC, whatever path your platform provides (see below).

## Look up connection details

The SSM parameters under the stack's prefix hold everything except the password.
For the `main` deployment:

```sh
PREFIX=/${{ values.domain_name }}/${{ values.system_name }}/${{ values.component_name }}/main
aws ssm get-parameters-by-path --path "$PREFIX/" \
  --query 'Parameters[].[Name,Value]' --output table
```

The password is in the secret whose ARN is in `$PREFIX/RdsDbAdminPassword`.
Reading it requires `secretsmanager:GetSecretValue` on that secret. Avoid
printing it to shared terminals or logs; prefer passing it to your client
through an environment variable such as `PGPASSWORD` in a private shell.

## Connect to the database

The cluster is reachable only from inside the VPC, from clients whose security
group is allowed by `RdsSecurityGroup`, and only over TLS.

1. Get a network path: run the client in the VPC, or use a platform-provided
   path such as a VPN or Session Manager port forwarding through an instance in
   the VPC. The template supplies none of these.
2. Allow that client's security group as described in the next section.
3. Download the Amazon RDS CA bundle, for example
   `https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem`.
4. Connect with certificate verification:

   ```sh
   psql "host=<RdsDbClusterHost> port=${{ values.db_port }} dbname=<database> \
     user=${{ values.db_username }} sslmode=verify-full sslrootcert=global-bundle.pem"
   ```

   With port forwarding, `verify-full` fails against `localhost` because the
   certificate names the cluster endpoint; map the endpoint hostname to the local
   address instead of downgrading `sslmode`.

Use the writer endpoint for changes and the reader endpoint (`RdsDbClusterROHost`)
for read-only work. Without HA there is no separate reader instance, and the
reader endpoint routes to the writer.

## Grant a service access

A consuming service needs three things; grant each in the consumer's own stack
where possible so access is owned by the consumer.

1. **Network.** Add an `AWS::EC2::SecurityGroupIngress` with `GroupId` resolved
   from the `RdsSecurityGroupId` SSM parameter, `SourceSecurityGroupId` set to the
   consumer's security group, protocol `tcp`, and port `${{ values.db_port }}`.
   Prefer a source security group to CIDR ranges. Lambda functions must run in
   the VPC to use this path.
2. **Discovery.** Allow `ssm:GetParameter` or `ssm:GetParametersByPath` on the
   stack's parameter prefix, or resolve the values at deploy time with SSM
   parameter types.
3. **Credentials.** Create a dedicated database role for the service (next
   section), store its credential in a secret the service can read, and grant
   `secretsmanager:GetSecretValue` on that secret only. Do not grant applications
   the admin secret.

Record the relationship in the consumer's `catalog-info.yaml`, for example with
`dependsOn: component:${{ values.component_name }}`.

## Create databases and application roles

The stack creates only the admin user{% if values.db_name %} and the
`${{ values.db_name }}` database{% endif %}. Using the admin credential, create
what each application needs:

```sql
CREATE DATABASE app_db;
CREATE ROLE app_user LOGIN PASSWORD '<generated password>';
GRANT CONNECT ON DATABASE app_db TO app_user;
-- then, connected to app_db, grant schema and table privileges
```

Generate the password rather than choosing one, and keep it in a secret rather
than in source or tickets. Manage ongoing schema changes with a migration tool
run from a pipeline that has network and credential access; this repository does
not provide one.

## Change capacity, high availability, or auto-pause

- **Capacity.** Edit `RdsScalingMinCapacity` and `RdsScalingMaxCapacity` in
  `cfn-parameters.json`. Changes apply without replacing the cluster. Raise the
  maximum if `ACUUtilization` stays high; raise the minimum if resume or scale-up
  latency hurts your service.
- **Auto-pause.** A minimum of `0` allows pausing. To tune the idle time,
  uncomment `SecondsUntilAutoPause` in `template.yaml`. Set the minimum above `0` for
  services that cannot tolerate a resume delay.
- **High availability.** Set `RdsEnableHA` to `"true"` to add two instances. This
  adds instances to the existing cluster and increases cost; it does not move data.

## Change backups and maintenance windows

Edit `RdsBackupRetentionPeriod`, `RdsPreferredBackupWindow`, and
`RdsPreferredMaintenanceWindow` in `cfn-parameters.json`. Windows are in UTC; the
backup window uses `hh24:mi-hh24:mi`, the maintenance window uses
`ddd:hh24:mi-ddd:hh24:mi`, and the two must not overlap. Retention applies to
automated backups; manual and final snapshots are kept until you delete them.

## Protect the database before production use

1. Set `DeletionProtection: true` on `RdsDbCluster`. With it on, deleting the
   cluster, by stack deletion or replacement, fails until you turn it off first.
2. Review backup retention against your recovery objectives.
3. Consider `RdsEnableHA` for workloads that need fast failover.
4. Set retention on the PostgreSQL log group and add alarms; see
   [Observability](06-observability.md).

## Changes that replace the cluster

Some property changes make CloudFormation create a **new, empty cluster** and
delete the old one. `UpdateReplacePolicy: Snapshot` takes a final snapshot first,
but data is not copied to the new cluster, the endpoints change, and consumers
lose data until you restore it. Examples include changing:

- the engine (`DbEngine`);
- the admin username (`DbUsername`);
- the initial database name (`DbName`), including adding or removing it;
- the subnet group or encryption settings.

Before merging a change to `template.yaml` or `cfn-parameters.json`, check the
CloudFormation documentation's *Update requires* entry for each changed property,
or create a change set without executing it (for example `sam deploy
--no-execute-changeset` against a non-production stack) and look for
`Replacement: True`. Enabling deletion protection makes an accidental replacement
fail instead of proceeding.

## Upgrade the engine version

Minor-version upgrades are applied by AWS according to the instances' default
settings, during the maintenance window. For a major-version upgrade:

1. Check the upgrade path and extension compatibility in the Aurora PostgreSQL
   documentation, and test on a restored copy.
2. Update both `DbEngineVersion` and `DbFamily` in `cfn-parameters.json`; the
   parameter group family must match the new major version.
3. Review the CloudFormation `AWS::RDS::DBCluster` documentation for properties
   needed to permit a major-version upgrade, and expect downtime.
4. Take a manual snapshot before deploying.

## Rotate the admin password

Rotation is not configured. The secret holds only `username` and `password`, so
the standard Secrets Manager rotation functions, which also expect connection
fields such as `host`, do not work with it unchanged. Options:

- Change the password in the database and update the secret together, during a
  window when consumers using the admin credential can tolerate a failure.
- Change the template to use RDS-managed credentials (`ManageMasterUserPassword`),
  which rotate automatically. This replaces the secret ARN consumers read, so
  plan it as a contract change.

Either way, CloudFormation does not notice secret value changes; see
[Architecture](04-architecture.md#anatomy-of-the-credential).

## Restore from a backup

Point-in-time and snapshot restores create a **new cluster** outside this
stack. To return to managed infrastructure, restore into a cluster you then
point consumers to, or plan how the stack will adopt or replace it.
Practice the procedure before you need it, and record the steps in this handbook.

## Validate and deploy changes

Run `sam validate --lint` and `sam build`. Review whether the change replaces
the cluster, renames SSM parameters, or changes network access. Commit the
template, parameter, and handbook updates together, and open a PR under your
review policy. Remember that **pushing a feature branch starts deployment**:
complete the [branch-isolation setup](05-ci-cd.md) before pushing.

After deployment, confirm the stack is healthy, the SSM parameters are correct,
and a permitted client can connect over TLS, then obtain human review. Linting
cannot prove that networking, IAM, or credentials work.
