---
name: palaryn-policy-edit
description: "Use this skill when the user wants to create, edit, or manage Palaryn gateway policies conversationally. Triggers on: 'add policy rule', 'block domain', 'allow API', 'restrict agent', 'require approval for', 'create policy', 'make agents read-only', 'edit policy', 'update rules', 'policy configuration'."
version: 1.0.0
---

You are helping the user create or edit Palaryn gateway policies through a conversational workflow. This is a multi-step process that translates natural language into validated policy rules.

## Step 1: Resolve workspace

1. Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces"` to list workspaces.
2. If only 1 workspace is returned, use its `id` automatically.
3. If multiple workspaces are returned, ask the user which one to use.
4. Handle errors:
   - **401** → "Your Palaryn session has expired. Please reconnect: run `claude mcp add --transport http palaryn https://app.palaryn.com/mcp` or reinstall the plugin."
   - **403** → "You don't have permission to access this workspace."
   - **429** → "Rate limit hit. Please wait a moment and try again."

## Step 2: Fetch current policies

Call `mcp__palaryn__http_get` with `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/policies"`.

Show a brief summary of the current policy state:
- How many rules exist
- Whether it's a custom policy or the default
- List rule names with their effects (allow/deny/transform/approval) as a compact table

## Step 3: Understand the user's intent

If the user already described what they want (e.g., "block DELETE requests to production"), proceed to Step 4.

Otherwise, ask what kind of rule they want. Give examples:
- "Block all requests to *.internal.company.com"
- "Require approval for any DELETE operation"
- "Allow only GET requests for agent 'readonly-bot'"
- "Transform requests to redact API keys in headers"
- "Rate limit agent X to 100 requests per hour"

## Step 4: Generate the rule with AI

Call `mcp__palaryn__http_post` with:
- `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/policies/generate-rule"`
- `body={"description": "<user's natural language description>"}`

This endpoint uses AI to generate a structured policy rule from natural language.

Handle errors:
- **429** → "Rule generation is rate-limited (10/hour). Please wait and try again."
- **503** → "AI rule generation is not configured on this Palaryn instance. You'll need to write the rule manually or ask your admin to configure PALARYN_LLM_API_KEY."
- **502/500** → "The AI service encountered an error generating the rule. Let me try rephrasing your request."

## Step 5: Explain the generated rule

Present the generated rule to the user in a human-readable format. For each field, explain what it does:

**Example output:**
```
Generated rule: "Block production DELETE requests"

  Effect:      DENY (requests matching this rule will be blocked)
  Priority:    100 (evaluated before lower-priority rules)
  Conditions:
    - Method:  DELETE
    - URL:     matches *://production.api.example.com/*
  Approval:    not required

  In plain English: "Any DELETE request targeting production.api.example.com
  will be blocked outright."
```

Ask: "Does this look right? I can adjust it, or we can proceed to validation."

## Step 6: Check for conflicts

Compare the generated rule against the existing rules fetched in Step 2:
- **Shadowed rules**: Does a higher-priority rule already cover the same conditions?
- **Contradictions**: Does an existing rule allow what this rule denies (or vice versa)?
- **Redundancy**: Is there already a rule doing the same thing?

Report any conflicts found. If there are conflicts, suggest how to resolve them (e.g., adjust priority, modify conditions, or remove the conflicting rule).

## Step 7: Validate

Call `mcp__palaryn__http_post` with:
- `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/policies/validate"`
- `body=<the full policy object with the new rule added>`

If validation fails, explain the errors and suggest fixes. Do NOT proceed to apply.

If validation succeeds, confirm: "Validation passed. Ready to apply this policy?"

## Step 8: Apply (only after explicit user confirmation)

**IMPORTANT:** Do NOT apply the policy without the user explicitly confirming (e.g., "yes", "apply it", "go ahead").

Call `mcp__palaryn__http_request` with:
- `method="PUT"`
- `url="https://app.palaryn.com/saas/workspaces/{workspace_id}/policies"`
- `body=<the full validated policy object>`

Handle errors:
- **403** → "You need owner or admin role to update policies."

After successful application, confirm:
```
Policy updated successfully.

Active rules ({count}):
  1. {rule_name} — {effect}
  2. ...

The new rule is now live and will affect all future requests through this workspace.
```

## Important notes

- Always show the rule to the user before applying. Never auto-apply.
- If the user wants to edit an existing rule (not create a new one), fetch the current policies, identify the rule, modify it, and go through validation → confirmation → apply.
- If the user wants to delete a rule, remove it from the policy object and go through validation → confirmation → apply.
- To reset to defaults, call DELETE on the policies endpoint (with user confirmation).
- Keep explanations concise but clear — the value of this skill is making policy editing accessible without knowing the YAML/JSON format.
