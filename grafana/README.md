Grafana CloudWatch provisioning added here:

- `datasources/cloudwatch.yaml`: provisions a CloudWatch datasource with UID `de3t7rxv00we8c` and default region `ap-south-1`
- `dashboards/providers/aws-infrastructure.yaml`: auto-loads dashboards from `dashboards/aws/`
- `dashboards/aws/*.json`: importable dashboards for RDS, EC2, and ECS
- `dashboards/aws/aws-overview.json`: central overview dashboard for the key AWS health signals
- `alerting/contactpoints.yaml`: Teams contact point
- `alerting/policies.yaml`: default notification routing
- `alerting/rules.yaml`: baseline AWS alert rules

Before starting Grafana:

1. Replace `https://REPLACE-WITH-YOUR-TEAMS-WEBHOOK` in `alerting/contactpoints.yaml` with your Teams webhook URL.
2. Adjust `datasources/cloudwatch.yaml`, dashboard target regions, and `alerting/rules.yaml` if your AWS region changes from `ap-south-1`.
3. Tune the thresholds in `alerting/rules.yaml`, especially `RDS High Connections`, to match your instance sizes and expected load.

The central overview and RDS dashboards intentionally use fixed `ap-south-1` CloudWatch `SEARCH(...)` queries with wildcard dimensions. This avoids blank panels when Grafana dashboard URL variables are empty or stale.

The alert rules reference the existing server CloudWatch datasource UID `de3t7rxv00we8c`. If the server datasource UID changes, update both `alerting/rules.yaml` and `datasources/cloudwatch.yaml`.

The ECS dashboard uses `ECS/ContainerInsights` for task count and network traffic panels. If Container Insights is not enabled, those specific panels will stay empty; CPU and memory panels will still work from `AWS/ECS`.
