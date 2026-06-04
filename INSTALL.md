# Install guide

The Expecta Connector installs from the Azure Marketplace as a Managed Application offer. The install runs entirely inside *your* Azure subscription — the connector container, its configuration database, its Key Vault, and its Log Analytics workspace are all provisioned in a managed resource group in your tenant.

## Prerequisites

You'll need:

- **An Azure subscription** in the tenant where Power BI lives, with enough quota for a small set of resources (one Container App, one Key Vault, one Basic-tier Azure Container Registry, one serverless Azure SQL database, optionally one Log Analytics workspace).
- **Owner or User Access Administrator** on the subscription, to create the connector's Entra app registration and grant role assignments. The wizard validates this up front.
- **A user with Power BI access** (`Dataset.Read.All` delegated, granted through Power BI workspace membership) — typically yourself. The connector authenticates each query under the calling user's identity, so users who can't see a workspace in Power BI Web won't see it through the connector either.
- **An email** for the first tenant admin who'll log into the connector's `/admin` web UI to configure analyses, reconciliation rules, and AI context.

## What the wizard collects

The install wizard has up to four steps. Most installs only touch the first two.

### Step 1 — Basics

| Field | Why |
|---|---|
| **Azure subscription / region / resource group** | Standard Azure resource selectors. We recommend a fresh resource group named after the install. |
| **First tenant admin email** | Becomes the first row in the connector's `admin_users` table so this user can log into `/admin` immediately after install. Resolved to an Entra Object ID at install time via Microsoft Graph. |
| **Resource name prefix** | 3–12 lowercase letters and digits. Used to derive resource names: prefix `acme` produces `acme-mcp-<unique-suffix>`, `acmekv<suffix>`, etc. |

### Step 2 — Audit and telemetry

Audit events (tool invocations, OAuth handshakes, DAX queries) need a Log Analytics workspace to land in. Two choices:

- **Create new workspace in this Managed App** (default) — a fresh workspace is provisioned alongside the connector, with 365-day retention. Simplest.
- **Use my existing workspace** — paste the full resource ID of an existing workspace in your subscription (the workspace can be cross-region). Useful if you already centralise telemetry in one place.

### Step 3 — Access control

Only users from the tenants you list here can authenticate to the connector. Default: your own tenant only — deny-by-default for everyone else. Add comma-separated tenant GUIDs if you want to extend access (e.g. for managed-service-provider scenarios where one operator administers connectors across multiple end-customer tenants).

### Step 4 — Custom domain (optional)

If you plan to expose the connector on a custom hostname (e.g. `mcp.acme.com`) instead of the auto-generated `<resourceName>.italynorth.azurecontainerapps.io`, type the FQDN here. The wizard pre-populates the OAuth app registration's callback URLs for both hostnames so once you bind the custom domain to the Container App post-install (see [Operations](./OPERATIONS.md)), redirect handling works without further changes.

The custom-domain DNS, TLS certificate, and Container Apps domain binding are post-install steps — the wizard does the OAuth-side prep only.

## What gets created in your subscription

Inside the managed resource group, the install provisions:

- **Container App** running the connector image. System-assigned managed identity has narrow scopes: pull from the per-install ACR, read secrets from Key Vault, read/write the config database.
- **Azure SQL serverless database** holding configuration (analyses, reconciliation rules, AI context, tenant admin allowlist) and audit metadata. AAD-only auth, no SQL passwords.
- **Key Vault** holding the OAuth client certificate, the database connection string, and any rotation-managed secrets. RBAC mode, soft-delete + purge-protection enabled.
- **Azure Container Registry (Basic tier)** holding a copy of the connector image, copied from Expecta's publisher registry at install time so future container pulls never leave your tenant.
- **Entra ID application registration** for OAuth client authentication and On-Behalf-Of token exchange.
- **Log Analytics workspace** (only if you selected "Create new" in Step 2) for audit event ingestion.

Ongoing Azure cost for a single-user install is approximately **$25–40/month**, dominated by the Basic ACR (~$5), the serverless SQL DB (auto-paused most of the time, ~$5–15 depending on usage), and the Container App's consumption tier (scales to zero when idle, ~$10–20 with light usage). Costs scale with usage; high-traffic installs run higher.

## After install

The post-install management blade in the Azure Portal shows:

- **MCP endpoint URL** — paste this into your MCP client (Claude Desktop, Claude Code, etc.).
- **Connector admin URL** — `https://<connector-fqdn>/admin/login` — the first-admin sign-in screen.
- **Metrics** — request count and CPU.
- **Useful links** — back to this documentation, support contact.

The first thing to do is sign into `/admin` as the tenant-admin email you set in the wizard, and seed the connector's content:

1. **Connect a workspace** — add the Power BI workspace + report + dataset that the connector should expose.
2. **Configure analyses** (optional) — the connector ships the runtime + admin UI; analyses are configured per customer via the `/admin` UI after install. You can author them yourself or work with Expecta engineering on a separate engagement.
3. **Add reconciliation rules** (optional) — give the connector "ground-truth" reference values so it can flag when its answers diverge from your audited numbers.
4. **Add AI Context** (optional) — domain-specific terminology, KPI definitions, business glossary that help the connector answer in your company's language.

## Updates

When Expecta publishes a new connector version, you'll see an "Update available" indicator in the Marketplace managed resource. The update flow re-runs the install template against the latest image SHA — your data and configuration are preserved.

## Uninstall

Deleting the Managed Application deletes the entire managed resource group: connector, database, vault, registry, audit workspace (if it was created by the install). Soft-deleted KV cert + secrets and SQL DB will persist for 30–90 days per Azure defaults, then purge.
