---
name: facebook-ads
description: Analyze and manage Meta Ads (Facebook & Instagram ad placements) through Windsor.ai. Use when the user asks about their Facebook Ads / Meta Ads performance — pulling campaign, ad set, or ad metrics (spend, impressions, clicks, conversions, CPA, ROAS), comparing date ranges, or finding top or underperforming campaigns — or when they want to create, pause, or enable a campaign, ad set, or ad, change a budget, boost an existing post, edit ad creative or targeting, duplicate an ad set, look up targeting interests or locations, or create a lead form (Instant Form). Do not use for other ad platforms (Google Ads, TikTok, LinkedIn, Microsoft Ads), organic Facebook Page posts or Instagram organic content, or designing/generating ad creative images.
---

# Facebook Ads via Windsor.ai

Windsor.ai exposes Meta Ads data (read) and a set of campaign management
actions (write) for Facebook and Instagram ad placements through MCP tools.
The connector id is `facebook`. Pass `facebook` as the connector on every tool call; the Windsor.ai server
serves all connectors, so it is never implied.

This connector covers **Meta Ads** (paid Facebook & Instagram ad placements)
only. It is distinct from Facebook Organic (Page posts) and Instagram
organic, which live in their own connectors — route organic questions there.

Golden rules:

- **Never guess field, account, action, or option names.** They come from
  `get_fields`, `get_connectors`, `list_actions`, and `get_options`. Call those
  first; do not invent identifiers.
- **Reads are safe; writes change live ad spend.** Only call `execute_action`
  after the user has confirmed the specific change.
- Report numbers the user asked for, concisely. Don't dump raw rows unless asked.

## 1. Ground yourself first

Call `get_connectors` to see which Meta Ads accounts are connected (each has
an id and, usually, a name). If none are connected, help the user connect one:
call `get_connector_authorization_url` (or `get_connector_connect_info` for the
auth type and steps) and give them the setup link — do not describe manual
dashboard navigation.

If the user names an account, match it to a connected account id. If several are
connected and the request is ambiguous, ask which account.

## 2. Reading performance data

1. Call `get_fields` for `facebook` to get valid field ids (e.g. campaign,
   ad set, ad, date, spend, impressions, clicks, conversions, purchases — use
   the exact ids returned, not these examples).
2. If the analysis needs attribution windows or other segmentation controls,
   call `get_options` to see the connector's options and valid values (e.g.
   click/view attribution windows).
3. Call `get_data` with:
   - `fields`: the ids you actually need (keep the set small).
   - `accounts`: the target account id(s); omit for all connected accounts.
   - A date range: either `date_from`/`date_to` (`"2026-02-01"`), or
     `date_preset` — `last_7d`, `last_30d`, `last_Xd`/`last_Xw`/`last_Xm`,
     `this_month`, `this_year`, `last_year`. Append `T` to include today
     (e.g. `last_7dT`, `this_monthT`).
   - `options`: only if get_options showed relevant ones.
   - `filters`: optional conditions, e.g. `[["spend", "gt", 100]]`; combine
     with `"and"`/`"or"` and nest as needed.

Common requests → recipe:

- **Overview / trend:** fields `date` + the metrics, over a preset range.
- **By campaign / ad set / ad:** fields campaign/ad set/ad + metrics; sort or
  aggregate in your answer.
- **Best / worst campaign:** pull campaign + the metric, then rank; return only
  what was asked (e.g. just the campaign name) if the user constrained the
  output format.
- **Efficiency:** cost per result = spend ÷ conversions (or purchases); ROAS =
  purchase value ÷ spend. Compute from returned fields; note if the
  denominator is zero.

Watch out: Meta spend fields are in the account currency (state it if known).
If a field the user wants isn't in `get_fields`, say so rather than
substituting a different metric.

## 3. Managing campaigns (write actions)

Meta Ads exposes a set of write actions via `list_actions` / `execute_action`.
Always call `list_actions` first for the exact ids and JSON schemas — treat
the summary below as guidance, not gospel. They cover:

- **Create / structure:** `create_campaign` (Facebook or Instagram, with an
  objective such as OUTCOME_TRAFFIC/OUTCOME_LEADS/OUTCOME_SALES and any
  required `special_ad_categories`; created paused), `create_adset` (in an
  existing campaign, with targeting, optimization goal, and billing event;
  created paused), `create_ad` (in an existing ad set — either a simple
  link-ad builder from page_id/link/message/image_url/call_to_action_type, a
  Click-to-Message ad via `messaging_destination`, or a raw creative spec;
  created paused).
- **Status:** `pause_campaign` / `enable_campaign`, `pause_adset` /
  `enable_adset`, `pause_ad` / `enable_ad`.
- **Budget:** `set_campaign_budget` and `set_adset_budget` (`daily` or
  `lifetime`; setting one budget type replaces the other).
- **Update existing objects:** `update_campaign` (name, bid_strategy,
  special_ad_categories, spend_cap), `update_adset` (name, targeting,
  optimization_goal, bid_amount/bid_strategy, destination_type, end_time),
  `update_ad` (name, creative reference, conversion_domain,
  display_sequence), `update_ad_creative` (edits an ad's copy, headline,
  description, link, image, call-to-action, URL tags, or one carousel card —
  Meta treats creatives as immutable, so this creates a new creative and
  re-points the ad at it).
- **Organic post promotion:** `boost_post` (turn an existing organic Facebook
  or Instagram post into a paid ad in an existing engagement ad set, reusing
  the post as the creative; created paused).
- **Messaging:** `set_page_welcome_message` (set the automated greeting shown
  when someone opens a conversation from a Click-to-Message ad).
- **Duplicate:** `duplicate_adset` copies an ad set with its targeting, budget
  and schedule, into the same campaign or another one (`campaign_id`), and
  with its ads when `deep_copy` is true. The copy is paused unless
  `status_option` says otherwise. There is no delete action, so a copy can be
  paused but not removed: confirm before copying, and do not repeat a copy
  that reported a partial result.
- **Images and video:** `create_ad_image` uploads image files from the chat
  (it has `requires_files`, so it runs through `upload_files`; in an IDE
  no widget appears, so run the one-time `curl` command it returns with the
  user's file paths) and returns one
  `image_hash` per image for `create_ad`. `create_ad_video` takes a public
  `video_url`, waits for Meta to process it and returns a `video_id` for
  `create_ad`, which then also needs a thumbnail. If Meta cannot fetch the
  video, report the error and ask for another URL; do not retry the same one.
  Uploads cannot be undone by an action.
- **Lead forms:** `create_lead_form` creates an Instant Form on a Facebook
  Page (questions, privacy policy URL, optional intro screen and end-screen
  link). Attach it in `create_ad` through
  `object_story_spec.link_data.call_to_action.value.lead_gen_form_id`, in an ad
  set with `destination_type` ON_AD of an OUTCOME_LEADS campaign. Meta does
  not allow editing or deleting a form: create a new one and archive the old
  one with `set_lead_form_status`. The connected Facebook user needs the
  Advertise task on the page.
- **Targeting lookups:** `search_targeting` (interests, behaviors, job titles
  and other demographics, by keyword) and `search_locations` (countries,
  regions, cities, postal codes) are read-only and return the ids and keys
  that `create_adset` and `update_adset` take in `targeting`. They change
  nothing, so they need no confirmation. Never invent a targeting id: look it
  up, and check that each returned name fits what the user meant, because
  Meta returns broad suggestions when nothing matches.

An action named above that `list_actions` does not return is not available on
this connection yet: say so, and do not try to run it.

Budget and creation mechanics:

- Money amounts are in the account currency's **minor unit** (e.g. cents):
  `5000` means 50.00 — not micros. Convert the user's amount before calling.
- Budget actions (`set_campaign_budget`, `set_adset_budget`, `spend_cap`) set
  an **absolute** value, not a delta. To "raise by 20%", read the current
  value first, compute the new absolute amount, and set that.
- `create_campaign`, `create_adset`, `create_ad`, `boost_post` and
  `duplicate_adset` all produce **paused** objects by default, so nothing spends until the user
  enables it.

Write workflow — every time:

1. Resolve the campaign/ad set/ad to its id (via a `get_data` read on the
   relevant name and id fields if the user gave a name).
2. Build the exact params from the `list_actions` schema.
3. **Show the user the concrete change and ask them to confirm** — e.g.
   "Pause campaign 'Summer Sale' (id 123…) on account 456…?" or "Set daily
   budget for ad set X to €50.00 (5000 in minor units)?".
4. Only after an explicit yes, call `execute_action` and report the result.

If the user says "pause the worst campaign", first do the read + ranking, then
name the specific campaign and confirm before pausing — don't pause on the
strength of the aggregate alone.

## 4. Reference the user's data, don't fabricate

If a tool returns an error (unknown field/connector/account), call the relevant
discovery tool (`get_fields`, `get_connectors`, `list_actions`) and retry with
correct ids. Never fill gaps with made-up numbers or campaign names.
