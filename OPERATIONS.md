# Operations

Day-2 runbook for keeping a Managed Application install healthy.

## Custom domain (optional)

The install gives you an auto-generated Container Apps FQDN — something like `acme-mcp-abc123.italynorth.azurecontainerapps.io`. If you want a friendlier name (e.g. `mcp.acme.com`), bind a custom domain after install:

1. **DNS** — create a CNAME from `mcp.acme.com` to the auto-FQDN.
2. **Domain ownership** — add a TXT record `asuid.mcp` with the verification value from the Container App's "Custom domains" blade.
3. **TLS certificate** — either upload your own PFX to the Container Apps managed cert store, or let Azure-managed certs issue one (free, Let's Encrypt-backed, auto-renewing).
4. **Hostname binding** — in the Container App, "Custom domains" → "Add custom domain" → bind `mcp.acme.com` to the cert.

If you supplied the custom domain in the install wizard's optional step 4, the connector's OAuth app registration is already set up to accept callbacks on both the auto-FQDN and your custom domain — no further OAuth changes needed.

If you did *not* supply it at install time, you'll need to add the OAuth callback URIs manually in the Entra ID portal: navigate to the connector's app registration, "Authentication" → "Add a platform" → "Web" → enter `https://mcp.acme.com/oauth/callback` and `https://mcp.acme.com/dashboard/callback`.

## Certificate rotation

The OAuth client certificate generated at install time is valid for 3 years. Approaching renewal:

1. The deployment script that originally created the cert (`create-oauth-cert.ps1`) is idempotent and supports rotation. Re-running it against the same install will generate a new cert, attach it to the Entra app registration's `keyCredentials` (alongside the old one), and store the new private key in Key Vault.
2. After the new cert is verified working (connector continues to authenticate successfully), remove the old `keyCredential` entry from the Entra app registration to close out the rotation.

Day-to-day: you don't need to do anything until the 3-year mark approaches. Microsoft will email the tenant admin associated with the app registration when expiration is imminent.

## Scaling

The default install runs **one Container App replica** (min=1, max=1). For most installs this is sufficient — the connector spends most of its time waiting on Power BI XMLA responses, not on CPU. If you experience latency under high concurrent load:

1. **Vertical first.** Bump CPU/memory on the Container App revision: Container App → "Containers" → edit → increase resources from 0.5 vCPU / 1.0 Gi → 1.0 vCPU / 2.0 Gi.
2. **Horizontal only if vertical isn't enough.** The connector holds some session state in-memory; multi-replica scaling requires switching to a shared session backend (Redis or stateless JWT state tokens). Contact Expecta engineering before raising the replica count.

The Container App is configured to scale **to zero** between requests in the Consumption profile if you change min replicas to 0 — cold-start adds ~3–5 seconds to the first request after idle. Trade-off you can decide based on usage patterns.

## Updates

When Expecta publishes a new connector version, the Marketplace offer surface shows an "Update available" indicator on the Managed Application resource. To apply:

1. Resource → "Update" → review the version notes and pricing impact (typically none).
2. Click "Update". The install template re-runs against the new image SHA: `az acr import` pulls the new image into your ACR, the Container App revision flips, and your configuration is preserved (config DB, KV, audit history all untouched).
3. The update takes ~3–5 minutes; in-flight requests will see a brief 503 during the revision flip.

Major version bumps may include backward-incompatible schema changes; the release notes will call these out explicitly and the update template handles the migration idempotently. If anything goes wrong during the update, Container Apps' built-in revision history lets you roll back to the previous revision instantly.

## Backup and disaster recovery

The install's data plane is the config DB (Azure SQL) and the audit log (Log Analytics). Recovery strategy depends on your standards:

- **Config DB.** Azure SQL serverless has automated point-in-time backups (7-day default retention). Restore via the SQL portal blade. Long-term retention can be enabled in the SQL portal if your compliance regime requires more than 7 days.
- **Audit log.** Log Analytics retention is set to 365 days at install time. Beyond that, configure Log Analytics' export rules to ship audit events to Azure Storage (cheap long-term retention) or your SIEM.
- **Key Vault.** Soft-deleted certs and secrets persist for 90 days after deletion before purging — recoverable by anyone with KV access through the standard "Soft deleted" blade.
- **Connector image.** Always re-pullable from Expecta's publisher registry at install time; no DR responsibility for the image itself.

Region failover: the install is deployed to a single Azure region. Cross-region DR isn't currently part of the offer — if your compliance regime requires it, contact Expecta engineering and we can discuss a multi-region split.

## Monitoring

The "Metrics" tab in the Managed Application blade shows the headline numbers:

- **Request count** — how many MCP calls land at the connector.
- **CPU and memory utilisation** — for capacity planning.
- **Replica count** — should be 1 in the default config.

For deeper monitoring, query the audit Log Analytics workspace directly. Sample KQL:

```kql
// Last 100 tool invocations with outcome
AppTraces
| where AppRoleName == "powerbi-online-mcp"
| where Properties.event == "tool.invoke"
| project TimeGenerated,
          tool = Properties.tool_name,
          user_oid = Properties.user_oid,
          workspace = Properties.workspace_id,
          outcome = Properties.outcome,
          duration_ms = Properties.duration_ms
| top 100 by TimeGenerated desc
```

```kql
// Error rate per tool over the last day
AppTraces
| where AppRoleName == "powerbi-online-mcp"
| where Properties.event == "tool.invoke"
| where TimeGenerated > ago(1d)
| summarize total = count(),
            errors = countif(Properties.outcome == "error")
            by tool = tostring(Properties.tool_name)
| extend error_rate = round(100.0 * errors / total, 2)
| order by error_rate desc
```

For installs feeding into a SOC's SIEM, see the "Audit log routing" section in your tenant's overall observability runbook.

## Troubleshooting

See [Troubleshooting](./TROUBLESHOOTING.md) for install-time and runtime issues with their fixes.
