# Observability

## Database logging

`EnableCloudwatchLogsExports: [postgresql]` publishes the PostgreSQL log to
CloudWatch Logs. RDS creates the log group
`/aws/rds/cluster/<cluster-identifier>/postgresql`; find the identifier in the
stack resources or the `RdsDbClusterName` SSM parameter.

The cluster parameter group enables:

| Setting | Effect |
| --- | --- |
| `log_connections: 'all'` | Logs connection attempts and authentication details, such as user, database, and client host. |
| `log_disconnections: true` | Logs session end and duration. |
| `rds.force_ssl: true` | Rejects non-TLS connections; rejected attempts appear in the log. |

Statement logging, such as `log_statement` or `log_min_duration_statement`, is not
configured. If you enable it for debugging, remember that statements can contain
personal data and literal values. Never log passwords; for example, avoid
`ALTER ROLE ... PASSWORD` statements while statement logging is on.

The template does not set log-group retention, so RDS-created log groups keep
data indefinitely unless changed. Set an appropriate retention policy for your
service.

## Metrics

AWS provides standard Aurora metrics in CloudWatch, including:

| Metric | Why it matters |
| --- | --- |
| `ServerlessDatabaseCapacity`, `ACUUtilization` | Current ACUs and how close each instance is to the maximum. Sustained high utilization suggests raising `RdsScalingMaxCapacity`. |
| `CPUUtilization`, `FreeableMemory` | Load relative to current capacity. |
| `DatabaseConnections` | Open connections; the connection limit depends on capacity. |
| `ReadLatency`, `WriteLatency`, `CommitLatency` | I/O and transaction latency. |
| `Deadlocks` | Application concurrency problems. |
| `AuroraReplicaLag` | Reader lag behind the writer when HA instances exist. |
| `VolumeBytesUsed` | Cluster storage consumed. |

Filter by the actual cluster and instance identifiers, especially when
investigating branch deployments. A capacity of `0` with no connections is
expected while the cluster is auto-paused.

The starter does not enable Performance Insights or Database Insights, Enhanced
Monitoring, alarms, or dashboards. Add these intentionally based on your
service's reliability targets.

## Availability in Datadog

The Datadog AWS integration and log shipping are **configured automatically in
all AWS accounts**. Developers do not need to enable the integration or configure
log forwarding for generated services. Use Datadog to inspect the cluster's AWS
metrics and shipped PostgreSQL logs.

Baseline tagging is also automatic: `cfn-tags.json` supplies `org:domain`,
`org:system`, and `org:component`, and `ServerlessOpsIO/gha-deploy-aws-sam` adds
additional deployment tags. Use the existing tags to locate and distinguish
resources; extra tagging is only needed for service-specific requirements.

Datadog Database Monitoring, which needs an agent with network access and a
dedicated monitoring role in the database, is not configured by this template.

After deployment, verify that a test connection appears in the PostgreSQL log
and that recent cluster metrics appear in Datadog.

In Datadog Logs Explorer, locate the cluster's log source or identifier and
narrow the time range. Use Metrics Explorer to inspect the imported RDS metrics
by cluster and instance identifier. Exact field names and tags depend on the
integration's pipelines. If data is missing, first confirm it exists in
CloudWatch and check the Datadog time range and resource/tag filters. Treat
persistent missing data as a problem with the existing collection or shipping
configuration, not a requirement to set up a new integration.

## Connection investigation checklist

1. Note the timestamp, client, target endpoint (writer or reader), and the exact
   error.
2. A timeout usually means the network path: check the client's subnet routing
   and that `RdsSecurityGroup` has an ingress rule from the client's security group
   on port `${{ values.db_port }}`.
3. A TLS or `pg_hba.conf`/no-encryption error means the client is not using TLS.
4. An authentication failure means a wrong username, password, or database name;
   confirm against the secret and the PostgreSQL log, without copying the
   password into tickets.
5. A slow first connection may be an auto-pause resume. Compare
   `ServerlessDatabaseCapacity` around the event.
6. Compare connection, capacity, and latency metrics with recent deployments and
   maintenance windows.
7. Check Datadog ingestion separately if CloudWatch data is present but missing
   from Datadog.
