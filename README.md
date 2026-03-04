# Palaryn — Claude Code Plugin

Security, cost control, and observability gateway for AI agents. Route all agent HTTP requests through Palaryn for automatic DLP scanning, policy enforcement, budget tracking, and full audit trails.

## Install

```bash
# Add the Palaryn marketplace
/plugin marketplace add palaryn-ai/claude-plugin

# Install the plugin
/plugin install palaryn@palaryn-ai-claude-plugin
```

## What it does

Palaryn sits between your AI agent and the outside world. Every HTTP request and response passes through the gateway's security pipeline — before the model sees it.

**Security & DLP**
- Prompt injection detection (regex + LLM-based semantic classifier)
- Secrets scanning (API keys, tokens, credentials)
- PII detection and redaction
- Memory poisoning attack prevention

**Cost Control**
- Per-task, per-user, and per-workspace budget limits
- Real-time token usage tracking across all LLM providers
- Automatic budget enforcement — no more surprise bills

**Policy Engine**
- YAML-based rules + OPA integration
- Allow/deny/require-approval per tool, domain, or action
- Role-based access control (RBAC)

**Observability**
- Full audit trail of every tool call
- Anomaly detection for unusual agent behavior
- OpenTelemetry tracing support

## MCP Tools

Once connected, all HTTP requests go through the Palaryn gateway:

| Tool | Description |
|------|-------------|
| `mcp__palaryn__http_get` | GET requests through the gateway |
| `mcp__palaryn__http_post` | POST requests through the gateway |
| `mcp__palaryn__http_request` | Any HTTP method (PUT, PATCH, DELETE, etc.) |

## Skills

| Skill | Trigger | Description |
|-------|---------|-------------|
| `/palaryn` | Status check | Show gateway status and quick help |
| `palaryn-security-audit` | "security audit", "what got blocked" | Comprehensive security report |
| `palaryn-budget-check` | "check budget", "how much spent" | Spending and token usage breakdown |
| `palaryn-investigate` | "investigate session", "why was this blocked" | Debug blocked requests and trace sessions |
| `palaryn-policy-edit` | "add policy rule", "block domain" | Create and edit gateway policies |
| `palaryn-approve` | "pending approvals", "approve request" | Manage approval queue |
| `palaryn-onboard` | "onboard to Palaryn" | Generate onboarding credentials |

## Connection

The plugin connects to Palaryn via MCP over HTTP with OAuth 2.0 authentication:

```
claude mcp add --transport http palaryn https://app.palaryn.com/mcp
```

OAuth flow is handled automatically on first use — you'll be redirected to log in and authorize.

## Dashboard

Manage your workspace, view audit logs, and configure policies at **https://app.palaryn.com**.

## Links

- [Website](https://palaryn.com)
- [Dashboard](https://app.palaryn.com)
- [GitHub](https://github.com/palaryn-ai/palaryn)
