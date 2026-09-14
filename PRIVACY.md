# Privacy policy — Expecta Connector (Azure Managed Application)

*Effective 2026-06-04. Scope clarified 2026-09-14. Maintained by Expecta. Source of truth: this file in [github.com/expecta-team/expecta-connector-docs](https://github.com/expecta-team/expecta-connector-docs).*

This privacy policy describes how the **Expecta Connector** (the Azure Managed Application offer published by Expecta on the Azure Marketplace) handles customer data. It applies to **customer organisations who install the Managed Application offer into their own Azure subscription**.

> **This policy covers the Managed Application only.** It applies when the connector runs **inside your own Azure subscription**, installed from the Azure Marketplace — the model in which no customer data reaches Expecta at all.
>
> If instead your organisation uses the **hosted service Expecta operates at `expecta.app`**, connected to an assistant such as Claude or Microsoft Copilot, the policy that applies to you is [PRIVACY-HOSTED.md](PRIVACY-HOSTED.md). The two differ in substance, not just in wording — do not rely on this one for the hosted service.

## Summary

**No customer data is ever transmitted to or processed by Expecta.** The connector runs entirely inside the customer's Azure subscription; every byte of customer data (Power BI query results, audit events, configuration) stays inside the customer's tenant boundary.

## What the connector handles

When a Managed Application install runs inside the customer's Azure subscription, the connector handles three categories of information — all of which stay in the customer's tenant:

| Category | Where it lives | Who can read it |
|---|---|---|
| Customer's Power BI query results (workspace data, datasets, DAX query outputs) | In-process at the connector's Container App for the duration of one request, then discarded | Only the calling user (via On-Behalf-Of authorisation) |
| Audit log of tool invocations | Customer's own Azure Log Analytics workspace | Whoever the customer grants access to in their own subscription |
| Connector configuration (analyses, reconciliation rules, tenant-admin allowlist, Context — all customer-authored via the admin UI after install) | Customer's own Azure SQL serverless database | Same — customer-controlled |
| OAuth client certificate + private key | Customer's own Azure Key Vault | Same — customer-controlled |

The connector binary running in the Container App is the same image published by Expecta to the customer's per-install Azure Container Registry at install time. After install, the image is owned and pulled from within the customer's tenant.

## What Expecta sees about customer installs

Expecta has **no standing read or write access** to any installed instance of the connector or its data:

- **Production runtime** — Expecta does not hold any credential, key, certificate, or service-principal access to any customer's connector deployment, Azure SQL database, Key Vault, Container App, or Log Analytics workspace. The Entra ID application registration created during install lives in the customer's tenant and is owned by the customer's tenant admin.
- **Telemetry** — the connector emits no telemetry to Expecta-controlled endpoints. Application logs land in the customer's Container Apps log destination; audit events land in the customer's Log Analytics workspace.
- **Updates** — when Expecta publishes a new connector version through the Marketplace, the customer initiates the update on their side. Copying the new image into the customer's own container registry uses a credential issued for that step alone; Expecta retains no ongoing access to the customer's resources through it.
- **Diagnostics** — if a customer raises a support ticket and chooses to share log excerpts, those are sent through the customer-initiated support channel (email or shared issue). Expecta does not pull diagnostics on its own.

## Personal data

The connector does not collect, process, or store personally identifiable information (PII) for any purpose other than user authentication:

- **User authentication** — the calling user's Microsoft Entra ID access token is verified at each request to confirm the caller is in the customer's tenant allowlist. The token is held in memory for the duration of the request and discarded. The user's Object ID (`oid` claim) is written to the audit event so the customer can attribute connector activity to specific users — this record lands in the customer's own Log Analytics workspace, not Expecta's.
- **First-admin seeding** — at install time the customer admin provides an email address. The installer resolves the email to a Microsoft Entra Object ID via Microsoft Graph and writes one row into the customer's configuration database. This data never leaves the customer's tenant.

The connector does not transmit any user identifier, email, or query content to Expecta-controlled servers under any circumstance.

## Marketplace publisher relationship

When a customer purchases the offer through the Azure Marketplace, the following information flows to Expecta as the publisher (this is standard for any Azure Marketplace transaction and is governed by Microsoft's commercial marketplace publisher agreement, not Expecta):

- The customer's organisation name and primary technical contact email, surfaced through Partner Center's lead-routing feature.
- Subscription billing identifiers, used by Microsoft for invoicing.
- High-level installation status (installed / updated / removed), surfaced through the Partner Center analytics dashboard.

Expecta uses this information solely for customer-support routing, invoicing reconciliation, and aggregate product-usage reporting. Expecta does not contact Marketplace customers for marketing purposes without their explicit opt-in.

## Data residency

The Managed Application installs all of its resources into the Azure region the customer selects at install time. No resource is created in any other region. The customer's data therefore resides in the geographic region of their choice, subject to Azure's regional infrastructure guarantees.

## Cookies, web tracking, and the Connector

The Marketplace listing pages on `azuremarketplace.microsoft.com` use cookies governed by Microsoft's privacy policy, not Expecta's. The Connector's own admin web UI (`/admin`) served from a customer's installed instance sets one session cookie for the admin login flow only — it lives entirely in the customer's tenant. No trackers, advertising pixels or analytics scripts are present in any Expecta-served code.

One third-party request is worth naming, and it is confined to a single place: the **browser-based administration pages** (`/admin`) load their web fonts from Google's font service, so a browser opening those pages makes a request to Google and Google sees its IP address. Google receives nothing else — no query, no figure, no identity. **The dashboards rendered inside your assistant do not make this request**, and everything else the interface needs — charts, tables, scripts — is served by the connector itself. We will serve the fonts locally on request.

## Sub-processors

A sub-processor is a third party that would handle your data on Expecta's behalf. **For the Managed Application there are none** — no Expecta-side service reaches your data at all, so there is nothing to pass on.

Microsoft handles the Marketplace purchase itself, under its own commercial marketplace publisher agreement rather than this policy.

## Contact

For privacy questions, data-subject requests, or security disclosures:

- **Primary contact**: Waddah Alhajar — `waddah.alhajar@expecta.io`
- **Secondary contact**: Claudio Clemente — `claudioclemente@expecta.io`
- **Security disclosures**: prefix subject with `[security]`; do not file in public channels.

## Changes to this policy

Material changes will be reflected in a new commit to this file. The git history at [github.com/expecta-team/expecta-connector-docs](https://github.com/expecta-team/expecta-connector-docs) is the authoritative changelog.
