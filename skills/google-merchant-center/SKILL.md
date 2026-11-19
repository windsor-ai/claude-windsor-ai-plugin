---
name: google-merchant-center
description: Analyze and update Google Merchant Center products through Windsor.ai. Use when the user asks about their Merchant Center product feed or Shopping results (clicks, impressions and CTR by product, category or product type, counts of active, disapproved, pending and expiring products by destination and country, price benchmarks and suggested prices, click potential, competitor domains and top merchants in a category, product attributes such as availability, GTIN or brand, or data sources), or wants to change a product, for example update a title, price, availability or another attribute, pause a product, add or delete a product, or create an API data source. Do not use for Google Ads Shopping campaign spend or ROAS (those are in the Google Ads connector), other platforms (e.g. Amazon), or writing product copy that is not applied to a connected account.
---

# Google Merchant Center via Windsor.ai

Windsor.ai pulls Google Merchant Center product feed and Shopping data, and
can change products in the connected account. The connector id is `google_merchant`. Pass `google_merchant` as the connector on every tool call; the Windsor.ai server
serves all connectors, so it is never implied.

This connector has **no Google Ads Shopping campaign data** (spend and ROAS
are in the Google Ads connector, so route those questions there). It reads
the product feed and Shopping performance, and it can update, pause, add and
delete products through API data sources (section 3). It does not edit a feed
file, feed rules, promotions, or shipping and tax settings.

Golden rules:

- **Never guess field, account, action, or option names.** Get them from
  `get_fields`, `get_connectors`, `list_actions`, and `get_options`. Call
  those first.
- **Reads are safe; writes change what shoppers see.** Only call
  `execute_action` after the user has confirmed the specific change.
- Report what the user asked for, concisely.

## 1. Ground yourself first

Call `get_connectors` to see which Merchant Center accounts are connected
(each has an id and, usually, a name). If none is connected, help the user
connect one: call `get_connector_authorization_url` (or
`get_connector_connect_info` for the auth type and steps) and give them the
setup link. Do not describe manual dashboard navigation.

If the user names an account, match it to a connected account id. If several
are connected and the request is ambiguous, ask which account.

## 2. Reading Merchant Center data

1. Call `get_fields` for `google_merchant` to get valid field ids. They
   come in groups:
   - Shopping performance by product: `product_clicks`,
     `product_impressions`, `product_ctr`, and for free listings
     `product_conversions` and `product_conversion_rate`, with
     `product_title`, `product_category_l1` to `l5`, `product_type_l1` to
     `l5` and `product_feed_label`.
   - Prices: `product_price`, `product_benchmark_price`,
     `product_price_gap`, `product_suggested_price` and the predicted
     `product_click_uplift`, `product_impression_uplift` and
     `product_conversion_uplift`; also `product_click_potential`.
   - Product status counts: `product_status_active_count`,
     `product_status_disapproved_count`, `product_status_pending_count`
     and `product_status_expiring_count`, by
     `product_status_reporting_context` and `product_status_country`.
   - Competitive visibility: `cv_competitor_*` (domains with visibility
     similar to yours), `cv_top_merchant_*` (domains with the highest
     visibility) and `cv_benchmark_*` (your trend against the category).
   - Product attributes: `product_availability`, `product_brand`,
     `product_gtin`, `product_mpn`, `product_image_link`,
     `product_google_category`, custom labels, sale price and sale dates.
   - Data sources: `data_source_*`.
   Use the exact ids returned, and keep one request to one group.
2. If the analysis needs segmentation or other controls, call `get_options`
   to see the connector's options and valid values.
3. Call `get_data` with:
   - `fields`: the ids you actually need (keep the set small).
   - `accounts`: the target account id(s); omit for all connected accounts.
   - A date range: either `date_from`/`date_to` (`"2026-02-01"`), or
     `date_preset`: `last_7d`, `last_30d`, `this_month`; append `T` to
     include today (e.g. `last_7dT`, `this_monthT`).
   - `filters`: optional conditions, e.g. filtering to out-of-stock products or
     to a price-benchmark threshold; combine with `"and"`/`"or"` as needed.

Merchant Center specifics to respect:

- Feed attributes (availability, price, price benchmark, suggested price)
  and product status counts describe the **current state of the feed**,
  while clicks, impressions, and CTR are **per-day performance metrics**
  over the requested range. Don't present a current feed attribute as if
  it varied over a historical date range.
- Product status is returned as **counts** per reporting context (for
  example SHOPPING_ADS or FREE_LISTINGS) and country: active, disapproved,
  pending and expiring. No field lists which products are disapproved or
  why. Say so, and point the user to the Products page in Merchant Center
  for the reason. If the user names the product and the attribute to fix,
  and `update_product` can change it, offer to make the change (section 3).
- Competitive visibility rows are per category, country and traffic source
  (ADS, ORGANIC or ALL). Ask for the category and country if the user did
  not give them. If the result is empty, say so.
- Conversions and conversion rate cover free listings only.

Common requests and their recipes:

- **Product status:** the four `product_status_*_count` fields with
  `product_status_reporting_context` and `product_status_country`.
- **Suggested prices:** product, price, suggested price and the predicted
  click, impression and conversion uplift; rank by click uplift.
- **Competitors:** `cv_competitor_domain` with rank, relative visibility,
  page overlap rate and higher position rate for one category and country.
- **Product data audit:** product id and title with GTIN, MPN, brand, image
  link and Google product category; list the products where one is empty.
- **High impressions, low CTR:** fields for product, impressions, clicks
  (or CTR), over a date range; rank by impressions and CTR in your answer.
- **Price benchmarking:** fields for product price and price benchmark;
  compare and flag products priced above or below benchmark.

If a field the user wants isn't in `get_fields`, say so rather than
substituting a different metric. There is no ad spend or campaign data here.

## 3. Changing products (write actions)

`list_actions` returns the write actions and their JSON schemas, and
`execute_action` runs one. Always call `list_actions` first and use the ids
and parameters it returns; treat the summary below as guidance. If
`list_actions` returns no actions, tell the user that product changes are not
available on this connection.

- `update_product`: change the given attributes of a product: title,
  description, link, image_link, additional_image_links, price (with
  currency), sale_price, availability, condition, brand, gtins, mpn,
  google_product_category, product_types and custom_label_0 to
  custom_label_4. Attributes that are not given keep their values.
  `clear_attributes` deletes attributes. `pause` set to `ADS` or `ALL` pauses
  the product, and `NONE` ends the pause.
- `insert_product`: add a product to an API data source. An entry with the
  same product_id, content_language and feed_label is replaced.
- `delete_product`: delete a product from an API data source.
- `create_data_source`: create an API data source, primary or supplemental.

Where a change goes:

- Every write names a `data_source`: the data_source_id or data_source_name
  of an **API** data source. Read `data_source_id`, `data_source_name`,
  `data_source_type`, `data_source_input` and
  `data_source_linked_primary_sources` to list the account's data sources.
- A product's own source is in `product_data_source`. If its input is API,
  write to that data source.
- If the product comes from a file feed, an app or the Merchant Center UI,
  `update_product` needs a supplemental API data source that is linked to the
  product's primary data source. If none exists, propose `create_data_source`
  with `type` supplemental and `link_to_primary_data_source`, and confirm
  that step on its own: it puts the new source first in the default rule of
  the primary source, so its values override the feed. Each call creates a
  new data source, so check the existing ones first.
- Values written to a supplemental data source override the feed until
  `delete_product` removes them from that supplemental source or
  `clear_attributes` clears them. Tell the user that the next feed upload
  does not undo the change.
- `delete_product` on a primary data source removes the whole product. On a
  supplemental data source it removes only that source's values.
- `product_id`, `content_language` and `feed_label` come from the
  `product_id`, `product_content_language` and `product_feed_label` fields.
- Merchant Center shows a change after a few minutes. Say that, so the user
  does not expect to see it at once.

Write workflow, every time:

1. Read the product with `get_data`: its id, content language, feed label,
   data source, and the current value of the attribute to change.
2. Pick the data source as described above.
3. **Show the user the concrete change and ask them to confirm:** the
   product, the attribute, the current value, the new value, and the data
   source the change is written to.
4. Only after an explicit yes, call `execute_action` and report the result.

For several products, list every product and its change, and confirm once for
that list. Do not extend the change to products the user has not seen. Never
invent a GTIN, a price or other product data: use values the user gives or
that the account returns.

## 4. Reference real data, don't fabricate

On an error (unknown field/connector/account/action), call the relevant
discovery tool and retry with correct ids, or report the specific error. Never
invent product statuses, disapproval reasons, click or impression counts, or
price figures.
