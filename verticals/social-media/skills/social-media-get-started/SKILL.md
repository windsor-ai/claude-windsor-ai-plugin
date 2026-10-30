---
name: social-media-get-started
description: "Set up Windsor.ai for Social Media and get to a first answer: sign in to the Windsor.ai social media MCP server, connect or reconnect Instagram, Facebook Pages, LinkedIn Pages, Threads, TikTok, YouTube or Google Business Profile accounts, resolve which account a request means, and run a first report. Use when the user has just installed the plugin, says \"set up\", \"get started\" or \"connect my social accounts\", sees no data, or hits a sign-in or permission error."
---

# Get started with Windsor.ai for Social Media

Get the user from a fresh install to a first real answer in as few steps as possible.

## 1. Check the connection
- Call `get_current_user`. If it fails with an authentication error, tell the user to run `/mcp`, select `windsor-social-media`, and sign in with their Windsor.ai account (sign-up happens in the same browser flow). Retry after they confirm.
- If `/mcp` does not list `windsor-social-media`, the plugin is not loaded: ask them to run `/plugin` and check that the plugin is enabled, then restart Claude Code.

## 2. See what is connected
- Call `get_connectors` with include_not_yet_connected=True. List the connected accounts for Instagram (`instagram`), Facebook Pages (`facebook_organic`), LinkedIn Pages (`linkedin_organic`), Threads (`threads`), TikTok (`tiktok_organic`), YouTube (`youtube`) and Google Business Profile (`google_my_business`), and which of these have no account yet.
- If nothing is connected, ask which platform matters most and continue with step 3.

## 3. Connect or reconnect an account
- Call `get_connector_connect_info` for the chosen connector and give the user the connect link it returns, with any steps it lists. `get_connector_authorization_url` also returns a setup link.
- Instagram needs a Professional account (Business or Creator).
- Never ask the user to paste API keys, tokens, or passwords into the chat, and never describe manual dashboard navigation instead of giving the link.
- After they confirm, call `get_connectors` again and name the account that appeared.
- A read that fails with a platform login or permission message (for example Facebook asking the user to log in to facebook.com) means the connection has expired: give the same connect link to reconnect. An Instagram account connected before comment access was added returns a permission error on comment actions until it is reconnected.
- If a post or other change is refused because write actions are off, send the user to https://onboard.windsor.ai/app/settings/account.

## 4. Resolve the account
If a request names a brand, client, Page or location, match it to a connected account id. If several accounts could match, ask which one before any read or write.

## 5. First answer
Offer one first report and run it on a yes: reach or views, engagement, follower change and the three best posts per connected platform for the last 30 days (`/social-report`).

## Out of scope on this server
This plugin connects only Windsor.ai's social media endpoint. Ad platforms, web analytics, CRM, e-commerce and finance sources are not available on it; for those, point the user to the main Windsor.ai plugin or connector.

## Support
`get_windsor_login_url` gives an auto-login link into the Windsor.ai dashboard; `contact_windsor` files a support or feature request. Never discuss plans or pricing.

## Hand-offs
- Any report -> /social-media-insights
- Publishing a post -> /social-media-publish
- Comments, replies and reviews -> /social-media-engage
- Google Business Profile listing edits -> /google-business-profile
