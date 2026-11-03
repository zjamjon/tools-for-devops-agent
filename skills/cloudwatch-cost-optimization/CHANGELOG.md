# Changelog

## [1.0.0] - 2026-11-03
- Initial version: read-only CloudWatch cost review that attributes spend by Cost Explorer usage type and checks log retention, heavy-ingestion and vended log groups, Essentials (formerly Infrequent Access) log class candidates, debug-level logging, custom metric cardinality, unused alarms and dashboards, Logs Insights scan spend, and API polling, metric stream and Contributor Insights spend.
- Report opens with a one-screen Leadership Summary (spend trend, share of the AWS bill, savings range, posture KPIs, top 3 actions, risk of inaction) above the engineering detail, with a short form for accounts whose spend is within the free tier.
- Workload scope for applications spread across several accounts (account list, Cost Category, or cost allocation tag), with a per-account view, alongside account and organization scope.
- §4.10 orphaned log groups, alarms, and dashboards.
- Step 2 pulls a 6-month trend and total AWS spend; stricter rule against quoting list prices; full `ListFunctions` pagination; known quirks for the shared Logs free tier, tiny usage missing from Cost Explorer, `EU-` vs `EUW2-` prefixes, the two-key `GroupBy` limit, and cross-account observability billing.
