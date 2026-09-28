# Troubleshooting

Common issues during install and runtime, with their fixes.

## Install-time

### Wizard validation fails: "Owner or User Access Administrator on the subscription is required"

The install grants Azure roles to the connector's own managed identities, which requires Owner or User Access Administrator. Either request the role from your subscription owner for the duration of the install, or have someone with the role run the install on your behalf.

### Wizard validation fails: "Resource name prefix must start with a letter; 3–12 lowercase letters and digits only"

The prefix is used to derive Azure resource names that have stricter naming rules than the wizard's regex (e.g. ACR doesn't allow hyphens). Pick something like `acme`, `lilla`, `mcp01`.

### Deployment fails: "Cert provisioning did not complete within 5 minutes"

The `create-oauth-cert.ps1` deployment script waits up to five minutes for Key Vault to issue the certificate, and fails the install if it doesn't. This is usually a transient Key Vault delay: delete the failed managed application and install again. If it happens repeatedly, open a support ticket with the deployment's correlation ID.

### Image import fails or times out: deployment script fails on `az acr import`

The install copies the connector image from Expecta's registry into your subscription's registry. If your tenant's egress policy blocks outbound connections to Microsoft Container Registry or Expecta's registry, this fails.

Fix: allow outbound HTTPS to `mcr.microsoft.com` and `expectaregistry.azurecr.io`, then install again. **Keep that access open:** the nightly update job uses the same connection to fetch new versions. If you block it again, the connector keeps running but stops updating, and each night's run logs the failure.

## After install: finish setup

### The finish-setup command fails with "Insufficient privileges" or 403

The person running it needs **Cloud Application Administrator** (or Application Administrator / Global Administrator) in Microsoft Entra ID **and** **Owner** (or **Contributor** plus **User Access Administrator**; User Access Administrator alone can't write the tags) on the resource group that holds the managed application. The first covers creating the app registration and granting consent; the second covers tagging the managed application and giving the connector read access to those tags. Have a person with both run it, or run it once with the missing role added. The script is safe to re-run.

### Sign-in still doesn't work a few minutes after finish-setup

The connector checks for the finish-setup result and restarts itself with sign-in on, usually within about two minutes. If it's still off after ten minutes:

- Check the managed application's **Tags**: `expecta-connector-client-id` and `expecta-connector-admin-oid` should both be present. If not, the script didn't finish; re-run it and read its last message.
- Re-run the command from the managed application's **Outputs**. It updates the same app registration rather than creating a second one.
- If both tags are present and sign-in is still off, contact support with the time you ran the script.

## Runtime

### MCP client says "401 Unauthorized" on every request

Check the user's bearer token:
- Is it within its expiry window? Expired tokens are rejected with a 401 that tells the client to refresh; MCP clients normally do this silently. If every request fails, sign out of the connector in the client and sign in again.
- Does the JWT's `tid` claim match a tenant in the install's allowlist? Check the wizard's "Allowed tenant IDs" setting and add the user's tenant if necessary.
- Is the user member of any Power BI workspace? The connector lets users authenticate but can't show them data they don't have Power BI permissions for.

### MCP client says "403 tenant_not_allowed"

The user's token comes from a tenant not in the connector's allowlist, which was set in the install wizard. The allowlist lives in the managed resource group, which Azure protects from changes; contact Expecta to add a tenant.

### Queries return "workspace not configured"

The workspace + dataset binding hasn't been added to the connector's config DB yet. Sign into `/admin/login` and add it via the "Workspaces" tab.

### Queries hang for >30 seconds then fail

Several possible causes:
- **SQL config DB auto-paused.** Serverless SQL pauses after 60 minutes of inactivity; the first query after a long idle period takes ~30 seconds to resume the DB. Re-run the query after the resume completes.
- **Power BI is slow to answer.** The underlying query may be expensive. The audit log records which report and dataset were reached and how long the pattern has persisted; reproduce the same question against the model in Power BI Desktop to see where the time goes. Heavy queries may need rewriting or pre-materialising as a measure.
- **Container App scaling event.** If you've enabled scale-to-zero, the first request after idle warms up a new container (~3–5 seconds). Subsequent requests are fast.

### Specific user can use the connector through their LLM client but the `/admin` UI returns 403

The `/admin` UI requires the user's Entra Object ID to be in the connector's administrator list. The connector itself only needs the user to have Power BI workspace access. The first administrator is the person who ran the finish-setup command; to add more, contact Expecta for now (managing administrators from `/admin` is planned).

### Cert renewal failing or imminent expiry warning from Microsoft

The OAuth client cert is 3-year self-signed at install. If you're at the 35-month mark, see [Operations § Certificate rotation](./OPERATIONS.md#certificate-rotation) for the renewal procedure.

### The nightly update logs "registry … refused the stored token"

The registry access stored in your install has been disabled or withdrawn, so the job can no longer fetch new versions. The connector keeps running the version it has. Contact Expecta.

### The nightly update logs "rolled back"

A new version started but failed its health check, so the job moved the connector back to the previous version automatically. Nothing is needed from you to keep working; send the log line to support so we can look at why.

## Diagnostics quick reference

| What you want to know | Where to look |
|---|---|
| Is the connector running? | Azure Portal → Container App → "Logs" tab, or the connector's container logs in your Log Analytics workspace |
| Why is a question failing? | Your Log Analytics workspace — find the failed request by the reference number shown with the error |
| Did sign-in complete for a specific person? | Your Log Analytics workspace — sign-in entries record each attempt and its outcome |
| What did the connector ask Power BI for? | Your Log Analytics workspace records each data request and how many rows came back. The detail of the request itself is not stored unless you enable it — ask us how |
| Connector's current health endpoint? | `https://<connector-fqdn>/health` returns 200 OK + version |
| Did last night's update run, and what did it do? | Your Log Analytics workspace — search the update job's console logs for `nightly-update` (`up to date`, `updated`, or `rolled back`) |
| OAuth metadata? | `https://<connector-fqdn>/.well-known/oauth-protected-resource` |

If you've worked through the above and still can't move forward, see [Support](./SUPPORT.md).
