# CloudWatch Cost Optimization Report Template

Output template for the Step 6 report artifact. Load this when generating the final
report so the structure, section order, and table schemas are consistent.

Artifact naming: `cloudwatch-cost-optimization-<account-id>-<YYYY-MM-DD>.md`
Example: `cloudwatch-cost-optimization-123456789012-2026-11-03.md`

The first section is for a technology leader: decisions, owners, and money, in plain
language, with no API names. Everything under "Engineering detail" is for the teams
making the changes.

---

### Report Header
```
# Amazon CloudWatch Cost Optimization — <account-id>
Date: <YYYY-MM-DD> | Regions: <regions> | Months analyzed: <first> to <last complete month>
```

## Executive Summary

| | Monthly | Annualized |
|---|---------|------------|
| CloudWatch spend (last complete month) | $<n> | $<n × 12> |
| Change vs 3 months ago | <+/-n>% | |
| Share of total AWS spend | <n>% | |
| Estimated savings, no visibility impact | $<n> | $<n × 12> |
| Estimated savings, needs a decision | $<n> | $<n × 12> |

- Where the money goes: the top 3 charge families with their share of CloudWatch spend.
- One sentence on the trend and its main driver.

### Decisions needed
Items the skill can't settle on its own. Each names the decision, the default the
report assumed, and the saving that depends on it.
- Retention requirement per data class (application, audit, security logs)
- Alarms that page on-call and are proposed for removal
- Essentials log class moves that need a producer change by another team
- Container Insights mode for non-production clusters

### Top actions
| # | Action (outcome in plain words) | Owner | Effort | Monthly saving |
|---|---------------------------------|-------|--------|----------------|

At most 5 rows, largest saving first. Owner is the team from the owner tag the user
named in Step 1, or "unassigned". Effort: **S** = configuration change in one place,
under a day; **M** = changes across several resources or a producer change, days;
**L** = a migration (for example new Essentials log groups and repointed producers) or
cross-team work, weeks.

**Prevent recurrence:** one line, for example a default log retention in the
infrastructure as code that creates log groups.

---

## Engineering detail

### Cost Attribution
| Charge family | Usage types | Last complete month | 3 months ago | Change | Effective rate |
|---------------|-------------|---------------------|--------------|--------|----------------|

Include an "Other" row for unmatched usage types and a "Not analyzed by this skill"
row (Synthetics, RUM, Application Signals, Database Insights, and similar). Write "rate
not observed" where a family has usage but no cost.

### Top Log Groups
| Log group | Region | Class | Retention | Ingested (GB, month) | Stored (GB) | Filters / indexes | Producer | Owner |
|-----------|--------|-------|-----------|----------------------|-------------|-------------------|----------|-------|

### Custom Metrics by Namespace
| Namespace | Region | Metrics (ListMetrics) | Cardinality driver (dimension) | Distinct values |
|-----------|--------|-----------------------|--------------------------------|-----------------|

State the billed metric count from Cost Explorer above the table.

### Cost Optimization Opportunities
| # | Check (§) | Resource | Severity | Current State | Recommendation | Impact | Evidence and method | Est. Monthly Saving |
|---|-----------|----------|----------|---------------|----------------|--------|---------------------|---------------------|

Impact and Evidence are mandatory for every row. Impact states, in plain language,
what visibility, alerting, or compliance coverage changes (or "none"). Evidence names
the Cost Explorer usage type, metric, or API field, and the method in one line (for
example "storedBytes × share ingested more than 90 days ago; lower bound"). A saving
that depends on an assumption says so.

### Coverage and Limits
- Regions reviewed, and Regions skipped (with the reason)
- Caps hit (log groups examined, `ListMetrics` pages, `DescribeQueries` span)
- Whether the optional log-level sample ran, on which log groups, and the bytes it
  scanned

### Next Steps
Follow the response times in SKILL.md Severity Definitions:
- CRITICAL — within 48 hours
- HIGH — within 7 days
- MEDIUM — within 30 days
- LOW — within 90 days

### Appendix — Reference Links
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [Analyzing, optimizing, and reducing CloudWatch costs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html)
- [CloudWatch Logs log classes](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatch_Logs_Log_Classes.html)
- [Working with log groups and log streams](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)
- [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)
