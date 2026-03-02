---
name: palaryn-approve
description: "Use this skill when the user asks about pending approvals, wants to approve or deny requests, or manage the approval queue in their Palaryn workspace. Triggers on: 'pending approvals', 'approve request', 'deny request', 'what's waiting for approval', 'approval queue', 'approve all', 'review approvals'."
version: 1.0.0
---

You are helping the user manage their Palaryn approval queue. Approvals have a TTL (time-to-live), so this is time-sensitive — getting through the queue quickly matters.

## Step 1: Resolve workspace

1. Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces"` to list workspaces.
2. If only 1 workspace is returned, use its `id` automatically.
3. If multiple workspaces are returned, ask the user which one to use.
4. Handle errors:
   - **401** → "Your Palaryn session has expired. Please reconnect: run `claude mcp add --transport http palaryn https://app.palaryn.com/mcp` or reinstall the plugin."
   - **403** → "You don't have permission to access this workspace."
   - **429** → "Rate limit hit. Please wait a moment and try again."

## Step 2: Fetch the approval queue

Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/approvals"`.

## Step 3: Display the queue

### If there are pending approvals

Group approvals by agent (actor_id) and display:

```
# Approval Queue — {n} pending

## Agent: {actor_id} ({count} requests)

| # | Tool          | Action Summary        | Reason              | Expires In |
|---|---------------|-----------------------|---------------------|------------|
| 1 | {tool_name}   | {args_summary}        | {reason}            | {time}     |
| 2 | {tool_name}   | {args_summary}        | {reason}            | {time}     |

## Agent: {actor_id_2} ({count} requests)

| # | ...           | ...                   | ...                 | ...        |
```

Calculate "Expires In" from `expires_at` relative to now. Flag any approvals expiring in <5 minutes as urgent.

Then offer actions:
- "Approve/deny a specific request by number"
- "Approve all from agent {X}" (if multiple from same agent)
- "Approve all" (if user trusts the queue)
- "Deny all" (with reason)

### If no pending approvals

Show a summary:
```
No pending approvals.

Tip: If you expected to see approvals here, check that your policy rules include
approval requirements (effect: "approval") for the operations you want to review.
```

## Step 4: Process user's decision

### Approve a specific request

Call `mcp__palaryn__http_post` with:
- `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/approvals/{approval_id}/approve"`
- `body={}`

Confirm: "Approved: {tool_name} — {args_summary} (agent: {actor_id})"

### Deny a specific request

Ask for a reason (optional), then call `mcp__palaryn__http_post` with:
- `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/approvals/{approval_id}/deny"`
- `body={"reason": "{user's reason}"}` (or `{}` if no reason)

Confirm: "Denied: {tool_name} — {args_summary} (agent: {actor_id})"

### Approve/deny multiple

Process each approval sequentially, reporting results:
```
Processing 3 approvals for agent {actor_id}:
  1. {tool_name} — {args_summary} → Approved
  2. {tool_name} — {args_summary} → Approved
  3. {tool_name} — {args_summary} → Approved

All done. 3 requests approved.
```

## Step 5: Show updated queue

After any action, fetch the queue again and show the updated count:
```
Queue updated: {n} approvals remaining.
```

If all approvals are cleared: "All clear — no more pending approvals."

## Important notes

- Approvals are time-sensitive. Always show the "Expires In" time prominently.
- If an approval has already expired when the user tries to act on it, the API will return an error — handle gracefully: "This approval has expired. The requesting agent will need to retry."
- Never auto-approve without explicit user consent. Always show what's being approved first.
- When showing args_summary, keep it concise but informative — the user needs to understand what they're approving.
