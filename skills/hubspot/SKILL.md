---
name: hubspot
description: Analyze HubSpot CRM and marketing data through Windsor.ai. Use when the user asks about their HubSpot contacts, companies, deals, pipeline stages, deal value, owners, lead sources, or marketing email and campaign metrics (opens, clicks, conversions, revenue). This connector is read-only — it has no write actions and cannot create or edit records or send emails. Do not use for other CRMs (Salesforce, GoHighLevel), outbound messaging, or ad-platform performance data.
---

# HubSpot via Windsor.ai

Windsor.ai exposes live HubSpot CRM and marketing data (read only) through MCP
tools. The connector id is `hubspot`. Pass `hubspot` as the connector on every tool call; the Windsor.ai server
serves all connectors, so it is never implied.

Golden rules:

- **Never guess field, account, or option names.** They come from `get_fields`,
  `get_connectors`, and `get_options`. Call those first; do not invent identifiers.
- HubSpot is **read-only** here — there are no write actions. Do not offer to
  create, edit, or delete records, or to send emails.
- Report the numbers the user asked for, concisely. Don't dump raw rows unless asked.

## 1. Ground yourself first

Call `get_connectors` to see which HubSpot accounts are connected (each has an id
and, usually, a name). If none are connected, help the user connect one: call
`get_connector_authorization_url` (or `get_connector_connect_info` for the auth
type and steps) and give them the setup link — do not describe manual dashboard
navigation.

If the user names an account, match it to a connected account id. If several are
connected and the request is ambiguous, ask which account.

## 2. Reading CRM and marketing data

1. Call `get_fields` for `hubspot` to get valid field ids — contacts, companies,
   deals, deal stage, pipeline, deal amount/value, close date, owner, lead source,
   and marketing metrics (email opens, clicks, conversions, revenue). Use the exact
   ids returned, not these examples.
2. If the analysis needs segmentation or attribution controls, call `get_options`
   to see the connector's options and valid values.
3. Call `get_data` with:
   - `fields`: the ids you actually need (keep the set small).
   - `accounts`: the target account id(s); omit for all connected accounts.
   - A date range: `date_from`/`date_to` (`"2026-07-01"`), or a `date_preset`
     (`last_7d`, `last_30d`, `last_90d`, `this_month`, `this_year`, `last_year`;
     append `T` to include today).
   - `options` / `filters` only if relevant, e.g. `[["deal_stage","eq","closedwon"]]`.

Common requests → recipe:

- **Pipeline value:** fields `deal_stage` + deal amount; aggregate by stage.
- **Win analysis:** filter to closed-won deals, group by `lead_source` or owner.
- **Marketing performance:** email/campaign fields (sends, opens, clicks,
  conversions) over a preset range; compute open and click rates.

Watch out: money fields are in the account currency (state it if known). If a
field the user wants isn't in `get_fields`, say so rather than substituting a
different metric.

## 3. Reference the user's data, don't fabricate

If a tool returns an error (unknown field/connector/account), call the relevant
discovery tool (`get_fields`, `get_connectors`) and retry with correct ids. Never
fill gaps with made-up numbers, deal names, or contacts.
