---
name: palaryn-security-audit
description: "Use this skill when the user asks for a security overview, audit, or threat report for their Palaryn workspace. Triggers on: 'check security', 'security audit', 'DLP detections', 'anomaly report', 'agent trust', 'what got blocked', 'shield score', 'security grade', 'threat summary'."
version: 1.0.0
---

You are performing a comprehensive security audit of the user's Palaryn workspace by correlating data from multiple sources into a single actionable report. The dashboard shows these metrics on separate pages — your job is to synthesize them into one narrative.

## Step 1: Resolve workspace

1. Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces"` to list workspaces.
2. If only 1 workspace is returned, use its `id` automatically.
3. If multiple workspaces are returned, ask the user which one to use.
4. Handle errors:
   - **401** → "Your Palaryn session has expired. Please reconnect: run `claude mcp add --transport http palaryn https://app.palaryn.com/mcp` or reinstall the plugin."
   - **403** → "You don't have permission to access this workspace."
   - **429** → "Rate limit hit. Please wait a moment and try again."

## Step 2: Gather data from all 4 sources

Make these 4 API calls (all in parallel if possible):

1. `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/dashboard/stats"`
2. `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/security"`
3. `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/anomalies/baseline"`
4. `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/agents/trust-leaderboard"`

If any call fails, note it in the report and continue with available data.

## Step 3: Compute security grade

Calculate a composite security grade (A–F) using these weights:

| Component | Weight | Source | Scoring |
|-----------|--------|--------|---------|
| Shield Score | 40% | dashboard/stats → `shield_score` | Use directly (0–100) |
| DLP Severity | 30% | security → `stats` | Score: 100 - (high×10 + medium×5 + low×1), min 0 |
| Anomalies | 15% | anomalies/baseline → `alerts` | 100 if no alerts, -20 per active alert, min 0 |
| Trust Health | 15% | trust-leaderboard → `agents` | Average trust score across all agents (0–100) |

**Grade thresholds:**
- A: 90–100
- B: 80–89
- C: 70–79
- D: 60–69
- F: below 60

## Step 4: Generate the report

Present the report in this format:

```
# Security Audit — {workspace_name}

## Overall Grade: {grade} ({score}/100)

| Component        | Score | Weight | Details                          |
|------------------|-------|--------|----------------------------------|
| Shield Score     | {x}   | 40%    | {allowed}% allowed, {blocked}% blocked |
| DLP Detections   | {x}   | 30%    | {high} high, {medium} medium, {low} low severity |
| Anomaly Status   | {x}   | 15%    | {n} active alerts                |
| Agent Trust      | {x}   | 15%    | {n} agents tracked               |

## Top Findings

1. **{finding}** — {description and recommendation}
2. **{finding}** — {description and recommendation}
3. **{finding}** — {description and recommendation}

## Problematic Agents (lowest trust)

| Agent | Trust Score | Top Concern |
|-------|-------------|-------------|
| {id}  | {score}     | {reason}    |

## Recommendations

- {actionable recommendation based on findings}
- {actionable recommendation based on findings}
- {actionable recommendation based on findings}
```

## Finding generation logic

Derive findings from the data:
- **High DLP detections** → "X high-severity DLP events detected in the last 24h. Review the patterns triggering these and tighten policies."
- **Low shield score** → "Shield score is {X}% — {Y}% of requests are being blocked. This could indicate misconfigured policies or actual threats."
- **Active anomaly alerts** → "Anomaly detector flagged {N} alerts: {describe each alert}. Check if these represent legitimate behavior changes."
- **Low trust agents** → "Agent '{id}' has a trust score of {X}. Consider restricting its permissions or requiring approval for sensitive operations."
- **No issues** → "No significant security issues detected. Workspace is operating within normal parameters."

## Important notes

- If there's no data (new workspace, no traffic), say so clearly instead of generating a meaningless report.
- Keep the tone professional but direct — this is a security report, not marketing.
- If the user asks follow-up questions about specific findings, help them drill down using the appropriate endpoints.
