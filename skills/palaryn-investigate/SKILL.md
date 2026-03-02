---
name: palaryn-investigate
description: "Use this skill when the user wants to debug, investigate, or trace agent sessions and blocked requests in their Palaryn workspace. Triggers on: 'what did agent X do', 'investigate session', 'debug blocked request', 'trace error', 'why was this blocked', 'show audit log', 'session history', 'what happened', 'trace request'."
version: 1.0.0
---

You are helping the user investigate and debug agent sessions, blocked requests, and pipeline traces in their Palaryn workspace. Your job is to translate raw event data and pipeline stages into a human-readable narrative.

## Step 1: Resolve workspace

1. Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces"` to list workspaces.
2. If only 1 workspace is returned, use its `id` automatically.
3. If multiple workspaces are returned, ask the user which one to use.
4. Handle errors:
   - **401** → "Your Palaryn session has expired. Please reconnect: run `claude mcp add --transport http palaryn https://app.palaryn.com/mcp` or reinstall the plugin."
   - **403** → "You don't have permission to access this workspace."
   - **429** → "Rate limit hit. Please wait a moment and try again."

## Step 2: Determine investigation scope

Based on what the user asked, choose the right approach:

### A) "What did agent X do?" / "Show recent sessions"

Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/sessions?limit=20"`.

Present a session list:
```
# Recent Sessions

| # | Agent        | Tools Used      | Requests | Status    | Duration | When         |
|---|--------------|-----------------|----------|-----------|----------|--------------|
| 1 | {actor_id}   | {tools_used}    | {count}  | {status}  | {dur}    | {timestamp}  |
| 2 | ...          | ...             | ...      | ...       | ...      | ...          |
```

If the user specified an agent name, filter the results to show only that agent's sessions.

Ask: "Want to drill into a specific session?"

### B) "Investigate session {X}" / User selects a session

Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/sessions/{session_id}"`.

Present a step-by-step timeline:
```
# Session Timeline — {session_id}

Agent: {actor_id}
Duration: {duration}
Status: {overall_status}

## Request Timeline

### 1. {timestamp} — {tool_name} → {status}
   - **URL:** {target URL if applicable}
   - **Pipeline stages:**
     - Auth: passed
     - Policy: {matched rule name or "no match"}
     - DLP scan: {result — clean / transformed / blocked}
     - Rate limit: {passed / throttled}
   - **Result:** {allowed / blocked / transformed}
   {If blocked:} **Blocked by:** {rule_name} — "{rule description}"
   {If transformed:} **Transformed:** {what was changed}

### 2. {timestamp} — {tool_name} → {status}
   ...
```

### C) "Why was this blocked?" / "Debug blocked request"

Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/traces?status=blocked&limit=10"`.

For each blocked trace, explain:
```
# Blocked Requests (last 10)

### 1. {timestamp} — {tool_name} by {actor_id}
   **Blocked by:** {stage that blocked it}
   **Rule:** {rule_name} — {rule description}
   **Reason:** {human-readable explanation}
   **Suggested fix:** {recommendation}
```

### D) "Show traces" / "Audit log"

Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/traces?limit=20"`.

Apply filters if the user specified them:
- `status` filter: allowed, blocked, transformed, error
- `tool` filter: specific tool name
- `event_type` filter: specific event type

Present as a compact log:
```
# Trace Log (last 20)

| Time       | Agent      | Tool        | Target          | Status      |
|------------|------------|-------------|-----------------|-------------|
| {time}     | {actor}    | {tool}      | {url/summary}   | {status}    |
```

## Step 3: Drill-down on specific trace

If the user wants more detail on a specific trace, call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/traces/{task_id}"`.

Present the full event chain for that task:
```
# Trace Detail — {task_id}

Events ({count}):

1. {timestamp} — {event_type}: {description}
2. {timestamp} — {event_type}: {description}
...
```

## Pipeline stage translation guide

When explaining pipeline stages, translate technical names to human-readable descriptions:
- **auth** → "Authentication check — verified the agent's identity"
- **policy** → "Policy evaluation — checked against workspace rules"
- **dlp** → "DLP scan — scanned for sensitive data (PII, secrets, credentials)"
- **rate_limit** → "Rate limiting — checked request frequency against limits"
- **budget** → "Budget check — verified spending is within limits"
- **transform** → "Transformation — modified the request (e.g., redacted sensitive data)"
- **approval** → "Approval required — request held pending human review"
- **proxy** → "Proxy forwarding — sent the request to the target service"

## Suggested fix logic for blocked requests

When a request was blocked, suggest a fix based on the blocking stage:
- **Policy block** → "To allow this, update the workspace policy to add an allow rule for {tool/URL pattern}. Use the `/palaryn-policy-edit` skill to do this conversationally."
- **DLP block** → "DLP detected sensitive data in the request. Either remove the sensitive content or adjust the DLP policy if this is a false positive."
- **Rate limit** → "This agent hit the rate limit. Either wait for the window to reset or increase the rate limit in workspace settings."
- **Budget exceeded** → "The budget limit was reached. Increase the budget or wait for the next billing period."
- **Auth failure** → "Authentication failed. Check that the agent has valid credentials and workspace membership."

## Important notes

- Always translate raw event types and status codes into human-readable language. The user shouldn't need to decode enum values.
- When showing timestamps, use relative time ("2 minutes ago") for recent events and absolute time for older ones.
- If there's no data (no sessions, no traces), say so and suggest the user sends some traffic through the gateway first.
- For follow-up questions, maintain context about which session/trace the user was looking at.
