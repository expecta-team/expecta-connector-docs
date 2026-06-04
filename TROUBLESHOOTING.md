# Troubleshooting

Common issues during install and runtime, with their fixes.

## Install-time

### Wizard validation fails: "Owner or User Access Administrator on the subscription is required"

The install creates an Entra app registration and several role assignments — these require subscription-level privileged roles. Either request the role from your subscription owner for the duration of the install, or have someone with the role run the install on your behalf.

### Wizard validation fails: "Resource name prefix must start with a letter; 3–12 lowercase letters and digits only"

The prefix is used to derive Azure resource names that have stricter naming rules than the wizard's regex (e.g. ACR doesn't allow hyphens). Pick something like `acme`, `lilla`, `mcp01`.

### Deployment fails: cert provisioning didn't complete in 5 minutes

The `create-oauth-cert.ps1` deployment script polls Key Vault for the cert to finish issuing. If the wait times out, the install proceeds but the connector won't be able to authenticate to Entra ID. To recover: re-run the install template (it's idempotent — existing resources are detected and reused; the cert script picks up where it left off).

If repeated re-runs fail with the same symptom, Key Vault may be experiencing a regional issue — try again in 15 minutes, or open a support ticket.

### Deployment script log: "Could not attach cert to Entra app via Graph"

This means the cert was created in Key Vault successfully, but the Microsoft Graph PATCH to attach the cert's public key to the app registration's `keyCredentials` failed — typically because admin consent for the installer's `Application.ReadWrite.OwnedBy` permission hasn't propagated by the time the script ran.

To complete the install manually: see the post-install runbook in the Expecta engineering team's onboarding email, or contact support — it's a one-line PowerShell command using a user with admin privileges on the Entra app.

### Deployment script log: "Graph lookup of `<admin email>` failed"

The admin email seeded in the config database needs to resolve to an Entra Object ID. If the script can't reach Graph (consent not propagated, network issue), the install completes but the connector's `/admin` web UI refuses login.

Workaround: have any Power BI workspace owner sign into `/admin/login` — they'll be added to the admin allowlist on first successful login, after which they can promote additional admins through the UI.

### Image import timed out: deployment script fails on `az acr import`

The install copies the connector image from Expecta's publisher registry into your subscription's ACR. If the deployment script's outbound connection to either Microsoft Container Registry or Expecta's registry is blocked by your tenant's egress policy, this fails.

Fix: temporarily unblock outbound access to `mcr.microsoft.com` and `expectaregistry.azurecr.io` for the install IP range, re-run, then re-block after the image is in your ACR (the connector pulls from *your* ACR at runtime, not Expecta's).

## Runtime

### MCP client says "401 Unauthorized" on every request

Check the user's bearer token:
- Is it within the 1-hour expiry window? Tokens older than ~55 minutes are rejected.
- Does the JWT's `tid` claim match a tenant in the install's allowlist? Check the wizard's "Allowed tenant IDs" setting and add the user's tenant if necessary.
- Is the user member of any Power BI workspace? The connector lets users authenticate but can't show them data they don't have Power BI permissions for.

### MCP client says "403 tenant_not_allowed"

The user's token comes from a tenant not in the connector's allowlist. Add the tenant ID via the Managed Application's configuration parameters (re-run the install template with an updated `tenantAllowlist` value).

### Queries return "workspace not configured"

The workspace + dataset binding hasn't been added to the connector's config DB yet. Sign into `/admin/login` and add it via the "Workspaces" tab.

### Queries hang for >30 seconds then fail

Several possible causes:
- **SQL config DB auto-paused.** Serverless SQL pauses after 60 minutes of inactivity; the first query after a long idle period takes ~30 seconds to resume the DB. Re-run the query after the resume completes.
- **Power BI XMLA endpoint slow.** The query may be expensive — check the audit log for the actual DAX text and run it directly in Power BI Desktop or DAX Studio to validate. Heavy queries may need to be rewritten or pre-materialised as a measure.
- **Container App scaling event.** If you've enabled scale-to-zero, the first request after idle warms up a new container (~3–5 seconds). Subsequent requests are fast.

### Specific user can use the connector through their LLM client but the `/admin` UI returns 403

The `/admin` UI requires the user's Entra Object ID to be in `config.admin_users`. The connector itself only needs the user to have Power BI workspace access. Add the user as an admin: sign into `/admin` as an existing admin → "Admins" tab → "Add admin" → enter the email.

### Cert renewal failing or imminent expiry warning from Microsoft

The OAuth client cert is 3-year self-signed at install. If you're at the 35-month mark, see [Operations § Certificate rotation](./OPERATIONS.md#certificate-rotation) for the renewal procedure.

### Deployment failure during update: "Image SHA not found"

If you've enabled a Container Apps cleanup policy that purges old revisions, an update can fail if your ACR has also been GC'd and lost the in-flight image. Re-run the update — the `az acr import` step re-pulls the image and the update can proceed.

## Diagnostics quick reference

| What you want to know | Where to look |
|---|---|
| Is the connector running? | Azure Portal → Container App → "Logs" tab; `ContainerAppConsoleLogs` table in your Log Analytics workspace |
| Why is a tool call failing? | Log Analytics → `AppTraces` → filter on `Properties.event == "tool.invoke"` and `Properties.outcome == "error"` |
| Did OAuth complete for a specific user? | Log Analytics → filter on `Properties.event startswith "oauth"` and `Properties.user_oid == "<oid>"` |
| What DAX did the connector send to Power BI? | Log Analytics → filter on `Properties.event == "dax.query"` |
| Connector's current health endpoint? | `https://<connector-fqdn>/health` returns 200 OK + version |
| OAuth metadata? | `https://<connector-fqdn>/.well-known/oauth-protected-resource` |

If you've worked through the above and still can't move forward, see [Support](./SUPPORT.md).
