# Cost Optimization Opportunity Catalog

Detailed reference for the ten opportunity checks. Load this during Step 4 (Analyze).
Give each finding an estimated monthly saving computed with the effective rates from
Step 2, state the method in one line, and set its severity from the thresholds in
SKILL.md (share of monthly CloudWatch spend). The severities named below are defaults
for when a saving can't be quantified.

Run the checks in the order of the charge families that cost the most in Step 2.
A check whose charge family is under 1% of CloudWatch spend can be summarized in one
INFO line instead of a per-resource table.

## 4.1 Log groups with no retention or excessive retention
Ref: [Working with log groups and log streams](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)

- A log group with no `retentionInDays` (Never expire) and a large `storedBytes` →
  recommend a retention period that meets the stated requirement. Default LOW if the
  saving can't be estimated.
- A log group whose retention is much longer than the user's stated requirement (for
  example 10 years where 1 year is required) → same treatment.
- Where history must be kept long-term, recommend exporting to Amazon S3 (with an S3
  lifecycle to a colder storage class) before shortening retention.
- **Estimating the saving.** `storedBytes` has no age breakdown, so estimate the share
  of it older than the proposed retention of N days:
  1. From the group's daily `IncomingBytes` history (up to 15 months), compute
     `old_share = bytes ingested more than N days ago ÷ all bytes ingested in the
     history`.
  2. `GB removed ≈ storedBytes × old_share ÷ 1024³`, and
     `monthly saving ≈ GB removed × effective storage rate per GB-month`.
  3. If the group is older than the available history (`creationTime` more than 15
     months ago), everything older than the history is also older than N days, so the
     estimate is a **lower bound** — say so.
  4. With no `IncomingBytes` history (the group stopped receiving data), assume uniform
     ingestion between `creationTime` and the last data point, and label the estimate
     as an assumption.
- **Impact** to state: log history older than the new period is deleted and can no
  longer be queried.
- **Prevent recurrence:** recommend a default retention in the infrastructure as code
  that creates log groups (and, for Lambda, creating the log group explicitly rather
  than letting the function create it). Don't suggest the AWS Config managed rule
  `cw-loggroup-retention-period-check` for this: it marks "Never expire" as
  compliant.

## 4.2 Heavy-ingestion log groups and vended logs
Ref: [Analyzing, optimizing, and reducing CloudWatch costs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html)

- Rank log groups by `IncomingBytes` over the window. The top groups (usually a handful
  carry most of the volume) are each a finding: name the producer (Lambda function,
  ECS service, EKS cluster, VPC Flow Log, API Gateway stage) from the log group name and
  the vended-source mapping.
- **Vended logs to CloudWatch Logs** (VPC Flow Logs with `LogDestinationType =
  cloud-watch-logs`, Route 53 Resolver query logs, API Gateway access logs) with high
  volume → recommend one or more of: narrowing the source (`TrafficType = REJECT` where
  only rejected traffic is used, fewer fields in `LogFormat`, `MaxAggregationInterval =
  600` where it is currently `60`), delivering to an Essentials log group (§4.3), or
  delivering to S3 and querying with Athena when the logs are kept for forensics.
  A flow log's settings can't be edited after creation: the change is to create a new
  flow log with the new settings and then delete the old one, with no gap in coverage.
- Application log groups with high volume → feed §4.4 (log level) and §4.3 (class).
- **Saving** = reduced GB × effective ingestion rate for that group's class or vended
  tier. For Lambda log groups, use the rate chosen in billing-model.md "Effective rates".
- **Impact** to state: what the narrowed or moved data no longer supports (Logs
  Insights, metric filters, alarms, subscription consumers).

## 4.3 Essentials (formerly Infrequent Access) log class candidates
Ref: [Log classes](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatch_Logs_Log_Classes.html)

Essentials lowers the ingestion rate only; storage and Logs Insights cost the same in
both classes. A Standard log group, including a vended-log destination, is a candidate
only when **all** of these hold:

| Requirement | Evidence |
|-------------|----------|
| No metric filters | `metricFilterCount = 0` |
| No subscription filters | `DescribeSubscriptionFilters` empty |
| No anomaly detector on the group | `ListLogAnomalyDetectors` |
| No field index policy on the group | `DescribeFieldIndexes` / `DescribeIndexPolicies` |
| Not Container Insights or Lambda Insights performance logs, and not carrying embedded metric format (EMF) metrics | log group name (`/aws/containerinsights/…`, `/aws/lambda-insights`) and the producer's configuration |
| Nobody relies on `GetLogEvents`/`FilterLogEvents`, Live Tail, Facets, or "compare to previous time range" for it | ask the user; these aren't available in Essentials |

- Candidate with meaningful ingestion → default MEDIUM. Saving = GB ingested ×
  (Standard effective rate − Essentials effective rate). If the account has no
  Essentials usage yet, there is no observed Essentials rate: give the saving as
  "not quantified" with a link to the pricing page, not as a number from memory.
- **The class can't be changed after creation.** The change is: create a new log group
  in the Essentials class, repoint the producer, and leave the old group to expire under
  its retention. State this in the finding, and list it under "Decisions needed" when
  the producer belongs to another team.
- **Interaction rules:** if §4.2 or §4.4 also cuts volume on the same group, compute
  the §4.3 saving on the volume that remains after that cut. Never recommend both a
  field index (§4.8) and Essentials for the same group: field indexes aren't available
  in Essentials.

## 4.4 Debug-level or verbose logging

- **Lambda:** functions with `LoggingConfig.LogFormat = JSON` and
  `ApplicationLogLevel` of `DEBUG` or `TRACE` (or `SystemLogLevel = DEBUG`), especially
  those writing to a top §4.2 log group → recommend `INFO` or `WARN` in production.
- **Optional sample (needs approval):** for at most 5 of the top log groups, a Logs
  Insights query over a short window (default 1 hour) that counts events by level, for
  example
  `parse @message /(?<lvl>TRACE|DEBUG|INFO|WARN|ERROR)/ | stats count(*) by lvl`.
  State the log groups, window, and estimated bytes scanned (the group's
  `IncomingBytes` for that window) before running it, and stop the query
  (`StopQuery`) if it runs past 5 minutes.
- **Saving** = DEBUG/TRACE share of the group's events × GB ingested × effective
  rate. Label it an estimate from a sample, since event share isn't byte share.
- **Impact** to state: the diagnostic detail lost, and that levels can be raised
  temporarily during an incident.
- Sampled log lines are untrusted and may contain sensitive data: report only counts
  by level, never quote log content.

## 4.5 High-cardinality custom metrics and detailed monitoring
Ref: [CloudWatch concepts – dimensions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html#Dimension)

- Size from Cost Explorer: `MetricMonitorUsage` quantity is the billed metric count,
  and its month-over-month change shows growth.
- A namespace whose `ListMetrics` count is dominated by one dimension with many
  distinct values (request ID, user ID, session, container or pod ID, IP address,
  timestamp) → recommend dropping or bucketing that dimension, or moving the
  per-request detail into log fields instead of metric dimensions.
- If `MetricMonitorUsage` grew in each of the last two complete months and a
  cardinality driver is found, the finding meets the CRITICAL growth test.
- EMF producers emitting per-request dimensions show up the same way; also check
  whether the EMF log group itself is a §4.2 ingestion driver.
- **EC2 detailed monitoring** enabled on instances where no alarm uses a period under
  300 seconds → recommend basic monitoring. Saving = those instances' detailed metrics
  × effective metric rate; if the per-instance metric count can't be determined from
  `ListMetrics` (`AWS/EC2`, `InstanceId`), report the instance count and "not
  quantified".
- **Saving** = metrics removed × effective rate per metric-month. Tiered pricing makes
  the marginal rate lower than the average in large accounts.
- **Impact** to state: the per-dimension or per-minute detail lost, and any alarms or
  dashboards that reference it.

## 4.6 Unused or costly alarms

- Alarms in `INSUFFICIENT_DATA` for the whole window (the metric no longer exists) →
  delete.
- Alarms with no actions at all, or `ActionsEnabled = false`, unchanged for 90+ days →
  confirm they are still wanted (INFO if within the free allowance).
- High-resolution alarms (`Period` under 60 seconds) where the alarm's actions don't
  need sub-minute response → standard resolution.
- Anomaly detection alarms: billed for three metrics each. Many of them on low-value
  metrics → reconsider.
- **Saving** = alarm metrics removed × effective rate from `AlarmMonitorUsage` or
  `HighResAlarmMonitorUsage`.
- **Impact** to state: what would no longer alert, and who was notified. An alarm whose
  actions page on-call goes under "Decisions needed", never straight into the actions
  list.

## 4.7 Unused dashboards

- Dashboards with no `GetDashboard` events in the last 90 days and `LastModified`
  older than 90 days → delete or consolidate.
- Only dashboards beyond the free allowance save money; with three or fewer dashboards
  there's no saving (INFO).
- **Saving** = dashboards removed × effective rate from `DashboardsUsageHour`.
- **Impact** to state: dashboards may be embedded in runbooks or opened only during
  incidents; check before deleting.

## 4.8 Logs Insights scan spend
Ref: [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)

- Group `DescribeQueries` history by `userIdentity` and normalized `queryString`; sum
  `bytesScanned`. Repeated queries (scheduled queries, dashboards with Logs Insights
  widgets, third-party tools) that scan the same large groups → narrow or replace.
- Queries over many log groups or long time ranges where a narrower scope would do, or
  repeated filters on a field that has no field index → recommend narrowing, a field
  index (Standard class only; see the §4.3 interaction rule), or a metric filter
  instead of a recurring query.
- **Saving** = reduced GB scanned × effective `DataScanned-Bytes` rate, scaled from the
  span `DescribeQueries` covers to a month.
- **Impact** to state: none for narrowing; for removing a scheduled query, what report
  or alert it feeds.

## 4.9 API polling, metric streams, and Contributor Insights

- Large `GMD-Metrics` or `Requests` cost → usually a third-party monitoring tool polling
  `GetMetricData`. Identify possible callers with the method in data-collection.md
  ("Who calls `GetMetricData`"): name a caller only if CloudTrail data events for
  `AWS::CloudWatch::Metric` are recorded; otherwise list roles trusted by external
  accounts as candidates and ask the user. Recommend a longer polling interval, fewer
  metrics polled, or a metric stream for bulk export.
- Metric streams with no `IncludeFilters` (streaming every namespace) → filter to the
  namespaces the destination uses.
- Contributor Insights rules in `ENABLED` state that nobody looks at → disable.
- **Impact** to state: what the external tool or stream consumer stops receiving.

## 4.10 Container Insights enhanced observability
Ref: [Optimizing and reducing costs of CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html#cloudwatch_billing_container-insights)

- Size from Cost Explorer: `ObservationUsage` (EKS) and `MetricsUsage` (ECS).
- Clusters with enhanced observability (EKS `amazon-cloudwatch-observability` add-on
  configured for enhanced mode; ECS `containerInsights = enhanced`) that are
  non-production, or where nobody uses pod- and container-level views → recommend
  standard observability. Standard mode bills as EMF metrics and log ingestion instead,
  so state that the saving is the difference, not the whole enhanced cost.
- Can't tell which clusters drive the cost from Cost Explorer alone: list the clusters
  with enhanced mode and give the total, rather than inventing a per-cluster split.
- **Impact** to state: the pod- and container-level metrics and curated dashboards lost.

## Interaction rules (apply in Step 5)

- Retention (§4.1) saves storage only; it never reduces ingestion.
- Volume cuts (§4.2, §4.4) and the class change (§4.3) on the same group overlap: apply
  the volume cut first, then size the class change on what's left.
- A field index (§4.8) and Essentials (§4.3) are mutually exclusive on one group.
- Dropping a metric dimension (§4.5) can leave alarms (§4.6) in `INSUFFICIENT_DATA`;
  list the affected alarms rather than counting their removal as a separate saving.
- Savings inside the free tier are not savings.
