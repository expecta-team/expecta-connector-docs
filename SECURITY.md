# Security posture

A summary of how the Managed Application offer is engineered to keep your data inside your tenant boundary and your users' permissions enforced.

## Data residency

- **Every customer-data byte stays in your tenant.** No customer data, query results, or audit events ever transit through an Expecta-controlled service. The connector's runtime lives in the Container App in your subscription, in the region you select at install.
- **The audit log lands in your Log Analytics workspace.** Retention is 365 days by default; configure exports to Storage / SIEM for longer retention if needed.
- **Configuration stays in your config DB.** Pre-built analyses, reconciliation rules, AI Context, admin allowlist — all in the SQL database in your subscription.

The only thing that crosses your tenant boundary is the connector container image, which is copied from Expecta's publisher registry into your per-install ACR at install time and at update time. After that, the connector pulls from your ACR.

## Authentication

The connector uses **certificate-based OAuth 2.1 client authentication** to talk to Entra ID — no client secret in transit, no client secret at rest:

- A self-signed RSA 2048 certificate is generated in your Key Vault at install time, valid for 3 years.
- The private key never leaves your Key Vault except for the connector's own runtime read via its system-assigned managed identity.
- Token requests to Azure AD use RFC 7521/7523 signed `client_assertion` JWTs — short-lived (10-minute exp), signed with the cert's private key.
- A leaked log line containing a signed assertion is useless 10 minutes later.

User-side authentication uses OAuth 2.1 with mandatory PKCE (S256), 10-minute auth-session TTL with automatic cleanup, and zero-leeway JWT signature/expiry validation on every request.

## Authorisation (per-user)

The connector enforces user permissions through the [On-Behalf-Of flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow):

- Every MCP call carries the calling user's bearer token.
- The connector exchanges that token for a Power BI access token *on the same user's behalf*.
- Power BI enforces that person's own workspace and dataset permissions on the resulting query.

Practical consequence: **the connector cannot read a workspace or dataset the calling user lacks Power BI permissions for.** Adding a user to a Power BI workspace grants them connector access for that workspace; removing them revokes it. There is no second-layer permission model to manage.

## Tenant isolation

The install enforces a deny-by-default tenant allowlist:

- The wizard's "Allowed tenant IDs" field defaults to your own tenant ID.
- The connector returns `403 tenant_not_allowed` for any JWT whose `tid` claim isn't in the allowlist, *before* any OBO exchange is attempted.
- The list is editable per-install if you need to extend access (e.g. managed-service-provider scenarios).

The connector also pins itself to your tenant via a one-row `config.tenant_info` table — the admin web UI refuses to serve requests whose JWT `tid` doesn't match. Belt and braces on multi-tenant misrouting.

## Audit

Every meaningful action is recorded in your own Log Analytics workspace, each entry carrying who did it, when, and a reference number you can quote to support:

- **Every question asked** — who asked it, which report and dataset it reached, and whether it succeeded.
- **Every sign-in** — successful and failed, with the address it came from.
- **Every access decision** — each time someone is granted or refused a workspace.
- **Every data request sent to Power BI** — recorded as *that* a request ran and how many rows came back, never the data itself. The technical detail of the request is not stored by default; it can be enabled if you want that depth.
- **Every change made in the settings panel** — who changed which setting, and when. The previous and new values are kept in your configuration database's own change history rather than in the log.

Every entry carries the same reference number for a given action, so one person's request can be followed from sign-in through to the answer. Ask us if you need the record format for SIEM ingestion.

## Network posture

- **TLS 1.2+ only.** Container Apps enforces minimum TLS 1.2 on inbound. SQL is configured with `minimalTlsVersion=1.2`.
- **Public surface kept to the connector itself.** The Container App's ingress is public HTTPS only (port 443). The SQL database restricts inbound to Azure services; the Key Vault's endpoint is publicly reachable, with Microsoft Entra authentication and Azure RBAC as the access control rather than a network boundary. Private Endpoint hardening is on the roadmap; see [Operations](./OPERATIONS.md#network-hardening) for the current state.
- **Strict CORS.** No `*` + credentials. Only origins you explicitly allowlist receive CORS headers.
- **Rate limiting.** Per-IP sliding window on auth endpoints (`/authorize`, `/token`, `/register`, `/dashboard/login`) and a per-user limit on `/mcp`. Tunable via env if your usage warrants different thresholds.
- **Request size cap.** 1 MiB default; oversized requests return 413.
- **Security headers.** `HSTS`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, `Content-Security-Policy` on HTML responses.

## DAX safety

User queries can drive DAX through the connector. A denylist suppresses noisy or expensive constructs (`SAMPLE`, `RANK` over unbounded sets, etc.) before the query reaches Power BI. The denylist is framed as *noise suppression* — Power BI's own per-query timeout and per-user permission boundary are the actual security gates.

## Secret management

- **No plaintext secrets in environment variables.** All sensitive material — OAuth cert, database connection details, OAuth signing material — is stored as Key Vault secrets and retrieved at runtime via the connector's system-assigned managed identity.
- **Key Vault hardened.** RBAC-mode authorisation, soft-delete and purge-protection enabled, 90-day soft-delete retention.
- **No long-lived service-principal client secret.** Per the cert-based authentication model above, the Entra app registration carries a `keyCredential` (public cert), not a `passwordCredential` (secret).

## What you can verify yourself

Several aspects of the security posture are independently verifiable from your side:

- **DB principals.** Connect to the config DB via SSMS or `sqlcmd`; `SELECT * FROM sys.database_principals` will show exactly one external user — the connector's Container App managed identity — with `db_datareader` + `db_datawriter`. Nothing Expecta-controlled.
- **Entra app credentials.** Look at the connector's app registration in the Entra ID portal → "Certificates & secrets" → only a `keyCredential` (cert), no `passwordCredentials`.
- **Audit log content.** Open your Log Analytics workspace and look at the connector's container logs. Everything listed above is there in full, in your storage, owned by you. Ask us if you would like ready-made queries for your reporting or SIEM tooling.
- **Container image provenance.** The Container App's image reference points at `<your-acr>.azurecr.io/expecta-mcp:<tag>`, not at Expecta's registry. The image is copied into your registry at install time and is pulled from there afterwards.

For deeper questions on threat model or compliance posture, contact Expecta engineering (see [Support](./SUPPORT.md)).
