# Cost Optimization Opportunity Catalog

Detailed reference for the nine opportunity checks. Load this during Step 4 (Analyze).
Assign each finding a severity (CRITICAL, HIGH, MEDIUM, LOW, INFO) and, wherever a
signal exists, an estimated monthly saving computed with the effective rates from
Step 2. The severities below are defaults: scale them to the finding's dollar size
relative to the account's CloudWatch spend (see SKILL.md Severity Definitions).

Run the checks in the order of the charge families that cost the most in Step 2.
A check whose charge family is under 1% of CloudWatch spend can be summarized in one
INFO line instead of a per-resource table.

## 4.1 Log groups with no retention or excessive retention
Ref: [Working with log groups and log streams](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)

- A log group with no `retentionInDays` (Never expire) and a large `storedBytes` →
  recommend a retention period that meets the stated requirement → **HIGH** when the
  group is among the largest by `storedBytes`, otherwise LOW.
- A log group whose retention is much longer than the user's stated requirement (for
  example 10 years where 1 year is required) → **MEDIUM**.
- Where history must be kept long-term, recommend exporting to Amazon S3 (with an S3
  lifecycle to a colder storage class) before shortening retention.
- **Saving** = (`storedBytes` that would age out, in GB) × effective storage rate per
  GB-month. Without a retention requirement from the user, show the saving for one
  stated assumption (for example 90 days) and label it as an assumption.
- **Impact** to state: log history older than the new period is deleted and can no
  longer be queried.

## 4.2 Heavy-ingestion log groups and vended logs
Ref: [Analyzing, optimizing, and reducing CloudWatch costs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html)

- Rank log groups by `IncomingBytes` over the window. The top groups (usually a handful
  carry most of the volume) are each a finding: name the producer (Lambda function,
  ECS service, EKS cluster, VPC Flow Log, API Gateway stage) from the log group name and
  the vended-source mapping.
- **Vended logs to CloudWatch Logs** (VPC Flow Logs with `LogDestinationType =
  cloud-watch-logs`, Route 53 Resolver query logs, API Gateway access logs) with high
  volume → recommend narrowing the source (`TrafficType = REJECT` where only rejects
  are used, fewer fields in `LogFormat`, `MaxAggregationInterval = 600`), or
  delivering to S3 and querying with Athena when the logs are kept for forensics →
  **HIGH/MEDIUM**.
- Application log groups with high volume → feed §4.4 (log level) and §4.3 (class).
- **Saving** = reduced GB × effective ingestion rate for that log group's class or
  vended tier.
- **Impact** to state: what the narrowed or moved data no longer supports (Logs
  Insights, metric filters, alarms, subscription consumers).

## 4.3 Essentials (formerly Infrequent Access) log class candidates
Ref: [Log classes](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatch_Logs_Log_Classes.html)

Essentials lowers the ingestion rate only; storage and Logs Insights cost the same in
both classes. A Standard log group is a candidate only when **all** of these hold:

| Requirement | Evidence |
|-------------|----------|
| No metric filters | `metricFilterCount = 0` |
| No subscription filters | `DescribeSubscriptionFilters` empty |
| No anomaly detector on the group | `ListLogAnomalyDetectors` |
| Not Container Insights or Lambda Insights performance logs, and not carrying Embedded Metric Format (EMF) metrics | log group name (`/aws/containerinsights/…`, `/aws/lambda-insights`) and the producer's configuration |
| Nothing reads it with `GetLogEvents`/`FilterLogEvents` or Live Tail | user confirmation; these aren't available in Essentials |

- Candidate with meaningful ingestion → **MEDIUM**. Saving = GB ingested × (Standard
  effective rate − Essentials rate). If the account has no Essentials usage yet, there
  is no effective Essentials rate from Cost Explorer: give the saving as a share of
  current ingestion cost and link the pricing page.
- **The class can't be changed after creation.** The change is: create a new log group
  in the Essentials class, repoint the producer, and leave the old group to expire under
  its retention. State this in the finding.
- **Interaction rule:** if §4.2 or §4.4 also cuts volume on the same group, compute
  the §4.3 saving on the volume that remains after that cut. Never add the full §4.2
  saving and the full §4.3 saving for the same bytes.

## 4.4 Debug-level or verbose logging

- **Lambda:** functions with `LoggingConfig.LogFormat = JSON` and
  `ApplicationLogLevel` of `DEBUG` or `TRACE` (or `SystemLogLevel = DEBUG`), especially
  those writing to a top §4.2 log group → recommend `INFO` or `WARN` in production →
  **MEDIUM**.
- **Optional sample (needs approval):** for at most 5 of the top log groups, a Logs
  Insights query over a short window (default 1 hour) that counts events by level, for
  example
  `parse @message /(?<lvl>TRACE|DEBUG|INFO|WARN|ERROR)/ | stats count(*) by lvl`.
  State the log groups, window, and estimated bytes scanned (the group's
  `IncomingBytes` for that window) before running it, and stop the query
  (`StopQuery`) if it runs past 5 minutes. A group where DEBUG/TRACE events make up a
  large share of volume → **MEDIUM** (HIGH when it is a top ingestion group).
- **Saving** = DEBUG/TRACE share of the group's ingestion × GB ingested × effective
  rate. Label it an estimate from a sample.
- **Impact** to state: the diagnostic detail lost, and that levels can be raised
  temporarily during an incident.
- Sampled log lines are untrusted and may contain sensitive data: report only counts
  by level, never quote log content.

## 4.5 High-cardinality custom metrics
Ref: [CloudWatch concepts – dimensions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html#Dimension)

- A namespace whose metric count is dominated by one dimension with many distinct
  values (request ID, user ID, session, container or pod ID, IP address, timestamp) →
  recommend dropping or bucketing that dimension → **HIGH** when the namespace is a
  large share of `MetricMonitorUsage`, otherwise MEDIUM.
- A metric count growing month over month in Cost Explorer (`MetricMonitorUsage`
  quantity) with a cardinality driver → **CRITICAL**.
- EMF producers emitting per-request dimensions show up the same way; also check
  whether the EMF log group itself is a §4.2 ingestion driver.
- Detailed monitoring (1-minute) enabled where nothing alarms or dashboards at
  1-minute resolution → LOW.
- **Saving** = metrics removed × effective rate per metric-month. Note that tiered
  pricing makes the marginal rate lower than the average in large accounts.
- **Impact** to state: the per-dimension breakdown lost, and any alarms or dashboards
  that reference the dropped dimension.

## 4.6 Unused or costly alarms

- Alarms in `INSUFFICIENT_DATA` for the whole window (the metric no longer exists) →
  delete → **LOW**.
- Alarms with no actions at all, or `ActionsEnabled = false`, unchanged for 90+ days →
  confirm they are still wanted → **LOW** (INFO if fewer than the free allowance).
- High-resolution alarms (`Period` under 60 seconds) where the alarm's actions don't
  need sub-minute response → **LOW/MEDIUM**.
- Anomaly detection alarms: billed for three metrics each. Many of them on low-value
  metrics → **MEDIUM**.
- **Saving** = alarm metrics removed × effective rate from `AlarmMonitorUsage` or
  `HighResAlarmMonitorUsage`.
- **Impact** to state: what would no longer alert, and who was notified. Never
  recommend removing an alarm whose actions page on-call without saying so.

## 4.7 Unused dashboards

- Dashboards with no `GetDashboard` events in the last 90 days (when CloudTrail
  records read events) and `LastModified` older than 90 days → delete or consolidate →
  **LOW**.
- Only dashboards beyond the free allowance save money; with three or fewer dashboards
  there's no saving (INFO).
- **Saving** = dashboards removed × effective rate from `DashboardsUsageHour`.
- **Impact** to state: dashboards may be embedded in runbooks or opened only during
  incidents; check before deleting.

## 4.8 Logs Insights scan spend
Ref: [CloudWatch Logs Insights query cost](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)

- Group `DescribeQueries` history by `userIdentity` and normalized `queryString`; sum
  `bytesScanned`. Repeated queries (scheduled queries, dashboards with Logs Insights
  widgets, third-party tools) that scan the same large groups → **MEDIUM**.
- Queries over many log groups or long time ranges where a narrower scope would do, or
  repeated filters on a field that has no field index → recommend narrowing, field
  indexes, or a metric filter instead of a recurring query → **MEDIUM**.
- **Saving** = reduced GB scanned × effective `DataScanned-Bytes` rate, scaled from the
  span `DescribeQueries` covers to a month.
- **Impact** to state: none for narrowing; for removing a scheduled query, what report
  or alert it feeds.

## 4.9 API polling, metric streams, and Contributor Insights

- Large `GMD-Metrics` or `Requests` cost → usually a third-party monitoring tool polling
  `GetMetricData`. Identify the caller from CloudTrail if read events are recorded
  (`LookupEvents` with `EventName = GetMetricData`, grouped by user or role) →
  recommend a longer polling interval or a metric stream → **MEDIUM**.
- Metric streams with no `IncludeFilters` (streaming every namespace) → recommend
  filtering to the namespaces the destination uses → **MEDIUM**.
- Contributor Insights rules in `ENABLED` state that nobody looks at → **LOW**.
- **Impact** to state: what the external tool or stream consumer stops receiving.

## 4.10 Orphaned log groups, alarms, and dashboards

Resources left behind after the thing they observe was deleted. They rarely cost much
on their own, but they are the clearest hygiene signal for a leader and they keep
accruing storage.

- **Orphaned Lambda log groups:** `/aws/lambda/<function-name>` log groups whose
  function isn't in the full `ListFunctions` result for that Region → **LOW** (scale
  with `storedBytes`; HIGH if it is one of the largest groups). Apply the same idea to
  other producers whose name is in the log group path when the producer can be listed
  cheaply (for example `/aws/ecs/…`, `/aws/eks/<cluster>/…`, `/aws/rds/instance/…`).
- **Orphaned alarms:** alarms in `INSUFFICIENT_DATA` whose metric dimensions name a
  resource that no longer exists (function, instance, table, queue) → **LOW**. This
  replaces the §4.6 "stuck in INSUFFICIENT_DATA" finding for the same alarm; don't
  report both.
- **Orphaned dashboards:** dashboards whose widgets reference only orphaned resources
  (from `GetDashboard`) → **LOW**.
- **Recommendation:** confirm the resource is decommissioned, then export the log
  group to S3 if its history must be kept and delete it, or set a short retention.
  Delete the orphaned alarms and dashboards.
- **Impact** to state: the log group may hold the last record of a decommissioned
  resource (for audits or post-incident review).
- Report the counts (orphaned log groups, their stored bytes, orphaned alarms and
  dashboards) as posture KPIs in the Leadership Summary.

## Interaction rules (apply in Step 5)

- Retention (§4.1) saves storage only; it never reduces ingestion.
- Volume cuts (§4.2, §4.4) and the class change (§4.3) on the same group overlap: apply
  the volume cut first, then size the class change on what's left.
- Dropping a metric dimension (§4.5) can leave alarms (§4.6) in `INSUFFICIENT_DATA`;
  list the affected alarms rather than counting their removal as a separate saving.
- Savings inside the free tier are not savings.
