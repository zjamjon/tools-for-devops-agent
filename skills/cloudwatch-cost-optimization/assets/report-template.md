# CloudWatch Cost Optimization Report Template

Output template for the Step 6 report artifact. Load this when generating the final
report so the structure, section order, and table schemas are consistent.

Artifact naming: `cloudwatch-cost-optimization-<account-id>-<YYYY-MM-DD>.md`
Example: `cloudwatch-cost-optimization-123456789012-2026-11-03.md`

---

### Report Header
```
# Amazon CloudWatch Cost Optimization — <account-id>
Date: <YYYY-MM-DD> | Scope: <regions / organization> | Analysis window: <start> to <end>
```

### Executive Summary
- CloudWatch spend in the window, and the change from the previous equal window
- Estimated total monthly savings (sum of quantified opportunities, after the
  interaction rules) or "not quantified"
- Finding counts by severity
- Top 3 opportunities by estimated saving

### Cost Attribution
| Charge family | Usage types | Cost in window | Share | Effective rate | Change vs previous window |
|---------------|-------------|----------------|-------|----------------|---------------------------|

Include an "Other" row for unmatched usage types and a "Not analyzed by this skill"
row (Synthetics, RUM, Application Signals, Database Insights, and similar).

### Top Log Groups
| Log group | Region | Class | Retention | Ingested (GB, window) | Stored (GB) | Metric / subscription filters | Producer |
|-----------|--------|-------|-----------|-----------------------|-------------|-------------------------------|----------|

### Custom Metrics by Namespace
| Namespace | Region | Metric count | Cardinality driver (dimension) | Distinct values |
|-----------|--------|--------------|--------------------------------|-----------------|

### Cost Optimization Opportunities
| # | Check (§) | Resource | Severity | Current State | Recommendation | Impact | Evidence | Est. Monthly Saving |
|---|-----------|----------|----------|---------------|----------------|--------|----------|---------------------|

Impact and Evidence are mandatory for every row. Impact states, in plain language,
what visibility, alerting, or compliance coverage changes (or "none"). Evidence cites
the Cost Explorer usage type, metric, or API field the finding rests on. A saving that
depends on an assumption (for example a 90-day retention target) says so in the row.

### Priority Matrix
| # | Opportunity | Severity | Effort | Est. Saving |
|---|-------------|----------|--------|-------------|

### Coverage and Limits
- Regions and accounts reviewed, and any skipped (with the reason)
- Caps hit (log groups examined, `ListMetrics` pages, `DescribeQueries` span)
- Whether the optional log-level sample ran, on which log groups, and the bytes it
  scanned

### Next Steps
- Immediate (CRITICAL/HIGH — within 7 days)
- Short-term (MEDIUM — within 30 days)
- Long-term (LOW — within 90 days)

### Appendix — Reference Links
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [Analyzing, optimizing, and reducing CloudWatch costs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html)
- [CloudWatch Logs log classes](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatch_Logs_Log_Classes.html)
- [Working with log groups and log streams](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)
- [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)
