# CloudWatch Cost Optimization — AWS DevOps Agent Skill

A read-only Amazon CloudWatch cost optimization skill for [AWS DevOps Agent](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent.html). It attributes CloudWatch spend by Cost Explorer usage type, traces it back to specific log groups, metrics, alarms, dashboards, and queries, and generates a severity-ranked report of savings aligned with [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/) and [Analyzing, optimizing, and reducing CloudWatch costs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html).

## Purpose

CloudWatch is often one of the largest lines on an AWS bill, and the cost builds up across many small decisions: log groups created without a retention policy, verbose application logging in production, VPC Flow Logs sent to CloudWatch Logs, a metric dimension that holds a request ID, alarms left behind after a service is retired, and third-party tools polling `GetMetricData`. This skill teaches the agent how each CloudWatch charge is computed, which lever actually reduces it, and how to find the resources behind it, then recommends concrete reductions with their visibility and compliance impact.

## Key Capabilities

- Attributes CloudWatch spend by Cost Explorer usage type and computes effective unit rates for the account, so savings use the rates the account actually pays
- Finds log groups with no retention ("Never expire") or retention far beyond the requirement, sized by stored bytes
- Ranks log groups by ingestion and names the producer, including vended logs such as VPC Flow Logs
- Identifies Standard log groups eligible for the Essentials (formerly Infrequent Access) log class, checking the features that class doesn't support
- Flags Lambda functions logging at `DEBUG` or `TRACE`, with an optional, capped, user-approved Logs Insights sample of log levels
- Finds high-cardinality custom metrics and the dimension driving them
- Flags alarms stuck in `INSUFFICIENT_DATA`, alarms with no actions, high-resolution and anomaly detection alarms, and dashboards nobody opens
- Attributes Logs Insights scan spend by query and caller from query history, and flags `GetMetricData` polling, unfiltered metric streams, and idle Contributor Insights rules
- Finds orphaned resources: log groups, alarms, and dashboards left behind after the Lambda function or other resource they observe was deleted
- Applies interaction rules so overlapping savings on the same bytes are never double-counted
- Reviews one account, a workload spread across several accounts (by account list, Cost Category, or cost allocation tag), or a whole organization, with a per-account view
- Opens the report with a one-screen Leadership Summary (spend trend, share of the AWS bill, savings range, posture KPIs, top 3 actions), followed by the engineering detail; accounts whose spend is within the free tier get a short form

## Prerequisites

### 1. An AWS DevOps Agent Space with the target AWS account configured as a cloud source

For a workload or organization review, the Agent Space also needs:

- the AWS Organizations management (payer) account, or an account with Cost Explorer access to the linked accounts, for spend across accounts
- a cloud-source association for each account whose resources should be inspected. Accounts without one are reported with spend only.
- for tag-based workloads, the tag activated as a cost allocation tag

### 2. IAM permissions for the DevOps Agent's primary cloud-source role

**No additional IAM permissions are required.** Every call this skill makes is covered by the AWS managed [`AIDevOpsAgentAccessPolicy`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AIDevOpsAgentAccessPolicy.html), checked against policy version v12:

- CloudWatch Logs: `logs:Describe*` and `logs:List*` (log groups, metric and subscription filters, query history, deliveries, field indexes, anomaly detectors), plus `logs:StartQuery`, `logs:GetQueryResults`, and `logs:StopQuery` for the optional log-level sample
- CloudWatch: `cloudwatch:GetMetricData`, `cloudwatch:List*`, `cloudwatch:Describe*`, `cloudwatch:GetDashboard`, `cloudwatch:GetMetricStream`
- `lambda:ListFunctions` (log level configuration)
- `ec2:DescribeFlowLogs` (vended log sources)
- `cloudtrail:LookupEvents` (optional: dashboard views and `GetMetricData` callers)
- `ce:GetCostAndUsage`, granted via `ce:Get*`
- `organizations:ListAccounts` (account names for workload and organization scope)

There is no CloudFormation template to deploy for this skill. The skill doesn't call the AWS Price List API; it derives unit rates from Cost Explorer instead.

The skill operates in **read-only** mode. It never calls `PutRetentionPolicy`, `DeleteLogGroup`, `PutMetricFilter`, `DeleteAlarms`, `DeleteDashboards`, `UpdateFunctionConfiguration`, or any other mutation.

## Limitations

- Without Cost Explorer access, findings are still reported but dollar impact is labeled "not quantified".
- `DescribeQueries` returns recent query history only; Logs Insights findings state the span they cover.
- `ListMetrics` returns only metrics with data in the past two weeks, and the skill stops counting after about 25,000 metrics per Region and reports a lower bound.
- Dashboard usage relies on CloudTrail recording `GetDashboard` read events. Where it doesn't, the skill reports usage as unknown.
- The skill doesn't read log content, except in the optional log-level sample, which runs only after the user approves the log groups, time window, and estimated bytes scanned, and reports counts only.
- Synthetics, RUM, Application Signals, Database Insights, Internet Monitor, and Network Flow Monitor charges are shown in the cost attribution but not analyzed.
- A workload defined by a cost allocation tag captures only tagged spend; the skill reports the untagged remainder separately.
- Retention and alerting requirements are the user's to confirm; the skill states the tradeoff but does not decide it.

## Agent Types

- **Chat tasks**: "why is my CloudWatch bill so high", "which log groups never expire", "find unused alarms".

## Uploading to AWS DevOps Agent

> Reference: [Uploading a skill](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-devops-agent-skills.html#uploading-a-skill)

Build the zip from **inside** the skill directory so `SKILL.md` sits at the archive root, excluding development-only files:

```bash
cd skills/cloudwatch-cost-optimization
zip -r ../cloudwatch-cost-optimization.zip . \
  -x 'README.md' 'CHANGELOG.md' '.skilleval.yaml' '.skilleval.yml' 'evals/*'
```

Then in the Operator Web App: **Skills → Add skill → Upload skill**, drag in the zip, select **Chat tasks**, review validation, and **Upload**.

## How to Use This Skill

In DevOps Agent Chat:

- *"Review my CloudWatch costs for account 123456789012 across all regions."*
- *"Why did my CloudWatch bill go up this month?"*
- *"Which log groups have no retention policy, and what would 90-day retention save?"*
- *"Which of my log groups could move to the Essentials log class?"*
- *"Are any of my custom metrics high cardinality?"*
- *"Find CloudWatch alarms and dashboards nobody uses."*
- *"Who is running the Logs Insights queries that cost the most?"*
- *"Review CloudWatch costs for our payments workload: accounts 111122223333 and 123456789012."*
- *"Give me a leadership summary of CloudWatch spend across the organization."*

The agent will attribute spend, inventory the resources behind it, size each opportunity, and generate `cloudwatch-cost-optimization-<account-id | workload | org>-<YYYY-MM-DD>.md`, opening with a Leadership Summary.

## Related

- [CloudTrail Cost Optimization](../cloudtrail-cost-optimization/) for CloudTrail trail and Lake costs
- [DevOps Agent Cost Insights](../devops-agent-cost-insights/) for the cost of DevOps Agent itself

## Non-production disclaimer

> ⚠️ This skill is sample code, not intended for production use without additional review and testing. Validate in a non-production environment first. Recommendations are derived from observed configuration and usage. Review their visibility, alerting, and compliance impact before applying them, and never apply a CloudWatch change you have not read.
