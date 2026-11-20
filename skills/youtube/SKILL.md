---
name: youtube
description: Analyze and manage a YouTube channel through Windsor.ai. Use when the user asks about their YouTube channel's organic insights — views, watch time, average view duration, subscribers gained and lost, likes, comments, shares, traffic sources and search terms, audience retention, or views by device, country or subscribed status, at the channel, playlist, or video level (including comparing Shorts vs. regular videos). It can also reply to and moderate comments, update a video's title, description, tags, category and privacy status, and add videos to or remove them from playlists, always after the user confirms the exact change. Do not use for YouTube Ads, other platforms, or uploading videos and thumbnails.
---

# YouTube via Windsor.ai

Windsor.ai pulls YouTube channel, playlist, and video analytics from the
YouTube Analytics API. The connector id is `youtube`. Pass `youtube` as the connector on every tool call; the Windsor.ai server
serves all connectors, so it is never implied.

Reads are the main job. The connector also exposes write actions, currently
`reply_to_comment`, `moderate_comment`, `delete_comment`,
`update_video_details`, `add_video_to_playlist` and
`remove_video_from_playlist` (the live set is whatever `list_actions`
returns). It has **no ad or cost data** (YouTube Ads run through Google Ads,
so route ad-spend questions there), and it does **not** upload videos,
thumbnails or captions or change channel settings; say that's out of scope.

Golden rules:

- **Never guess field, account, action, or option names** — get them from
  `get_fields`, `get_connectors`, `list_actions`, and `get_options`. Call
  those first.
- **Reads are safe; writes are public.** A reply, a moderation change or a
  video edit shows on the channel. Only call `execute_action` after the user
  confirms the specific change.
- Report what the user asked for, concisely.

## 1. Ground yourself first

Call `get_connectors` to see which YouTube channels are connected (each has
an id and usually a channel name). If none is connected, help the user
connect: call `get_connector_authorization_url` (or
`get_connector_connect_info` for the auth type and steps) and give them the
setup link — don't describe manual dashboard navigation. If several channels
are connected and the request is ambiguous, ask which one.

## 2. Reading YouTube data

1. Call `get_fields` for `youtube` to get valid field ids — channel,
   playlist, and video metrics (views, watch time, average view duration,
   subscribers gained and lost, likes, comments, shares) and dimensions
   (e.g. video title, content type, traffic source). Use the exact ids
   returned, not these examples.
2. If the analysis needs segmentation controls, call `get_options` to see
   the connector's options and valid values.
3. Call `get_data` with `fields`, the target `account`(s) (channel), and a
   date range (`date_from`/`date_to` or a `date_preset` like `last_7d`,
   `last_30d`, `last_Xd`/`last_Xw`/`last_Xm`, `this_month`, `this_year`,
   `last_year`; append `T` to include today, e.g. `last_30dT`). For ranges
   without a preset (e.g. "this quarter"), compute explicit `date_from`/
   `date_to`. Add `filters` as needed, e.g. to isolate Shorts vs. regular
   videos.

Common requests → recipe:

- **Channel overview / trend:** fields `date` + the metrics, over a preset
  range.
- **Top / worst videos:** fields `video` (or video title) + the metric
  (e.g. watch time), then rank; return only what was asked.
- **Shorts vs. regular videos:** fields for content type/format + the
  metric (e.g. average view percentage), grouped by format.
- **Traffic sources:** fields for traffic source + views, over the
  requested range.
- **Search terms and referring sites:** traffic source with its detail field
  (search terms, external sites) + views.
- **Audience retention:** one video (filter on its id) with the retention
  fields (elapsed video time ratio, audience watch ratio, relative retention
  performance) to find where viewers stop watching.
- **Devices, countries, subscribers:** device type, operating system,
  playback location, country, or subscribed status + views and watch time.
- **Comments:** the comment fields (text, author, date, likes, reply count)
  for the videos asked about.

If a metric the user wants isn't in `get_fields`, say so rather than
substituting a different one. There is no cost/spend data here, and
YouTube Ads questions belong in the Google Ads connector.

## 3. Managing the channel (write actions)

1. Call `get_connectors` to pick the channel; ask if several are connected.
2. Read what the change applies to with `get_data`: the video (id, title,
   description, tags, privacy status), the comment (id, text, author, video)
   or the playlist (id, title).
3. Call `list_actions` for `youtube` and read the schema of the action.
4. Show the exact change (which video, comment or playlist, the old value
   and the new value) and wait for an explicit yes. One change per
   confirmation unless the user approved an explicit list.
5. Call `execute_action` once and report the result from the tool.

Rules by action:

- **Video details:** send only what changes. Tags replace the whole list, so
  show the full new list, including the tags that stay. A privacy change to
  public or private takes effect at once; say so in the confirmation.
- **Replies:** show the exact text and the comment it answers. Replies are
  public and posted as the channel.
- **Moderation:** holding a comment for review hides it until a moderator
  acts; rejecting removes it with its replies and cannot be reversed;
  banning the author is a separate choice the user must ask for. Prefer
  holding over rejecting unless the user asks to reject.
- **Deleting:** only the channel's own comments and replies can be deleted,
  and it is permanent. To remove someone else's comment, reject it.
- **Playlists:** adding a video the playlist already holds changes nothing;
  removing takes out every entry of that video. The video itself stays.
- Each change uses part of the channel's daily YouTube API quota. For a long
  list, say how many changes will run before starting.

Reading comments and every write action need the channel management
permission that Google shows when the user connects YouTube. If a call
answers that the connection has read-only access, give the user the
reconnect link from that message or from `get_connector_authorization_url`
and ask them to keep every permission ticked. Analytics work without it.

## 4. Reference real data, don't fabricate

On an error (unknown field/connector/account), call the relevant discovery
tool and retry with correct ids, or report the specific error. Never invent
view counts, subscriber numbers, watch-time figures, video or comment ids. If
a write result is uncertain (for example a timeout on a reply), say it may or
may not have been posted and check before retrying.
