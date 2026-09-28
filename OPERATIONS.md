# Operations

Day-2 runbook for keeping a Managed Application install healthy.

## Custom domain (optional)

The install gives you an auto-generated address, something like `acme-mcp-abc123.italynorth.azurecontainerapps.io`. If you'd rather use your own name (e.g. `mcp.acme.com`), Expecta sets it up with you after the install. You can't bind it yourself: the managed resource group is locked against changes by anyone but the publisher, which is how a managed application works.

1. **Your DNS.** Create a CNAME from `mcp.acme.com` to the auto-generated address, and a TXT record `asuid.mcp` holding the Container App's domain verification ID. You can read the ID yourself: the Container App's **Custom domains** page, or `az containerapp show --name <app> --resource-group <managed RG> --query properties.customDomainVerificationId -o tsv`.
2. **Tell Expecta** the name. Through the publisher's management access to the managed resource group, we add the domain to the Container App with a free Azure-managed certificate (issued and renewed automatically), and point the connector's sign-in at the new address.
3. **Your app registration.** An administrator adds the two sign-in addresses to the connector's app registration in Microsoft Entra ID (**App registrations** → *Expecta Connector (…)* → **Authentication** → **Web** redirect URIs): `https://mcp.acme.com/oauth/callback` and `https://mcp.acme.com/dashboard/callback`. The app registration is yours, in your tenant, so Expecta can't do this step.
4. **Reconnect your AI assistants** with the new URL, `https://mcp.acme.com/mcp`. Sign-in answers on one address at a time, so the old one stops working for sign-in once the switch is made.

## Certificate rotation

The OAuth client certificate generated in your Key Vault at install time is valid for 3 years. Day-to-day you don't need to do anything. Microsoft notifies the app registration's owners when a credential is close to expiry.

Renewal is not automatic yet. Before the certificate expires, contact Expecta and we will renew it with you:

1. A new certificate is issued in your Key Vault (the private key never leaves it).
2. Your administrator re-runs the finish-setup command from the managed application's Outputs. The script adds the new public certificate to the app registration **next to** the old one, so sign-in keeps working throughout.
3. Once the connector is confirmed signing in with the new certificate, the old entry is removed from the app registration's *Certificates & secrets*.

## Scaling

The default install runs **one Container App replica** (min=1, max=1). For most installs this is sufficient — the connector spends most of its time waiting on Power BI, not on CPU. If you experience latency under high concurrent load:

1. **Vertical first.** Bump CPU/memory on the Container App revision: Container App → "Containers" → edit → increase resources from 0.5 vCPU / 1.0 Gi → 1.0 vCPU / 2.0 Gi.
2. **Horizontal only if vertical isn't enough.** The connector holds some session state in-memory; multi-replica scaling requires switching to a shared session backend (Redis or stateless JWT state tokens). Contact Expecta engineering before raising the replica count.

The Container App is configured to scale **to zero** between requests in the Consumption profile if you change min replicas to 0 — cold-start adds ~3–5 seconds to the first request after idle. Trade-off you can decide based on usage patterns.

## Updates

**New connector versions install themselves.** A Container Apps job in the managed resource group runs every night at 02:00 UTC:

1. It compares the connector image you are running with the latest one Expecta has published. If they are the same, it stops there.
2. If there is a new version, it copies the image into your own registry and starts a new revision of the connector on it (pinned by image digest).
3. It waits for the new revision to be ready and healthy. If it is not, it moves the connector back to the previous image.

An update takes a few minutes; requests in flight during the switch may see a brief error and succeed on retry. Your configuration, secrets and audit history are untouched.

**Checking what happened.** Each run writes one line to your Log Analytics workspace: `up to date`, `updated <old digest> -> <new digest>`, or `rolled back`. Search the job's console logs for `nightly-update`.

**Holding updates.** If you need a freeze (for example during a quarter-end close), tell Expecta and we will pause updates for your install until you say so.

Changes to the install's Azure resources themselves (a new plan version of the offer) still arrive as an "Update available" on the Managed Application resource, which you apply from the portal.

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

The nightly update job's runs are listed under the job in the managed resource group, and its outcome lines are in your Log Analytics workspace (see [Updates](#updates)).

For anything deeper, your Log Analytics workspace holds the connector's own audit records — every question asked, by whom, against which report, and whether it succeeded. That is where to look for usage patterns over time, which questions are failing, or the history behind a particular support reference number.

Ask us if you would like ready-made queries for your reporting or SIEM tooling; we will supply ones matching your deployment.

For installs feeding into a SOC's SIEM, see the "Audit log routing" section in your tenant's overall observability runbook.

## Troubleshooting

See [Troubleshooting](./TROUBLESHOOTING.md) for install-time and runtime issues with their fixes.
