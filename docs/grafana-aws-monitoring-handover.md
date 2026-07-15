# Grafana AWS Monitoring Handover

## Purpose

This document explains the AWS monitoring setup in Grafana for RDS, EC2, and ECS. It is intended for new team members who need to maintain dashboards, alert rules, and Teams notifications.

The setup uses AWS CloudWatch as the metrics source, Grafana dashboards for visualization, Grafana-managed alerting for threshold checks, and Microsoft Teams notifications through a webhook/workflow endpoint.

## Environment

| Item | Value |
| --- | --- |
| Grafana URL | `https://grafana.bhiveworkspace.io` |
| Runtime | Standalone Docker container |
| Container name | `grafana` |
| Grafana image observed | `grafana/grafana-oss:11.3.0-ubuntu` |
| AWS region | `ap-south-1` |
| CloudWatch datasource name | `cloudwatch` |
| CloudWatch datasource UID | `de3t7rxv00we8c` |

The datasource UID matters because Grafana alert rules reference datasource UIDs, not only datasource names. If the UID changes, alert rules can fail with `data source not found`.

## Local File Layout

Local project directory:

```text
/Users/harshitruwali/Developer/grafana_env/grafana-loki-otel
```

Grafana provisioning files:

```text
grafana/
  datasources/
    cloudwatch.yaml
  dashboards/
    providers/
      aws-infrastructure.yaml
    aws/
      aws-overview.json
      rds-overview.json
      ec2-overview.json
      ecs-overview.json
  alerting/
    rules.yaml
    contactpoints.yaml
    policies.yaml
  README.md
```

Documentation:

```text
docs/
  grafana-aws-monitoring-handover.md
```

## Server File Layout

Grafana runs as a standalone Docker container named `grafana`.

Alert provisioning files are loaded from this path inside the container:

```text
/etc/grafana/provisioning/alerting
```

Datasource provisioning is loaded from:

```text
/etc/grafana/provisioning/datasources
```

Dashboard provisioning is loaded from:

```text
/etc/grafana/provisioning/dashboards
```

If provisioning files are copied with `docker cp`, changes exist inside the current container. If the container is recreated, those copied files may be lost unless host volumes are mounted permanently.

## Dashboards

### AWS Infrastructure Overview

File:

```text
grafana/dashboards/aws/aws-overview.json
```

Purpose:

Provides a central overview for AWS infrastructure health.

Key panels:

| Panel | Purpose |
| --- | --- |
| RDS CPU | Shows RDS CPU utilization across RDS instances |
| RDS Connections | Shows database connection count across RDS instances |
| EC2 CPU | Shows EC2 CPU utilization across instances |
| EC2 Health | Shows EC2 status check failures |
| ECS CPU | Shows ECS service CPU utilization |
| ECS Memory | Shows ECS service memory utilization |
| Key Activity Trends | Shows a compact trend view across key metrics |

Implementation note:

This dashboard uses fixed `ap-south-1` CloudWatch `SEARCH(...)` expressions with wildcard dimensions. This was done because Grafana dashboard URL variables were causing blank panels when values such as region or resource selectors were empty or stale.

### AWS RDS Overview

File:

```text
grafana/dashboards/aws/rds-overview.json
```

Purpose:

Detailed RDS view for database health and load.

Panels:

| Panel | CloudWatch metric |
| --- | --- |
| RDS CPU Utilization | `AWS/RDS` `CPUUtilization` |
| RDS Database Connections | `AWS/RDS` `DatabaseConnections` |
| RDS Freeable Memory | `AWS/RDS` `FreeableMemory` |
| RDS Read/Write Latency | `AWS/RDS` `ReadLatency`, `WriteLatency` |

Implementation note:

The RDS dashboard uses CloudWatch `SEARCH(...)` expressions by RDS resource series. This avoids dependency on the `DB Instance` dashboard variable.

### AWS EC2 Overview

File:

```text
grafana/dashboards/aws/ec2-overview.json
```

Purpose:

Detailed EC2 instance monitoring.

Panels:

| Panel | CloudWatch metric |
| --- | --- |
| EC2 CPU Utilization | `AWS/EC2` `CPUUtilization` |
| EC2 Status Checks Failed | `AWS/EC2` `StatusCheckFailed_Instance`, `StatusCheckFailed_System` |
| EC2 Network In/Out | `AWS/EC2` `NetworkIn`, `NetworkOut` |
| EC2 Disk Read/Write Ops | `AWS/EC2` `DiskReadOps`, `DiskWriteOps` |

### AWS ECS Overview

File:

```text
grafana/dashboards/aws/ecs-overview.json
```

Purpose:

Detailed ECS service monitoring.

Panels:

| Panel | CloudWatch metric |
| --- | --- |
| ECS Service CPU Utilization | `AWS/ECS` `CPUUtilization` |
| ECS Service Memory Utilization | `AWS/ECS` `MemoryUtilization` |
| ECS Running Task Count | `ECS/ContainerInsights` `RunningTaskCount` |
| ECS Network Rx/Tx | `ECS/ContainerInsights` `NetworkRxBytes`, `NetworkTxBytes` |

Note:

Task count and network panels depend on ECS Container Insights. If Container Insights is not enabled, those panels can show no data while CPU and memory still work.

## CloudWatch Datasource

File:

```text
grafana/datasources/cloudwatch.yaml
```

Current expected settings:

```yaml
name: CloudWatch
uid: de3t7rxv00we8c
type: cloudwatch
jsonData:
  authType: default
  defaultRegion: ap-south-1
```

Important:

The UID must match the datasource UID on the server. The dashboards may render by datasource name or selected datasource, but alert rules depend heavily on the datasource UID.

If alerts show `data source not found`, check:

1. Grafana UI datasource settings.
2. The datasource UID in the browser URL or datasource details.
3. `grafana/alerting/rules.yaml` for `datasourceUid`.

## Alerting Overview

Alert rules are defined in:

```text
grafana/alerting/rules.yaml
```

Contact points are defined in:

```text
grafana/alerting/contactpoints.yaml
```

Notification policies are defined in:

```text
grafana/alerting/policies.yaml
```

Configured alert groups:

| Group | Purpose |
| --- | --- |
| `aws-rds` | RDS alert rules |
| `aws-ec2` | EC2 alert rules |
| `aws-ecs` | ECS alert rules |

Configured alert rules:

| Alert rule | Service | Condition |
| --- | --- | --- |
| RDS High Connections | RDS | Database connections above threshold |
| EC2 Instance Health Check Failed | EC2 | Instance health check failure |
| EC2 High CPU | EC2 | CPU utilization above threshold |
| ECS Service High CPU | ECS | Service CPU utilization above threshold |
| ECS Service High Memory | ECS | Service memory utilization above threshold |

## Alert Thresholds

Current thresholds:

| Alert | Threshold | Pending period |
| --- | --- | --- |
| RDS High Connections | `DatabaseConnections > 80` | `5m` |
| EC2 Instance Health Check Failed | `StatusCheckFailed_Instance > 0` | `5m` |
| EC2 High CPU | `CPUUtilization > 85%` | `10m` |
| ECS Service High CPU | `CPUUtilization > 85%` | `10m` |
| ECS Service High Memory | `MemoryUtilization > 90%` | `10m` |

`Pending` state means the threshold is currently breached, but the condition has not been true for the full pending period yet. For example, an RDS alert with `for: 5m` becomes `Firing` only after 5 continuous minutes above the threshold.

## Alert Behavior

Current rule behavior:

```yaml
noDataState: OK
execErrState: Error
```

Reasoning:

CloudWatch can return sparse or missing series. Setting `noDataState: OK` avoids noisy notifications such as `MissingSeries` for metrics that temporarily disappear.

Setting `execErrState: Error` makes query/configuration problems show as alert errors instead of real incidents.

## Teams Notifications

Teams notifications are configured in:

```text
grafana/alerting/contactpoints.yaml
```

The contact point is named:

```text
teams-webhook
```

Notification routing is configured in:

```text
grafana/alerting/policies.yaml
```

The policy routes alerts to:

```text
teams-webhook
```

The Teams message template includes:

- Status.
- Alert name.
- Resource or series name.
- Instance ID when available.
- DB instance when available.
- Service.
- Severity.
- Summary.
- Description.
- Expression values.
- Source link.
- Silence link.

Do not expose the Teams webhook URL in tickets, documentation, or commits. Treat it as a secret.

## Understanding Alert Values

Teams cards can include values like:

```text
Value: B=117.2, C=1
```

Meaning:

| Ref | Meaning |
| --- | --- |
| `A` | Raw CloudWatch query |
| `B` | Reduced metric value, usually the current value used for threshold evaluation |
| `C` | Threshold expression result |

For example:

```text
B=117.2, C=1
```

Means:

- Current reduced metric value is `117.2`.
- Threshold check is true.
- `C=1` means the alert condition is breached.
- `C=0` means the threshold condition is not breached.

## Resource Labels In Alerts

RDS alerts currently use:

```text
{{ $labels.Series }}
```

Grafana showed RDS resources under the `Series` label for the CloudWatch `SEARCH(...)` query output.

EC2 alerts can use labels such as:

```text
{{ $labels.InstanceId }}
```

If a label renders literally in Grafana or Teams, it means Grafana did not receive that label from the query result. In that case, inspect the alert instance labels in the Grafana UI and update the annotation template to use the actual label name.

## Grafana External URL

Grafana alert links must point to:

```text
https://grafana.bhiveworkspace.io
```

If Teams notifications show links like:

```text
http://localhost:3000/alerting/...
```

Then Grafana's external URL is not configured correctly.

For the running standalone container, update it with:

```bash
docker exec -u 0 grafana bash -lc "sed -i 's#^;*domain =.*#domain = grafana.bhiveworkspace.io#; s#^;*root_url =.*#root_url = https://grafana.bhiveworkspace.io/#' /etc/grafana/grafana.ini"
docker restart grafana
```

For a recreated container, pass:

```bash
-e GF_SERVER_DOMAIN=grafana.bhiveworkspace.io
-e GF_SERVER_ROOT_URL=https://grafana.bhiveworkspace.io/
```

## Deploying Alert Provisioning Files

Run these commands on the server where the Grafana Docker container is running.

Create the alerting provisioning directory:

```bash
docker exec -it grafana mkdir -p /etc/grafana/provisioning/alerting
```

Copy files into the container:

```bash
docker cp rules.yaml grafana:/etc/grafana/provisioning/alerting/rules.yaml
docker cp contactpoints.yaml grafana:/etc/grafana/provisioning/alerting/contactpoints.yaml
docker cp policies.yaml grafana:/etc/grafana/provisioning/alerting/policies.yaml
```

Restart Grafana:

```bash
docker restart grafana
```

Check logs:

```bash
docker logs grafana --tail 100
```

Check Teams-specific errors:

```bash
docker logs grafana --tail 100 | grep -i teams
```

## Deploying From Local Machine To Server

If the provisioning files are on a local machine, copy them to the server first:

```bash
scp /Users/harshitruwali/Developer/grafana_env/grafana-loki-otel/grafana/alerting/*.yaml user@server:/tmp/grafana-alerting/
```

Then SSH into the server and copy the files into the container:

```bash
docker exec -it grafana mkdir -p /etc/grafana/provisioning/alerting
docker cp /tmp/grafana-alerting/rules.yaml grafana:/etc/grafana/provisioning/alerting/rules.yaml
docker cp /tmp/grafana-alerting/contactpoints.yaml grafana:/etc/grafana/provisioning/alerting/contactpoints.yaml
docker cp /tmp/grafana-alerting/policies.yaml grafana:/etc/grafana/provisioning/alerting/policies.yaml
docker restart grafana
```

## Permanent Server Setup Recommendation

The current `docker cp` workflow works, but it is not ideal long term because files copied into a container can disappear when the container is recreated.

Recommended improvement:

Mount provisioning from the host:

```bash
-v /opt/grafana/provisioning:/etc/grafana/provisioning
```

Then store files permanently on the server:

```text
/opt/grafana/provisioning/alerting/rules.yaml
/opt/grafana/provisioning/alerting/contactpoints.yaml
/opt/grafana/provisioning/alerting/policies.yaml
/opt/grafana/provisioning/datasources/cloudwatch.yaml
/opt/grafana/provisioning/dashboards/aws-infrastructure.yaml
```

Restart Grafana after updates:

```bash
docker restart grafana
```

## Importing Dashboards

Dashboards are JSON files under:

```text
grafana/dashboards/aws
```

They can be imported through the Grafana UI:

1. Go to Grafana.
2. Open `Dashboards`.
3. Choose `New` or `Import`.
4. Upload or paste the JSON file.
5. Select the CloudWatch datasource if prompted.
6. Save the dashboard.

Dashboard files:

```text
aws-overview.json
rds-overview.json
ec2-overview.json
ecs-overview.json
```

## Importing Alert Rules

Grafana's alert rules screen has an `Export rules` button, but it does not provide a direct YAML upload/import button.

To import alert rules:

1. Put YAML files in the container path:

```text
/etc/grafana/provisioning/alerting
```

2. Restart Grafana:

```bash
docker restart grafana
```

3. Verify in Grafana:

```text
Alerting -> Alert rules
```

Provisioned rules appear with a `Provisioned` label.

## Editing Rules

Provisioned rules should be edited in YAML files, not in the Grafana UI.

Workflow:

1. Edit `grafana/alerting/rules.yaml`.
2. Validate YAML.
3. Copy file to the container.
4. Restart Grafana.
5. Verify rule state in the UI.
6. Confirm Teams notification if applicable.

Validate locally:

```bash
ruby -e 'require "yaml"; YAML.load_file("grafana/alerting/rules.yaml"); puts "OK"'
```

## Common Troubleshooting

### Alert error: `data source not found`

Cause:

The alert rule references a datasource UID that does not exist on the server.

Fix:

1. Open Grafana datasource settings.
2. Confirm the CloudWatch datasource UID.
3. Update `datasourceUid` and `model.datasource.uid` in `rules.yaml`.
4. Restart Grafana.

Current expected UID:

```text
de3t7rxv00we8c
```

### Teams error: `webhook failed validation`

Cause:

The Teams webhook URL is invalid, expired, placeholder, or incompatible with the Grafana Teams contact point.

Fix:

1. Check `contactpoints.yaml`.
2. Confirm `settings.url` is the real Teams webhook/workflow URL.
3. Restart Grafana.
4. Check logs.

Command:

```bash
docker logs grafana --tail 100 | grep -i teams
```

### Alert stays in `Pending`

Cause:

The threshold is breached, but not for the full pending period.

Example:

If `for: 5m`, Grafana waits for 5 continuous minutes of breach before changing from `Pending` to `Firing`.

Fix:

Wait for the pending period or reduce the `for` value in `rules.yaml`.

### Teams card shows raw values like `B=117.2, C=1`

This is normal Grafana alert expression output.

Interpretation:

- `B` is the reduced metric value.
- `C` is the threshold check.
- `C=1` means breached.
- `C=0` means normal.

### Teams card is blank or too minimal

Cause:

The Teams workflow may not render the Grafana Teams payload exactly like a classic incoming webhook.

Fix:

Update `contactpoints.yaml` message template to include top-level fields before per-alert loops. Redeploy the contact point file and restart Grafana.

### Alert notification links show `localhost:3000`

Cause:

Grafana `root_url` is not set.

Fix:

Set:

```text
domain = grafana.bhiveworkspace.io
root_url = https://grafana.bhiveworkspace.io/
```

Then restart Grafana.

### Dashboard panels show no data

Common causes:

- Wrong AWS region.
- Empty dashboard variables.
- CloudWatch metric not available.
- ECS Container Insights not enabled for Container Insights panels.

Fix:

- Confirm region is `ap-south-1`.
- Confirm CloudWatch datasource works in Explore.
- Use a larger time range such as `Last 1 hour` or `Last 6 hours`.
- Check whether the metric exists in AWS CloudWatch.

## Validation Checklist

After changes, verify the following:

- Grafana container is running.
- CloudWatch datasource test succeeds.
- AWS Infrastructure Overview dashboard loads.
- RDS dashboard shows RDS metrics.
- EC2 dashboard shows EC2 metrics.
- ECS dashboard shows ECS metrics.
- Alert rules appear under `Alerting -> Alert rules`.
- Provisioned rules show the `Provisioned` label.
- No alert rules show `data source not found`.
- Teams contact point sends a test message.
- Teams alert card includes summary, description, values, source link, and silence link.
- Alert links use `https://grafana.bhiveworkspace.io`.

## Operational Runbook

### High RDS Connections

Alert:

```text
RDS High Connections
```

Meaning:

An RDS resource has database connections above the configured threshold.

Current threshold:

```text
DatabaseConnections > 80 for 5 minutes
```

Initial checks:

1. Open the alert source link.
2. Identify the RDS resource from `Series`.
3. Check the RDS dashboard.
4. Check active application deployments or connection pool changes.
5. Verify whether the DB is near max connections for its instance class.

Possible actions:

- Check application connection pools.
- Check long-running transactions.
- Check stuck jobs or traffic spikes.
- Scale RDS or tune DB connection settings if needed.

### EC2 Health Check Failed

Alert:

```text
EC2 Instance Health Check Failed
```

Meaning:

An EC2 instance has a failed instance health check.

Initial checks:

1. Identify `InstanceId`.
2. Open EC2 console.
3. Check instance system and instance status checks.
4. Check CPU, memory, disk, and application logs.
5. Confirm whether the instance is behind a load balancer and whether traffic is impacted.

Possible actions:

- Reboot instance if appropriate.
- Replace unhealthy instance in Auto Scaling Group.
- Check disk saturation.
- Check network/security group issues.
- Check application process health.

### EC2 High CPU

Alert:

```text
EC2 High CPU
```

Meaning:

An EC2 instance has sustained high CPU.

Initial checks:

1. Identify `InstanceId`.
2. Check EC2 dashboard CPU panel.
3. Check application logs and process usage.
4. Check recent traffic or batch jobs.

Possible actions:

- Restart runaway processes if safe.
- Scale out application workers.
- Increase instance size if this is expected load.
- Review autoscaling policy.

### ECS High CPU

Alert:

```text
ECS Service High CPU
```

Meaning:

An ECS service has sustained high CPU utilization.

Initial checks:

1. Identify ECS service labels in Grafana.
2. Check ECS dashboard.
3. Check desired/running task count.
4. Check recent deployments.
5. Check service logs.

Possible actions:

- Scale ECS service task count.
- Increase CPU allocation.
- Roll back problematic deployment.
- Investigate request or job spikes.

### ECS High Memory

Alert:

```text
ECS Service High Memory
```

Meaning:

An ECS service has sustained high memory utilization.

Initial checks:

1. Identify ECS service labels in Grafana.
2. Check ECS dashboard memory panel.
3. Check service logs for memory pressure or OOM events.
4. Check recent deployments.

Possible actions:

- Increase task memory limit.
- Scale task count.
- Investigate memory leaks.
- Roll back problematic deployment.

## Maintenance Notes

- Keep datasource UID consistent across dashboards and alert rules.
- Keep AWS region consistent as `ap-south-1` unless infrastructure moves.
- Do not expose webhook URLs.
- Prefer file-based alert changes for provisioned rules.
- Restart Grafana after provisioning file changes.
- Keep dashboard JSON files versioned with alert YAML files.
- Re-test Teams notification after changing contact point templates.
- Review thresholds periodically as traffic patterns change.

## Future Improvements

Recommended improvements:

- Mount provisioning directories from the host instead of using `docker cp`.
- Add environment labels such as `production`, `staging`, or `shared`.
- Add owner/team labels for routing and filtering.
- Add runbook URLs to alert annotations.
- Split production and staging alert thresholds if needed.
- Add ALB/NLB, EBS, Lambda, and SQS dashboards if those services are operationally important.
- Add CloudWatch billing/cost visibility if required.
