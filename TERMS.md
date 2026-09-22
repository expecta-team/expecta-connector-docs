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

## Expecta's access to your deployment

The Managed Application is deployed into your Azure subscription, into a managed resource group. Azure grants Expecta a **Contributor** role on that resource group, and only on that resource group. The grant is held by a single Microsoft Entra group, “Expecta Connector Maintainers”, whose membership is limited to Expecta staff responsible for the connector.

**What it is for.** Shipping you a new connector version, and recovering an install that cannot recover itself — for example, restoring administrator access when the first-admin record was not created at install time.

**Why an update needs us.** Your connector runs an image held in your own container registry. A new version has to be copied in and a new revision started. That is a deliberate design choice: nothing we publish reaches your environment until that step is taken, so a change on our side can never alter a running install of yours without action.

**What it does not grant.**

- It does not grant access to your business data. The connector reads Power BI as the signed-in user; secrets live in your own Key Vault and are read at runtime by the connector's managed identity.
- It does not grant access to your Azure subscription outside the managed resource group.

**What you see.** Every action Expecta takes appears in your Azure Activity Log, attributed to the individual who performed it. You can review it at any time, and configure alerts on it.

**Your own access.** Azure applies a deny assignment to the managed resource group, which is how a Managed Application protects its own resources from being changed out from under it. This is created and controlled by Azure, not by Expecta, and it cannot be edited or removed — by you or by us. If you want full control of the underlying resources rather than a managed deployment, tell us and we will discuss the alternatives.

**Ending it.** Expecta's access exists for as long as the Managed Application does, and ends when you delete it.

## Acceptable use

Do not:

- attempt to reach data belonging to another organisation, or to bypass the per-organisation and per-user access controls
- resell, sublicense or provide the connector as a service to a third party, unless your agreement with Expecta says you may
- reverse-engineer, decompile or attempt to extract the source of the hosted service
- use the connector to circumvent restrictions that apply to you in Power BI itself
- probe, load-test or scan the hosted service without arranging it with us first — tell us and we will help you do it properly

## Maintenance, updates and change

The hosted service is updated from time to time, which can briefly interrupt it. New capabilities are added and existing ones change; where a change affects how you use the connector, we will tell you. Any availability or response-time commitments are those set out in your agreement with Expecta.

For the Managed Application, a new version does not reach your install by itself. Updates are applied by Expecta to your deployment, and appear in your Azure Activity Log — see “Expecta's access to your deployment”.

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
