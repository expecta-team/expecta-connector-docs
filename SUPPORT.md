# Support

The Expecta engineering team supports Marketplace deployments of the connector. Contact paths in order of urgency:

| Situation | Contact |
|---|---|
| Install-time blocker or production-down | Email **waddah.alhajar@expecta.io** and **claudioclemente@expecta.io** with subject prefix `[urgent]`; include your Azure subscription ID, region, and the deployment correlation ID from the Marketplace install operation |
| Bug report or feature request | Email **waddah.alhajar@expecta.io** with reproduction steps and audit-log excerpts if available |
| Security report | Email **waddah.alhajar@expecta.io** with subject prefix `[security]`; do not file in public channels |
| General questions / sales | Email **claudioclemente@expecta.io** |

## Information helpful to include

For technical issues, the more of the following you can include, the faster we can diagnose:

- **Install ID** — visible on the Marketplace Managed Application resource page.
- **Region** — where the install is deployed.
- **Connector version** — visible at `https://<connector-fqdn>/health`.
- **Time window** — when the issue started, in UTC.
- **Audit log excerpt** — relevant rows from your Log Analytics workspace (`AppTraces` table, filtered by `req_id` or time window).
- **Reproduction steps** — what tool was called, with what inputs, expected vs actual behaviour.

Do **not** include:
- Bearer tokens, OAuth client secrets, KV secret values, SQL connection strings.
- Customer-data rows from query results (description of the issue is sufficient).

## Response time

Best-effort during European business hours (Italy / CET). Production-down issues are typically acknowledged within 2 hours during business hours and within 12 hours outside them. Non-urgent issues see a response within 2 business days.

## Status and release notes

- Release notes for each Marketplace version are published in the Marketplace listing's "What's new" section.
- For a deeper technical changelog, contact engineering and we can share specific PR details.

## Feedback

The connector's developer-focused features are evolving rapidly based on customer use. If you're using it in a novel way or hitting limits we haven't accommodated, tell us — early-customer feedback is what drives the roadmap.
