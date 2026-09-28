# Architecture

A high-level view of what's installed where and how data flows. The premise of the Managed Application offer is that *everything customer-related stays in the customer's tenant* — your data, your queries and your audit log never pass through an Expecta service.

## Components in your tenant

After install, the managed resource group in your subscription holds:

```
Your Azure subscription ─────────────────────────────────────────────────────
│
│  Managed Resource Group (e.g. mrg-acme-mcp-<suffix>)
│
│  ┌──────────────────────────┐         ┌────────────────────────────────┐
│  │   Container App          │         │   Entra ID app registration    │
│  │   (the connector)        │ ◄─────► │   (in your directory, created  │
│  │   • Python runtime       │ uses    │    by your admin after install)│
│  │   • User-assigned MI     │         │   • Cert-based client auth     │
│  └────────────┬─────────────┘         └────────────────────────────────┘
│               │
│   ┌───────────┼──────────────┬───────────────┐
│   │           │              │               │
│   ▼           ▼              ▼               ▼
│  ┌────────┐ ┌─────────────┐ ┌──────────┐ ┌────────────────────┐
│  │ Key    │ │ Azure SQL   │ │ Per-     │ │ Log Analytics      │
│  │ Vault  │ │ (config DB) │ │ install  │ │ workspace          │
│  │ (cert, │ │             │ │ ACR      │ │ (audit events,     │
│  │ secrets│ │             │ │          │ │  update results)   │
│  └────────┘ └─────────────┘ └────▲─────┘ └────────────────────┘
│                                  │
│                        ┌─────────┴──────────┐
│                        │ Nightly update job │  (Container Apps job)
│                        └────────────────────┘
```

Each connector install is a self-contained slice. No cross-customer shared resources, no Expecta-side bridge identities.

## Identity model

These identities operate inside the install:

| Identity | What it is | When it's used |
|---|---|---|
| **Connector runtime identity** | A user-assigned managed identity attached to the Container App | Runtime: reads its secrets from Key Vault, reads/writes the config DB, pulls the image from your registry, reads the managed application's tags |
| **Entra ID app registration** (created by your admin's post-install script) | An app registration in your directory, unique per install | Runtime: OAuth client for user sign-in and On-Behalf-Of token exchange to Power BI. It is registered as multi-tenant because sign-in goes through Microsoft's common endpoint; the connector itself only accepts users from the tenants in its allowlist (default: yours). |
| **Installer identity** | A user-assigned managed identity used by the install's scripts | Deploy time: copies the image into your registry, applies the database schema and creates the connector's database user, creates the OAuth certificate in Key Vault. It never touches Microsoft Entra ID. |
| **Updater identity** | A user-assigned managed identity used by the nightly update job | Nightly: reads the registry access from Key Vault, imports new images into your registry, moves the connector to them. No Microsoft Graph rights, nothing outside the managed resource group. |

**No Expecta credential gives access into your tenant.** The only Expecta-issued credential in the install is the registry access (access name + password), which lets *your* install pull images from Expecta's registry; it grants Expecta nothing. Everything else — certificate, secrets, identities — is created in and owned by your tenant.

## Per-user authorisation (OBO)

When an LLM client calls the connector's MCP endpoint, the request carries an Entra ID access token issued to the *calling user*. The connector:

1. Validates the token (signature, expiry, tenant — must match your tenant allowlist).
2. Uses the [On-Behalf-Of flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow) to exchange the user's token for a Power BI access token on the same user's behalf.
3. Calls the Power BI REST API to run queries using that person's own permissions.

The identity is taken from **each request**, not from the conversation that opened the connection: after a token refresh the connector uses the new token, and an MCP session can only be used by the user who opened it.

The practical consequence: **a user can only see the workspaces and datasets that user can already see in Power BI Web.** The connector has no privilege escalation path — it cannot read a dataset the user lacks access to, even if a malicious prompt tries to coerce it.

## Data flow

A typical "what were Q3 net sales?" query:

1. User asks the LLM client a business question.
2. LLM client decides which connector tool to call (`query`, `analyze`, `get_report`, etc.) and sends the call to the connector's MCP endpoint with the user's Bearer token.
3. Connector validates the token, looks up the workspace/dataset binding in the config DB.
4. Connector performs OBO exchange against Entra ID to get a Power BI access token *as the user*.
5. Connector calls the Power BI REST API with that token and runs the query.
6. Power BI returns rows.
7. Connector reshapes into a structured MCP response.
8. An audit entry lands in your Log Analytics workspace: who asked, what was asked for, which report and dataset, the outcome, and a reference number.
9. Response goes back to the LLM client; LLM client renders the natural-language answer.

No data is held in the connector beyond the immediate request lifecycle. There is no caching of query results outside a short in-process TTL for hot-path metadata.

## What stays in your tenant

- All audit events (tool invocations, OAuth handshakes, DAX queries) — they only ever land in your Log Analytics.
- All configuration (analyses, reconciliation rules, AI Context, admin allowlist) — they only ever live in your config DB. The offer ships the runtime + admin UI; the analyses and rules are configured post-install via the admin UI.
- All OAuth credentials (cert + private key + thumbprint) — they only ever live in your Key Vault.
- The connector image — copied into your registry, pulled from your registry at runtime.
- Power BI access tokens — never persisted; held in-process for the duration of one OBO exchange and discarded.

The one thing that leaves by design is **the answer**: query results are returned to the AI assistant the user is working in (for example Claude), because that is how the assistant writes its reply. That assistant's provider processes them under its own terms; Expecta never sees them.

## What Expecta has access to

- **Your data: none.** No Expecta service sits in the data path, and no Expecta credential can read Power BI.
- **The managed resource group:** as with every Azure Managed Application, the publisher holds a role on the managed resource group only. What it is for, who holds it, and how you can see every action taken with it is described in [Terms — Expecta's access to your deployment](./TERMS.md#expectas-access-to-your-deployment).
- **Updates:** the nightly job inside your install checks Expecta's registry and, when there is a new version, copies it into your registry. The connector image is the only thing that crosses your tenant boundary, and the connection is made from your install, not by Expecta.

For deeper technical details on the auth and audit flows, see [Security](./SECURITY.md). For day-2 operations, see [Operations](./OPERATIONS.md).
