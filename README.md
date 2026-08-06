# Expecta Connector — Documentation

The Expecta Connector is an [Azure Managed Application](https://learn.microsoft.com/en-us/azure/azure-resource-manager/managed-applications/overview) that bridges large-language-model clients (Claude Desktop, Claude Code, custom MCP clients) to your Power BI semantic models. Once installed, the connector exposes an [MCP (Model Context Protocol)](https://modelcontextprotocol.io) endpoint that lets LLMs ask business-language questions against your data — *"what were Q3 net sales by segment?"*, *"show me retention trends"*, *"reconcile this number against management accounts"* — and return grounded, audited answers.

This site is the customer-facing documentation. Source code and release notes are maintained privately by [Expecta](https://expecta.io); reach out if you need access for a specific integration question.

## Highlights

- **Runs in your tenant.** The connector + config database + audit logs all live in *your* Azure subscription. No data leaves your tenant boundary; no Expecta-side service has standing access to your Power BI workspaces.
- **Per-user authorisation.** Every query is performed using the calling user's own Power BI permissions via On-Behalf-Of (OBO). The connector cannot see data the user can't already see in Power BI.
- **Modern auth posture.** Certificate-based OAuth client authentication; no long-lived client secrets.
- **Tenant-scoped audit log.** Every tool invocation lands in *your* Log Analytics workspace, with the calling user's identity, the workspace and dataset touched, and the outcome.
- **Interactive dashboards, in the chat.** On clients that support it, answers can render as live, refreshable dashboards inside the conversation — not just text. A user can also have one built against their own model and save it, then reopen it later by name. Saved dashboards run under that user's own Power BI permissions, like every other query.
- **Zero-ops install.** A few wizard fields, ~10 minutes, no post-install steps for the common path.

## Read next

- [**Install guide**](./INSTALL.md) — what the install wizard collects, what gets deployed in your subscription, prerequisites.
- [**Architecture**](./ARCHITECTURE.md) — what runs where, identity model, data flow.
- [**Operations**](./OPERATIONS.md) — custom domain, certificate rotation, scaling, updates.
- [**Security**](./SECURITY.md) — auth model, data residency, audit, what's verifiable from your side.
- [**Troubleshooting**](./TROUBLESHOOTING.md) — install-time and runtime issues with their fixes.
- [**Support**](./SUPPORT.md) — how to reach Expecta engineering.
