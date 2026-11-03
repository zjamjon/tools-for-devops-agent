# Changelog

## [1.0.0] - 2026-11-03
- Initial version: read-only CloudWatch cost review that attributes spend by Cost Explorer usage type and checks log retention, heavy-ingestion and vended log groups, Essentials (formerly Infrequent Access) log class candidates, debug-level logging, custom metric cardinality, unused alarms and dashboards, Logs Insights scan spend, and API polling, metric stream and Contributor Insights spend.
- Review changes before merge: decision-first Executive Summary (monthly and annualized spend and savings, decisions needed, owners, S/M/L effort); numeric severity thresholds; a 3-month trend collected on every run; effective rates that exclude free-tier `Global-` lines; a retention estimate from ingestion history; `GetMetricData` callers named only when CloudTrail data events record them; Container Insights enhanced observability and EC2 detailed monitoring checks; complete Essentials eligibility, including vended destinations and field indexes; billed metric counts from Cost Explorer; added usage types; evals built on a synthetic context file.
- Scope limited to a single account per run.
