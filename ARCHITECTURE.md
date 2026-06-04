# Architecture

A high-level view of what's installed where and how data flows. The premise of the Managed Application offer is that *everything customer-related stays in the customer's tenant* — Expecta has no standing access to your data, your queries, or your audit log.

## Components in your tenant

After install, the managed resource group in your subscription holds:

```
Your Azure subscription ─────────────────────────────────────────────────────
│
│  Managed Resource Group (e.g. mrg-acme-mcp-<suffix>)
│
│  ┌──────────────────────────┐         ┌────────────────────────────────┐
│  │   Container App          │         │   Entra ID app registration    │
│  │   (the connector)        │ ◄─────► │   (OAuth client)               │
│  │   • Python runtime       │ uses    │   • Single-tenant              │
│  │   • System-assigned MI   │         │   • Cert-based client auth     │
│  └────────────┬─────────────┘         └────────────────────────────────┘
│               │
│   ┌───────────┼──────────────┬───────────────┐
│   │           │              │               │
│   ▼           ▼              ▼               ▼
│  ┌────────┐ ┌─────────────┐ ┌──────────┐ ┌────────────────────┐
│  │ Key    │ │ Azure SQL   │ │ Per-     │ │ Log Analytics      │
│  │ Vault  │ │ (config DB) │ │ install  │ │ workspace          │
│  │ (cert) │ │             │ │ ACR      │ │ (audit events)     │
│  └────────┘ └─────────────┘ └──────────┘ └────────────────────┘
```

Each connector install is a self-contained slice. No cross-customer shared resources, no Expecta-side bridge identities.

## Identity model

Three distinct identities operate inside the install:

| Identity | What it is | When it's used |
|---|---|---|
| **Connector's system MI** | A managed identity automatically assigned to the Container App | Runtime: reads OAuth cert from Key Vault, reads/writes the config DB, pulls the image from ACR |
| **Entra ID app registration** (created by the install) | A single-tenant Entra app, generated with a unique app ID per install | Runtime: OAuth client for user authentication and On-Behalf-Of token exchange to Power BI |
| **Installer managed identity** | A user-assigned MI provisioned for the install only | Deploy time: creates the Entra app, applies database schema, mints the OAuth cert. Not used after install. |

There is **no Expecta-controlled credential** at any point after install. The connector binary is the same one Expecta ships, but the identities and secrets it operates with are entirely owned by your tenant.

## Per-user authorisation (OBO)

When an LLM client calls the connector's MCP endpoint, the request carries an Entra ID access token issued to the *calling user*. The connector:

1. Validates the token (signature, expiry, tenant — must match your tenant allowlist).
2. Uses the [On-Behalf-Of flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow) to exchange the user's token for a Power BI access token on the same user's behalf.
3. Calls the Power BI XMLA endpoint to execute DAX queries using the user's permissions.

The practical consequence: **a user can only see the workspaces and datasets that user can already see in Power BI Web.** The connector has no privilege escalation path — it cannot read a dataset the user lacks access to, even if a malicious prompt tries to coerce it.

## Data flow

A typical "what were Q3 net sales?" query:

1. User asks the LLM client a business question.
2. LLM client decides which connector tool to call (`query`, `analyze`, `get_report`, etc.) and sends the call to the connector's MCP endpoint with the user's Bearer token.
3. Connector validates the token, looks up the workspace/dataset binding in the config DB.
4. Connector performs OBO exchange against Entra ID to get a Power BI access token *as the user*.
5. Connector calls Power BI XMLA with the OBO token, runs the DAX query.
6. Power BI returns rows.
7. Connector reshapes into a structured MCP response.
8. Audit event lands in your Log Analytics workspace: `req_id`, user `oid`, tool name, workspace+dataset, outcome, latency.
9. Response goes back to the LLM client; LLM client renders the natural-language answer.

No data is held in the connector beyond the immediate request lifecycle. There is no caching of query results outside a short in-process TTL for hot-path metadata.

## What stays in your tenant

- All audit events (tool invocations, OAuth handshakes, DAX queries) — they only ever land in your Log Analytics.
- All configuration (analyses, reconciliation rules, AI Context, admin allowlist) — they only ever live in your config DB. The offer ships the runtime + admin UI; the analyses and rules are configured post-install via the admin UI.
- All OAuth credentials (cert + private key + thumbprint) — they only ever live in your Key Vault.
- The connector image binary — copied to your ACR at install time, pulled from your ACR at runtime.
- Power BI access tokens — never persisted; held in-process for the duration of one OBO exchange and discarded.

## What Expecta has access to

- **Nothing standing.** No persistent Expecta-side service has access to your install.
- **Image updates.** When Expecta publishes a new version, the Marketplace offer's update flow pulls the new image SHA from Expecta's publisher registry into your ACR. This is the only time anything Expecta-controlled crosses your tenant boundary, and it's customer-initiated (you click the update).

For deeper technical details on the auth and audit flows, see [Security](./SECURITY.md). For day-2 operations, see [Operations](./OPERATIONS.md).
