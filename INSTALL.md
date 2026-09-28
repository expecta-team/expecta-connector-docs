# Install guide

The Expecta Connector installs from the Azure Marketplace as a Managed Application offer. The install runs entirely inside *your* Azure subscription — the connector container, its configuration database, its Key Vault, and its Log Analytics workspace are all provisioned in a managed resource group in your tenant.

Setup has two parts: the **Marketplace install** (a short wizard), then **one command your Microsoft Entra administrator runs** to switch on sign-in. An Azure Marketplace install cannot create an app registration in your directory, so that one step is done by your own admin, with your own rights.

## Prerequisites

You'll need:

- **An Azure subscription** in the tenant where Power BI lives, with enough quota for a small set of resources (one Container App, one Container Apps job, one Key Vault, one Basic-tier Azure Container Registry, one serverless Azure SQL database, optionally one Log Analytics workspace).
- **Owner or User Access Administrator** on the subscription, for the person running the install: the install grants Azure roles to the connector's own identities. The wizard validates this up front.
- **The registry access name and password** Expecta sends you. The install uses them to copy the connector image into your own registry.
- **For the post-install step:** a person with **Cloud Application Administrator** (or Application Administrator / Global Administrator) in Microsoft Entra ID, who is also **Owner or User Access Administrator** on the resource group that holds the managed application. That person becomes the connector's first administrator.
- **Users with Power BI access** through their normal Power BI workspace membership. The connector runs each query as the signed-in user, so users who can't see a workspace in Power BI won't see it through the connector either.

## What the wizard collects

### Basics

| Field | Why |
|---|---|
| **Azure subscription / region / resource group** | Standard Azure resource selectors. We recommend a fresh resource group named after the install. |
| **Resource name prefix** | 3–12 lowercase letters and digits. Used to derive resource names: prefix `acme` produces `acme-mcp-<unique-suffix>`, `acmekv<suffix>`, etc. |

### Expecta registry access

The access name and password from Expecta. They let the install copy the connector image from Expecta's registry into the registry created in your subscription, and are stored in your own Key Vault so the connector can keep itself up to date (see [Updates](#updates)).

### Audit and telemetry

Audit events (tool invocations, OAuth handshakes, DAX queries) need a Log Analytics workspace to land in. Two choices:

- **Create new workspace in this Managed App** (default) — a fresh workspace is provisioned alongside the connector, with 365-day retention. Simplest.
- **Use my existing workspace** — paste the full resource ID of an existing workspace in your subscription (the workspace can be cross-region). Useful if you already centralise telemetry in one place.

### Access control

Only users from the tenants you list here can sign in to the connector. Default: your own tenant only — deny-by-default for everyone else. Add comma-separated tenant GUIDs if you want to extend access (e.g. for managed-service-provider scenarios where one operator administers connectors across multiple end-customer tenants).

### Custom domain (optional)

If you plan to expose the connector on a custom hostname (e.g. `mcp.acme.com`) instead of the auto-generated `<resourceName>.<region>.azurecontainerapps.io`, type the FQDN here. Its sign-in callback addresses are then included when the app registration is created in the post-install step, so once you bind the custom domain to the Container App (see [Operations](./OPERATIONS.md)), sign-in works on both hostnames without further changes.

The custom-domain DNS, TLS certificate, and Container Apps domain binding are separate steps — the wizard only prepares the sign-in side.

## What gets created in your subscription

Inside the managed resource group, the install provisions:

- **Container App** running the connector image, under its own **user-assigned managed identity** with narrow rights: pull from the per-install registry, read its secrets from Key Vault, read/write the config database, and read the managed application's tags.
- **Azure SQL serverless database** holding configuration (analyses, reconciliation rules, AI context, tenant admin allowlist). Microsoft Entra authentication only — no SQL passwords.
- **Key Vault** holding the OAuth client certificate and private key, the connector's own signing secrets, and the registry access name and password. RBAC mode, soft-delete (90 days) and purge-protection enabled.
- **Azure Container Registry (Basic tier)** holding a copy of the connector image, copied from Expecta's registry, so the connector always runs from your own registry.
- **A nightly update job** (a Container Apps job with its own identity) — see [Updates](#updates).
- **Log Analytics workspace** (only if you selected "Create new") for audit events, plus diagnostic settings that send the Key Vault, database and registry logs there.

The **Entra ID app registration** is not created by the install; it is created in the post-install step below, in your directory.

Ongoing Azure cost for a single-user install is approximately **$25–40/month**, dominated by the Basic registry (~$5), the serverless SQL DB (auto-paused most of the time, ~$5–15 depending on usage), and the Container App's consumption tier (scales to zero when idle, ~$10–20 with light usage). Costs scale with usage; high-traffic installs run higher.

## After install: finish setup (once)

Open the managed application in the Azure Portal. Its **Overview** page explains the step, and its **Outputs** contain the command, `finishSetupCommand`.

1. The person described in the prerequisites (Cloud Application Administrator + Owner on the resource group) opens **Azure Cloud Shell (PowerShell)** and runs that command. The command downloads the script from your connector and checks its SHA-256 against the value fixed in the install before running it. If the check fails, it deletes the download and stops without running anything.
2. The script, running with that person's own rights in your tenant:
   - creates the connector's app registration (sign-in, callback addresses, the `access_as_user` scope, Power BI read permissions, and Microsoft Graph *read all users' basic profiles*, used only to add another connector administrator by email);
   - attaches the connector's public certificate — the private key never leaves your Key Vault;
   - grants consent for those delegated permissions;
   - tags the managed application with the app's ID and your object ID, and gives the connector read access to those tags.
3. About two minutes later the connector restarts with sign-in on. **The person who ran the script is the connector's first administrator.**

Expecta gets no access at any point of this step. The script can be re-run safely; it updates the same app registration.

## Using the connector

The Overview page and Outputs show:

- **MCP endpoint URL** (`mcpEndpoint`) — paste this into your MCP client (Claude Desktop, Claude Code, etc.).
- **Connector admin URL** — `https://<connector-fqdn>/admin/login`.
- **Metrics** — request count and CPU.

The first administrator signs into `/admin` and seeds the connector's content:

1. **Connect a workspace** — add the Power BI workspace + report + dataset that the connector should expose.
2. **Configure analyses** (optional) — the connector ships the runtime + admin UI; analyses are configured per customer via the `/admin` UI after install. You can author them yourself or work with Expecta engineering on a separate engagement.
3. **Add reconciliation rules** (optional) — give the connector "ground-truth" reference values so it can flag when its answers diverge from your audited numbers.
4. **Add AI Context** (optional) — domain-specific terminology, KPI definitions, business glossary that help the connector answer in your company's language.

To add more administrators, contact Expecta for now; managing administrators from `/admin` itself is planned.

## Updates

The connector updates itself. Every night at 02:00 UTC a small job inside your managed resource group checks whether Expecta has published a new connector image:

- **No new version:** it does nothing.
- **New version:** it copies the image into your own registry, moves the connector to it, and checks that it is healthy. If the new version fails its health check, it rolls back to the previous one automatically.

Every run records its outcome ("up to date", "updated", or "rolled back") in your Log Analytics workspace. Your data and configuration are untouched by updates. The job uses the registry access you entered in the wizard; if Expecta disables that access, updates stop and the job logs the failure.

## Uninstall

Deleting the Managed Application deletes the entire managed resource group: connector, database, vault, registry, update job, and audit workspace (if it was created by the install). The app registration created in the post-install step stays in your directory; delete it from Microsoft Entra ID if you no longer need it. Soft-deleted Key Vault secrets and the SQL database persist for 30–90 days per Azure defaults, then purge.
