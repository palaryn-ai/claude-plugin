---
name: palaryn-budget-check
description: "Use this skill when the user asks about budget, spending, costs, or token usage in their Palaryn workspace. Triggers on: 'check budget', 'how much spent', 'token usage', 'cost breakdown', 'burn rate', 'am I over budget', 'spending report', 'cost projection', 'which model costs most'."
version: 1.0.0
---

You are analyzing the user's Palaryn workspace budget and spending, providing insights the dashboard doesn't: burn rate projections, cost optimization suggestions, and budget exhaustion estimates.

## Step 1: Resolve workspace

1. Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces"` to list workspaces.
2. If only 1 workspace is returned, use its `id` automatically.
3. If multiple workspaces are returned, ask the user which one to use.
4. Handle errors:
   - **401** → "Your Palaryn session has expired. Please reconnect: run `claude mcp add --transport http palaryn https://app.palaryn.com/mcp` or reinstall the plugin."
   - **403** → "You don't have permission to access this workspace."
   - **429** → "Rate limit hit. Please wait a moment and try again."

## Step 2: Gather budget and usage data

Make these 3 API calls (all in parallel if possible):

1. `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/budgets"`
2. `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/budget-config"`
3. `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/llm-usage?range=7d"`

## Step 3: Compute derived metrics

From the gathered data, calculate:

### Spend vs Limit
For each configured budget limit (daily, monthly, task-level):
- Current spend amount
- Budget limit
- Percentage used: `(spend / limit) × 100`
- Status: green (<60%), yellow (60-80%), red (>80%), critical (>95%)

### Daily Burn Rate
From the 7-day LLM usage time series:
- Calculate average daily cost over the last 7 days
- Identify trend: increasing, stable, or decreasing
- Compare weekday vs weekend if data allows

### Budget Exhaustion Projection
If a monthly budget is set:
- Days remaining in current month
- Projected total spend: `current_spend + (daily_burn_rate × days_remaining)`
- Will it exceed the limit? By how much?
- Estimated exhaustion date: `current_date + (remaining_budget / daily_burn_rate)`

### Cost Breakdown by Model
From `by_model` in LLM usage:
- Sort by cost descending
- Calculate percentage of total cost per model
- Identify the most expensive model

## Step 4: Generate the report

Present the report in this format:

```
# Budget Report — {workspace_name}

## Current Spending

| Budget          | Spent     | Limit     | Used  | Status |
|-----------------|-----------|-----------|-------|--------|
| Daily           | ${x}      | ${limit}  | {x}%  | {status} |
| Monthly         | ${x}      | ${limit}  | {x}%  | {status} |
| Per-task        | ${x} avg  | ${limit}  | —     | {status} |

## Burn Rate

- **Daily average (7d):** ${x}/day
- **Trend:** {increasing/stable/decreasing} ({+/-x}% vs previous week)
- **Projection:** At this rate, monthly spend will be ~${x}

## Budget Exhaustion

{If monthly budget set:}
- Remaining budget: ${x}
- At current burn rate: budget exhausts on **{date}** ({n} days from now)
{If on track:}
- On track to use {x}% of monthly budget

## Cost by Model

| Model            | Requests | Cost     | % of Total | Avg Cost/Req |
|------------------|----------|----------|------------|--------------|
| {model}          | {n}      | ${x}     | {x}%       | ${x}         |

## Optimization Suggestions

{Generate 1-3 suggestions based on the data:}
```

## Optimization suggestion logic

Generate suggestions based on actual data patterns:

- **Expensive model dominance**: If one model accounts for >70% of costs, suggest: "**{x}%** of your spend is on {model}. For routine tasks, switching to {cheaper_alternative} could save ~${estimate}/day. Consider adding a policy rule to route non-critical requests to a cheaper model."
- **Budget near limit**: If any budget >80%, warn: "Your {daily/monthly} budget is at **{x}%**. Consider increasing the limit or reviewing which agents are driving the most spend."
- **Increasing trend**: If burn rate is increasing week-over-week, note: "Spending is trending **up {x}%** compared to last week. Check if new agents or workflows were added."
- **No budget set**: If no custom budget is configured, suggest: "No custom budget limits are set. Consider configuring monthly limits to prevent unexpected costs."
- **Low usage**: If spending is minimal, note: "Usage is low — no optimization needed at this time."

## Important notes

- If no LLM usage data exists (new workspace), say so and suggest the user sends some traffic first.
- Always show actual dollar amounts, not just percentages.
- If budget limits are not configured, still show spending data and suggest setting limits.
- For the cost model comparison, use common knowledge about model pricing tiers (e.g., Opus > Sonnet > Haiku) to suggest cheaper alternatives.
