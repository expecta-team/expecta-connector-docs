# Privacy policy — Expecta Connector (hosted service)

*Effective 2026-09-14. Maintained by Expecta. Source of truth: this file in [github.com/expecta-team/expecta-connector-docs](https://github.com/expecta-team/expecta-connector-docs).*

This privacy policy describes how the **hosted Expecta Connector** — the service Expecta operates at `expecta.app` and connects to assistants such as Claude and Microsoft Copilot — handles customer data. It applies to **organisations that Expecta has enabled on the hosted service**.

> **This is not the same offer as the Marketplace Managed Application.** If your organisation installed the connector into its own Azure subscription from the Azure Marketplace, the policy that applies to you is [PRIVACY.md](PRIVACY.md), and it differs in the most important respect: in that model nothing reaches Expecta at all. **In the hosted service described here, your query results do pass through infrastructure Expecta operates.** Read whichever of the two matches how you use the connector.

## Summary

The hosted connector runs in Expecta's own Azure subscription, in the **Italy North** region. Every question asked through it runs **as the person asking it**, using that person's own Microsoft sign-in, so the connector can only reach data that person is already permitted to see in Power BI.

Query results pass through the service to answer a request and are **not stored**. What Expecta does retain is your configuration, and an audit record of who asked what.

## Where the service runs

| | |
|---|---|
| Application | Azure Container App, **Italy North**, in Expecta's Azure subscription |
| Your configuration | Azure SQL database in the same subscription |
| Audit and application logs | Azure Log Analytics in the same subscription |
| Service credentials (certificates, keys) | Azure Key Vault in the same subscription |

The service runs entirely on Microsoft Azure. There are no analytics scripts, advertising pixels or third-party telemetry services in the hosted connector, and no figure or question is sent to any third party.

One third-party request is worth naming, and it is confined to a single place: the **browser-based administration pages** load their web fonts from Google's font service, so a browser opening those pages makes a request to Google and Google sees its IP address. Google receives nothing else — no query, no figure, no identity. **The dashboards rendered inside your assistant do not make this request**; everything they need is served by the connector itself. We will serve the fonts locally on request.

## What passes through, and what is kept

| Category | Kept? | Detail |
|---|---|---|
| **Power BI query results** — the figures, rows and report content returned to answer a question | **No** | Held in memory at the Container App for the duration of one request, returned to the assistant that asked, then discarded. No table in the service stores result rows. |
| **Your configuration** — saved dashboards and the queries behind them, reconciliation reference figures, Context entries, topics, the list of reports in scope, and your tenant-admin list | **Yes** | Stored per organisation in Expecta's Azure SQL database, with change history for the items you edit. All of it is authored by your own administrators through the admin interface. |
| **Audit record of activity** | **Yes, 30 days** | See below. |
| **Sign-in tokens** | **No** | The calling user's access token is verified per request, used to obtain a delegated Power BI token on that user's behalf, and discarded. No user token is written to storage. |

A point worth being precise about: when you save a dashboard, the service stores **the question, not the answer** — the query definition is kept and re-run when someone opens the dashboard. The figures themselves are never retained.

## The audit record

Every request is logged. Each entry can contain:

- the identity of the person who made the request — their Microsoft Entra object identifier, and in application log lines their work email address
- your organisation's Microsoft Entra tenant identifier
- which capability was invoked, with what parameters, and whether it succeeded or failed
- the Power BI workspace and dataset identifiers involved
- timestamp, request identifier, and the originating network address at sign-in

**Retention is 30 days**, in Expecta's Azure Log Analytics workspace, after which entries age out. Audit entries are visible to Expecta engineering staff for support and security purposes.

We can use this record to demonstrate, on request, that a given person's questions ran under their own identity — which is the check we ask every organisation to perform during setup.

## What Expecta can and cannot reach

- **No standing access to your data.** Expecta holds no credential that reads your Power BI content. Every read happens under a signed-in user's own delegated token, for the duration of their request.
- **Row-level security still applies.** If your reports restrict a person to one region, company or cost centre, the connector returns only that. The connector cannot widen what a person can see.
- **Access is gated per organisation.** The service answers only organisations Expecta has explicitly enabled. If that entry is removed, requests from your organisation are refused before any data is touched — nothing needs to be uninstalled on your side.

## The assistant you use is your own relationship

The connector answers a question that an assistant — Claude, Microsoft Copilot, or another Model Context Protocol host — sends on a user's behalf. **The conversation itself, and whatever the assistant retains of it, is governed by that provider's terms and privacy policy, not by this one.** Expecta neither controls nor receives a copy of your chat history. If your organisation needs to know how long an assistant keeps a conversation containing your figures, that question belongs to whoever provides the assistant.

## Personal data

The connector processes personal data only to identify and authorise the person asking:

- **Authentication** — the calling user's Entra ID token is verified per request to confirm they belong to an enabled organisation.
- **Attribution** — the user's object identifier is written to the audit record so activity can be traced to a person; application log lines can also contain their work email address. Both age out with the 30-day retention.
- **Administrator seeding** — when your organisation is set up, an administrator's email address is recorded so the admin interface can recognise them.

No question text, report content or figure is transmitted to any Expecta-controlled endpoint beyond the service itself, and none of it is used to train any model.

## Retention and deletion

Audit and application log entries age out after **30 days**.

Your configuration is retained for as long as your organisation is enabled on the service, and is deleted when you ask us to delete it. There is no automatic expiry: nothing removes your configuration on a timer, so if you want it removed, tell us and we will remove it. We will confirm when it is done.

## Your organisation's own obligations

Two things sit with you rather than with us, because only you can control them:

- **Who can ask.** Membership of the Power BI workspaces and the row-level security rules in your reports determine what any person sees through the connector. We cannot expand that, and we do not manage it for you.
- **What you put into the configuration.** The Context entries and reference figures your administrators author are stored as written. If you would rather something not sit in Expecta's database, do not enter it there.

## Security reporting

Report a suspected vulnerability by email with the subject prefixed `[security]`. Please do not open a public issue.

## Sub-processors

Microsoft Azure hosts the service, in the region named above. Expecta engages no other sub-processor that can reach customer data through the hosted connector.

## Contact

For privacy questions, data-subject requests, or security disclosures:

- **Primary contact**: Waddah Alhajar — `waddah.alhajar@expecta.io`
- **Secondary contact**: Claudio Clemente — `claudioclemente@expecta.io`
- **Security disclosures**: prefix the subject with `[security]`; do not file in public channels.

## Changes to this policy

Material changes will be reflected in a new commit to this file. The git history at [github.com/expecta-team/expecta-connector-docs](https://github.com/expecta-team/expecta-connector-docs) is the authoritative changelog.
