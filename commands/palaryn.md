---
description: Show Palaryn gateway status and quick help
allowed-tools: [Read]
model: haiku
---

You are a helpful assistant providing Palaryn gateway status and quick-start information.

When the user runs `/palaryn`, display the following information in a clear, concise format:

## Palaryn Gateway

**Dashboard:** https://app.palaryn.com

### Available MCP Tools

All HTTP requests routed through Palaryn get automatic DLP scanning, policy enforcement, budget tracking, and audit logging.

| Tool | Description |
|------|-------------|
| `mcp__palaryn__http_get` | GET requests through the gateway |
| `mcp__palaryn__http_post` | POST requests through the gateway |
| `mcp__palaryn__http_request` | Any HTTP method (PUT, PATCH, DELETE, etc.) |

### Quick Start Tips

1. **Use Palaryn tools for all HTTP requests** — replace `curl`, `WebFetch`, or direct API calls with the MCP tools above
2. **Check your budget** — the dashboard shows real-time spend tracking per workspace
3. **Review audit logs** — every request is logged with full request/response details
4. **Set policies** — configure DLP rules, rate limits, and allowed domains in the dashboard

### Connection Status

Check if the MCP connection is active by looking at your configured MCP servers. If `palaryn` appears in the list, you're connected and ready to go.

If you need to reconnect, the plugin should handle OAuth automatically. Visit the dashboard to manage your API keys and workspace settings.
