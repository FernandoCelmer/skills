---
name: cloud-billing-audit
description: Monthly financial audit across cloud and SaaS services — AWS, Hostinger, GitHub, OpenAI, Anthropic. Reads billing APIs/MCPs, reports spend per service, delta vs previous month, anomalies, and savings recommendations. Trigger phrases "quanto gastei", "custo mensal", "billing audit", "cloud cost", "relatório financeiro cloud", "auditoria de custos".
version: 1.0.0
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Cloud Billing Audit

Produce a single consolidated spend report across one, several or all supported services. Default output: markdown table + delta vs last month + anomalies + at least one actionable recommendation per service.

Make a todo list and track progress through all steps.

---

## Step 1 — Scope

Ask the user which services to audit when not stated. Supported:

| Service | Data source | Required access |
|---------|-------------|-----------------|
| AWS | `mcp__plugin_aws-amplify_aws-mcp__aws___run_script` → Cost Explorer `GetCostAndUsage` + `GetAnomalies` | AWS creds with `ce:*`, `pricing:Get*` |
| Hostinger | `mcp__hostinger-*` search/execute (`billing-*`, `subscriptions-*` operations) | Hostinger API token |
| GitHub | `gh api /orgs/{org}/settings/billing/{actions,packages,shared-storage}` + `/users/{u}/settings/billing/*` | `gh auth` with `read:org` |
| OpenAI | `curl https://api.openai.com/v1/organization/costs` (requires `OPENAI_ADMIN_KEY`) | Admin API key |
| Anthropic | `curl https://api.anthropic.com/v1/organizations/usage_report/messages` + `/cost_report` | Admin API key, header `anthropic-version: 2023-06-01`, `x-api-key` |

Default period: **current month-to-date** + **previous full month** for delta. User may override: "últimos 90 dias", "janeiro a março".

Accept: `all`, comma list (`aws, github`), single name. Unsupported name → list supported and stop.

---

## Step 2 — Collect

Run every selected source **in parallel** when the tools allow (single message, multiple tool calls). For each source, store raw JSON as an intermediate so re-analysis does not re-query.

### AWS

```python
# via aws-mcp run_script
response = await call_boto3(
    service_name="ce",
    operation_name="GetCostAndUsage",
    params={
        "TimePeriod": {"Start": start, "End": end},
        "Granularity": "MONTHLY",
        "Metrics": ["UnblendedCost", "UsageQuantity"],
        "GroupBy": [{"Type": "DIMENSION", "Key": "SERVICE"}],
    },
)
```

Also call `GetAnomalies` for the same window; include any anomaly with impact ≥ USD 5.

### Hostinger

Use `mcp__hostinger-*__search` to find `billing-*` / `subscriptions-*` operations in each Hostinger product (hosting, domains, vps). Call via `execute` with the user's token. Aggregate by product.

### GitHub

```bash
gh api /orgs/$ORG/settings/billing/actions
gh api /orgs/$ORG/settings/billing/packages
gh api /orgs/$ORG/settings/billing/shared-storage
# for a user account, replace /orgs/$ORG with /users/$USER
```

Convert Actions `total_minutes_used` + `included_minutes` → USD using the current per-minute rate (Linux USD 0.008, Windows 0.016, macOS 0.08) when the billing endpoint does not already include cost.

### OpenAI

```bash
curl -s "https://api.openai.com/v1/organization/costs?start_time=$EPOCH_START&end_time=$EPOCH_END&bucket_width=1d" \
  -H "Authorization: Bearer $OPENAI_ADMIN_KEY"
```

Secret handling: read from `~/.env` or user-provided env var only; never echo the key in the transcript. See the project secrets hygiene rules.

### Anthropic

```bash
curl -s "https://api.anthropic.com/v1/organizations/cost_report?starting_at=$ISO_START&ending_at=$ISO_END" \
  -H "x-api-key: $ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01"
```

---

## Step 3 — Normalize

Convert every source into one record shape:

```json
{"service": "AWS", "subservice": "EC2", "period": "2026-11", "amount_usd": 42.17, "currency": "USD", "quantity": null, "unit": null}
```

Currency conversion: if a source reports BRL/EUR, convert using the daily rate from the end of the period (AWS Pricing's `ExchangeRate` dimension, or a single `curl https://open.er-api.com/v6/latest/USD` call). Show both the native amount and the USD equivalent in the output.

---

## Step 4 — Analyze

For every (service, subservice) compute:

1. **current** (period requested)
2. **previous** (same length, immediately before)
3. **delta_abs** = current − previous
4. **delta_pct** = delta_abs / previous (handle previous = 0 → mark as "new")
5. **anomaly** when |delta_pct| ≥ 50% **and** delta_abs ≥ USD 5

Sum per service and across all services.

---

## Step 5 — Recommend

At least one recommendation per service with current spend > 0. Pick from this catalog; add service-specific items when the raw data points there:

| Service | Checks |
|---------|--------|
| AWS | idle EC2 (< 5% CPU 7d), unattached EBS volumes, old snapshots, S3 Intelligent-Tiering gain, Savings Plans / RIs for sustained usage, NAT Gateway cross-AZ traffic, CloudWatch log retention |
| Hostinger | duplicate domains, unused VPS, hosting plan over-sized vs real traffic, upcoming renewals in next 30 days |
| GitHub | Actions minutes on macOS → Linux where possible, cache bloat, LFS footprint, archived repos still paid for |
| OpenAI | model routing (GPT-4o → GPT-4o-mini where quality allows), cached prompts, batch API for offline jobs |
| Anthropic | prompt caching (hit rate < 50% is a red flag), batch API, Haiku for shallow tasks, Sonnet for routine, Opus only where it matters |

Each recommendation names: **what**, **estimated monthly saving in USD**, **effort** (`low`/`medium`/`high`), **how** (one command or link).

---

## Step 6 — Output

Default: one markdown block, in this order:

```markdown
# Cloud Billing Audit — <period>

## Summary
| Service | Current | Previous | Δ | Δ% |
|---------|--------:|---------:|---:|---:|
| AWS     | ... | ... | ... | ... |
| Total   | ... | ... | ... | ... |

## Breakdown
<per-service table of subservices>

## Anomalies
- <service / subservice>: <delta>, <why if known>

## Recommendations
- [ ] <service>: <what> — save ~USD X/mo, effort <low|medium|high>. How: <command>

## Raw
<paths to intermediates, if saved>
```

Alternative formats on request: `csv`, `json`, `google-doc` (via the Google Docs MCP), `email` (via the `email-profile` MCP). Confirm before publishing anything externally.

---

## Rules

- Never print secrets. Read `OPENAI_ADMIN_KEY` / `ANTHROPIC_ADMIN_KEY` / `HOSTINGER_API_TOKEN` inside the called script, not interpolated into the visible command.
- A missing credential for one service does not block the others — report "skipped: no credentials" and continue.
- A failed API call for one service does not block the others — report "failed: <error>" and continue.
- Always show the window used (`<start> → <end>`) at the top of the report; relative dates ("este mês") must resolve to absolute dates.
- Currency: USD is the canonical unit. Native currency shown in parentheses where it differs.
- Round to 2 decimals for USD, no decimals for percentages.
- The report is read-only. Any action a recommendation points to (stopping an instance, cancelling a subscription, changing a plan) is a separate request — never execute from inside the audit.
