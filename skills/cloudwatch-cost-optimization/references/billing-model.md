# CloudWatch Billing Model

Reference for how each CloudWatch charge is computed and which Cost Explorer usage
types it appears under. Load this in Step 2 to classify spend and in Step 4 to decide
what a recommendation actually saves.

This file explains the structure of charges only. It deliberately holds no unit
prices: derive effective rates from Cost Explorer (cost ÷ usage quantity) for the
account and Region being analyzed, and link to
[Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/) when the user
wants list prices.

## Charges and what reduces them

| Charge family | Billed on | Lever that reduces it | Does NOT reduce it |
|---------------|-----------|-----------------------|--------------------|
| Logs ingestion (Standard) | GB of uncompressed data ingested | Log less (level, sampling, filtering at source), move to Essentials | Shorter retention |
| Logs ingestion (Essentials, formerly Infrequent Access) | GB ingested, at a lower rate than Standard | Log less | Retention |
| Vended logs (VPC Flow Logs, Route 53 Resolver, API Gateway access logs, etc.) | GB ingested, volume-tiered per month, at a Standard or Essentials rate depending on the destination log group's class | Narrow the source (filter, fields, aggregation interval), deliver to an Essentials log group, or deliver to S3 or Firehose instead | Retention |
| Logs storage (archive) | GB-month of **compressed** stored data | Set or shorten retention; export to S3 for long-term holding | Log level changes (only reduce future accrual) |
| Logs storage (Archive Instant Access) | GB-month in the Archive Instant Access tier | Retention | — |
| Logs Insights | GB scanned per query, same in both classes | Narrower time ranges, fewer log groups per query, field indexes (Standard class only), fewer scheduled or dashboard queries | Retention (unless queries scanned the deleted range) |
| Custom metrics (incl. EMF and EC2 detailed monitoring) | Per metric-month, prorated hourly, volume-tiered | Drop high-cardinality dimensions, stop publishing unused metrics, aggregate before publishing, disable unused detailed monitoring | Deleting alarms |
| Container Insights, enhanced observability | Per observation (EKS) or per metric (ECS) | Standard observability where pod- and container-level detail isn't needed | Log retention |
| Alarms | Per metric per alarm-month; high-resolution and anomaly detection cost more; composite is a flat fee per alarm | Delete idle alarms, use standard resolution where sub-minute alerting isn't needed | — |
| Dashboards | Per dashboard-month beyond the free allowance | Delete or consolidate unused dashboards | — |
| API requests | `GetMetricData` per metric requested; other APIs per 1,000 requests beyond the free allowance | Lower third-party polling frequency, use metric streams for bulk export | — |
| Metric streams | Per 1,000 metric updates (plus Firehose charges) | Filter namespaces and metrics in the stream | — |
| Contributor Insights | Per rule-month and per million log events matched | Remove unused rules | — |
| Telemetry centralization | GB centralized beyond the free allowance | Centralize only the data the central account uses | — |
| Live Tail | Per session-minute beyond the free allowance | Close idle sessions | — |

Anomaly detection alarms are billed for the evaluated metric **plus** its upper and
lower band metrics. Metric math and Metrics Insights alarms are billed per metric
analyzed.

## Free tier

The free tier is account-wide per month. In Cost Explorer it shows up as usage types
with a `Global-` prefix at $0. Treat these values as reference only and confirm them
on the [pricing page](https://aws.amazon.com/cloudwatch/pricing/) before quoting:

| Item | Free allowance |
|------|----------------|
| Logs | 5 GB of data (ingestion, archive storage, and Logs Insights scanned, combined) |
| Metrics | 10 custom or detailed monitoring metrics, plus 1 million API requests |
| Alarms | 10 standard-resolution alarm metrics |
| Dashboards | 3 custom dashboards of up to 50 metrics each |
| Contributor Insights | 1 rule, and the first 1 million log events matching it |
| Live Tail | 1,800 minutes |

Never present a saving that falls entirely inside the free allowance as a saving.

## Classifying Cost Explorer usage types

Group `ce.GetCostAndUsage` (`SERVICE = AmazonCloudWatch`) by `USAGE_TYPE`. The full
string is `<Region>-<Feature>` or `<Region>-CW:<Feature>` (for example `EU-CW:GMD-Metrics`
for Europe (Ireland)), and free-tier usage uses `Global-` as the Region. Strip any
prefix ending in `-` before the first feature token (the prefix may be absent) and
match on the remainder:

| Usage type suffix | Charge family | Check |
|-------------------|---------------|-------|
| `DataProcessing-Bytes` | Logs ingestion (Standard) | §4.2, §4.3, §4.4 |
| `DataProcessingIA-Bytes` | Logs ingestion (Essentials / Infrequent Access) | §4.2 |
| `DataProcessingOtel*-Bytes` | Logs ingestion via OpenTelemetry | §4.2 |
| `VendedLog-Bytes*` | Vended logs to Standard log groups | §4.2, §4.3 |
| `VendedLogIA-Bytes*` | Vended logs to Essentials log groups | §4.2 |
| `S3-Egress-*`, `FH-Egress-*` | Vended logs delivered to S3 or Firehose | §4.2 (INFO) |
| `TimedStorage-ByteHrs`, `TimedStorage-IA-ByteHrs` | Logs storage | §4.1 |
| `TimedStorage-AIA-ByteHrs` | Logs storage, Archive Instant Access | §4.1 |
| `DataScanned-Bytes` | Logs Insights scanned | §4.8 |
| `DataProtection-Bytes` | Log data protection (masking) scanned | §4.2 (INFO) |
| `CW:MetricMonitorUsage` | Custom, EMF, and detailed monitoring metrics | §4.5 |
| `CW:ObservationUsage` | Container Insights enhanced observability (EKS) | §4.10 |
| `CW:MetricsUsage` | Container Insights enhanced observability (ECS) | §4.10 |
| `CW:OTEL:*` | OpenTelemetry metrics | §4.5 |
| `CW:AlarmMonitorUsage`, `CW:HighResAlarmMonitorUsage`, `CW:CompositeAlarmMonitorUsage`, `CW:LogAlarmMonitorUsage`, `CW:MetricInsightAlarmUsage`, `CW:PromQLAlarmMonitorUsage` | Alarms | §4.6 |
| `DashboardsUsageHour`, `DashboardsUsageHour-Basic` | Dashboards | §4.7 |
| `CW:GMD-Metrics` | `GetMetricData` metrics requested | §4.9 |
| `CW:GMWI-Metrics` | `GetMetricWidgetImage` metrics requested | §4.9 |
| `CW:GIRR-Metrics` | `GetInsightRuleReport` (Contributor Insights) | §4.9 |
| `CW:Requests` | Other API requests | §4.9 |
| `CW:MetricStreamUsage` | Metric streams | §4.9 |
| `CW:ContributorInsightRules`, `CW:ContributorInsightEvents` | Contributor Insights | §4.9 |
| `CW:PromQL:SamplesScanned` | PromQL queries | §4.9 |
| `CentralizedBytes` (`CentralizedBytesFree` is the free allowance) | Telemetry centralization | INFO: centralized copies held in this account, billed again at the source |
| `Logs-LiveTail` | Live Tail | INFO |

Other CloudWatch features also bill under `AmazonCloudWatch` (Synthetics
`CW:Canary-runs`, RUM, Application Signals, Database Insights, Internet Monitor,
Network Flow Monitor, X-Ray spans indexed). This skill doesn't analyze them: list
them in the cost attribution table as "Not analyzed by this skill" with their cost.

These strings come from the
[CloudWatch usage type table](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html)
and the AWS Price List, and change over time. Put any usage type that matches none of
them in an "Other" family, show it in the report, and don't guess what it means.

## Effective rates

Effective rate = `UnblendedCost` ÷ `UsageQuantity` for a family over the window,
computed from the **Region-prefixed lines only**. Exclude `Global-` lines: they carry
free-tier usage at $0 and would understate every rate.

- When usage crosses a pricing tier (vended logs, custom metrics), the effective rate
  is a blend; say so when sizing a large reduction, because the marginal saving may be
  at a lower tier rate.
- **Lambda logs:** check which family a Lambda log group's bytes bill under before
  sizing a §4.4 saving. Sum `IncomingBytes` for the Region's `/aws/lambda/*` log groups
  over the window and compare it with the `UsageQuantity` of `DataProcessing-Bytes` and
  of `VendedLog-Bytes` for that Region. Use the rate of the family whose quantity
  accounts for those bytes. If neither clearly does, use the lower of the two rates and
  say the saving may be understated.
- With no Region-prefixed line for a family (all usage in the free tier), the rate is
  "rate not observed".

## Converting signals to billing units

- `IncomingBytes` (per log group) → GB ingested: divide the window sum by 1024³.
- `storedBytes` (from `DescribeLogGroups`) → GB-month of storage: divide by 1024³.
  It is a point-in-time compressed size, not a monthly total, and has no breakdown by
  age (see §4.1 for how to estimate what a retention change removes).
- `bytesScanned` (from `DescribeQueries`) → GB scanned: divide by 1024³.
- Billed custom metrics: the `MetricMonitorUsage` `UsageQuantity` from Cost Explorer.
  `ListMetrics` counts are for finding the driving namespace and dimension, not for
  billing.
