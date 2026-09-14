# Terms of service — Expecta Connector

*Effective 2026-09-14. Maintained by Expecta. Source of truth: this file in [github.com/expecta-team/expecta-connector-docs](https://github.com/expecta-team/expecta-connector-docs).*

These terms describe how the Expecta Connector may be used. They apply alongside the agreement between your organisation and Expecta; **where the two differ, your agreement governs.**

## What the service is

The Expecta Connector lets people in your organisation ask questions about your own Power BI reports in plain language, from an assistant such as Claude or Microsoft Copilot. It reads your reports to answer a question. It does not write to your reports, your semantic models or your data warehouse.

The connector is available in two forms, and both are covered here:

- the **hosted service** Expecta operates at `expecta.app`
- the **Azure Managed Application** you install into your own Azure subscription from the Azure Marketplace

## Who may use it

Access to the hosted service requires that Expecta has enabled your organisation on it. Enablement follows from your agreement with us; it is not self-service, and it can be withdrawn if the agreement ends.

Separately, each person using the connector needs their own Microsoft licences — a Power BI licence, and whatever their assistant requires. Those are between your organisation and Microsoft, and the connector cannot substitute for them.

## What you are responsible for

- **Permissions.** What any person sees through the connector is decided by their Power BI access and by the row-level security in your reports. You control both. The connector cannot grant access your own permissions do not already allow, and it is not a substitute for managing them.
- **Your users.** You decide who in your organisation may use the connector, and you remain responsible for their use of it.
- **What you configure.** The business context, reference figures and saved analyses your administrators author are stored as written. Treat that configuration the way you would treat any document you place with a supplier.
- **Your credentials.** Do not share sign-in credentials between people. Every request is attributed to the identity that made it, and shared credentials make that attribution meaningless.

## What Expecta is responsible for

- Operating the hosted service, and maintaining the security of the infrastructure it runs on.
- Handling your data as described in the privacy policy for the form you use — [hosted service](PRIVACY-HOSTED.md) or [Managed Application](PRIVACY.md).
- Telling you about material changes to how the service handles your data.
- Responding to support requests through the channels in [SUPPORT.md](SUPPORT.md).

## Acceptable use

Do not:

- attempt to reach data belonging to another organisation, or to bypass the per-organisation and per-user access controls
- resell, sublicense or provide the connector as a service to a third party, unless your agreement with Expecta says you may
- reverse-engineer, decompile or attempt to extract the source of the hosted service
- use the connector to circumvent restrictions that apply to you in Power BI itself
- probe, load-test or scan the hosted service without arranging it with us first — tell us and we will help you do it properly

## Maintenance, updates and change

The hosted service is updated from time to time, which can briefly interrupt it. New capabilities are added and existing ones change; where a change affects how you use the connector, we will tell you. Any availability or response-time commitments are those set out in your agreement with Expecta.

For the Managed Application, updates are initiated by you on your side, from the Marketplace.

## Your data stays yours

Your reports, your figures and the configuration you author remain yours. Expecta claims no ownership of them, does not use them to train any model, and does not disclose them to a third party except where you ask us to or the law requires it.

## Ending access

Either party may end the arrangement in line with your agreement. When access to the hosted service ends, your organisation is removed from the service's enabled list and requests from it stop being answered. Your configuration is handled as described in the [hosted privacy policy](PRIVACY-HOSTED.md) — tell us if you want it deleted, and we will confirm when it is done.

Ending access to the hosted service does not affect a Managed Application install in your own subscription, which remains yours to keep or remove.

## Changes to these terms

Material changes will be reflected in a new commit to this file. The git history at [github.com/expecta-team/expecta-connector-docs](https://github.com/expecta-team/expecta-connector-docs) is the authoritative changelog. Where these terms and your agreement with Expecta differ, your agreement governs.

## Contact

- **General and commercial**: Claudio Clemente — `claudioclemente@expecta.io`
- **Technical and security**: Waddah Alhajar — `waddah.alhajar@expecta.io` (prefix security reports with `[security]`)
