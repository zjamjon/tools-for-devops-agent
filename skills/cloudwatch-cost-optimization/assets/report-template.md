# CloudWatch Cost Optimization Report Template

Output template for the Step 6 report artifact. Load this when generating the final
report so the structure, section order, and table schemas are consistent.

Artifact naming: `cloudwatch-cost-optimization-<scope>-<YYYY-MM-DD>.md`, where
`<scope>` is the account ID, the workload name, or `org`.
Example: `cloudwatch-cost-optimization-123456789012-2026-11-03.md`

The report has two audiences. The **Leadership Summary** is for a technology leader:
plain language, dollars and percentages, no API names, at most one screen. Everything
under **Engineering detail** is for the engineers who will make the changes.

---

### Report Header
```
# Amazon CloudWatch Cost Optimization — <account ID | workload name | organization>
Date: <YYYY-MM-DD> | Scope: <accounts> / <regions> | Analysis window: <start> to <end>
```

## Leadership Summary

**Spend**
| Measure | Value |
|---------|-------|
| CloudWatch spend, last complete month | $<n> (<+/-n>% vs previous month) |
| 6-month trend | <rising / flat / falling>, <first month> $<n> → <last month> $<n> |
| Share of total AWS spend | <n>% of $<total> |
| Largest charge family | <family>, <n>% of CloudWatch spend |

**Savings potential:** $<low>–$<high> per month. Low = changes with no visibility
impact (orphaned resources, unused alarms and dashboards, Essentials class moves).
High = also includes retention and logging-level changes that need an owner's
agreement. "Ready now" = $<n>.

**Posture** (meaningful even when spend is low)
| KPI | Value |
|-----|-------|
| Stored log data with no retention (Never expire) | <n>% of <n> GB |
| Log groups with a retention policy | <n> of <n> (<n>%) |
| Orphaned log groups / alarms / dashboards | <n> / <n> / <n> |
| Log ingestion, month over month | <+/-n>% |
| Billed custom metrics | <n> |

**Top 3 actions**
| # | Action (outcome in plain words) | Owner | Effort | Monthly saving |
|---|---------------------------------|-------|--------|----------------|

Owner is a team type (platform / application / security), not a person.

**Risk if nothing changes:** one or two sentences, for example "storage on
never-expiring log groups grows about <n> GB a month and is never released".

### Per-Account View (workload or organization scope only)
| Account | CloudWatch spend | Trend | Share of scope | Never-expire bytes | Orphans | Top action |
|---------|------------------|-------|----------------|--------------------|---------|------------|

Accounts the Agent Space can't reach show spend only and "not inspected" in the other
columns.

---

## Engineering detail

### Cost Attribution
| Charge family | Usage types | Cost in window | Share | Effective rate | Change vs previous window |
|---------------|-------------|----------------|-------|----------------|---------------------------|

Include an "Other" row for unmatched usage types and a "Not analyzed by this skill"
row (Synthetics, RUM, Application Signals, Database Insights, and similar). Write "rate
not observed" where a family has usage but no cost.

### Top Log Groups
| Account | Log group | Region | Class | Retention | Ingested (GB, window) | Stored (GB) | Metric / subscription filters | Producer |
|---------|-----------|--------|-------|-----------|-----------------------|-------------|-------------------------------|----------|

### Custom Metrics by Namespace
| Account | Namespace | Region | Metric count | Cardinality driver (dimension) | Distinct values |
|---------|-----------|--------|--------------|--------------------------------|-----------------|

### Cost Optimization Opportunities
| # | Check (§) | Account | Resource | Severity | Current State | Recommendation | Impact | Evidence | Est. Monthly Saving |
|---|-----------|---------|----------|----------|---------------|----------------|--------|----------|---------------------|

Impact and Evidence are mandatory for every row. Impact states, in plain language,
what visibility, alerting, or compliance coverage changes (or "none"). Evidence cites
the Cost Explorer usage type, metric, or API field the finding rests on. A saving that
depends on an assumption (for example a 90-day retention target) says so in the row.
Drop the Account column for single-account scope.

### Priority Matrix
| # | Opportunity | Severity | Effort | Est. Saving |
|---|-------------|----------|--------|-------------|

### Coverage and Limits
- Accounts and Regions reviewed, accounts that were spend-only (with the reason), and
  Regions skipped
- For tag or Cost Category scope: the untagged remainder and how it was handled
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

---

## Low-spend short form

Use this instead of the full report when all CloudWatch spend in the window is inside
the free tier (cost ≈ $0):

1. Report Header
2. Leadership Summary — Spend (state plainly that spend is within the free tier and
   savings are $0), Posture, and Risk if nothing changes. Replace Top 3 actions with
   the hygiene list below.
3. **Hygiene list** — at most 5 bullets, highest stored bytes first, each naming the
   resource count, the action, and the owner type (for example "40 log groups have no
   retention; set one per your retention policy — platform team").
4. Coverage and Limits

Omit the engineering tables and any "no action" rows.
