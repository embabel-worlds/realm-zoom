# pack-zoom

Zoom Cloud Meetings — recordings, transcripts (VTT), and AI Companion
summaries, typed via a curated, vendored OpenAPI 3 mini-spec.

> Pack authoring reference: see
> [`docs/pack-format.md`](https://github.com/embabel/assistant/blob/main/docs/pack-format.md)
> in the assistant repo for the full pack format spec.

## What you get

- **Tools** (`gateway.zoom.*`) for listing recordings, fetching a
  meeting's recording artefacts, reading post-meeting metadata, and
  pulling the AI Companion summary.
- **Webhooks** for `recording.completed`,
  `recording.transcript_completed`, and `meeting.summary_completed` —
  HMAC-SHA256 verified against the Zoom Webhook Secret Token, with
  per-tenant routing on `payload.account_id`.
- A **`zoom-recordings` Agent Skill** that teaches the post-meeting
  ingest workflow (transcript download via `download_token`, AI
  summary fold-in, document persistence shape).
- The assistant's webhook controller already handles Zoom's
  synchronous `endpoint.url_validation` CRC handshake (added in the
  same release as this pack); just paste the Secret Token into
  `oauth-apps.yml` and click "Validate" in the Zoom Marketplace UI.

## Auth — OAuth2

End users **never** paste tokens. They click **Authorize** in
Settings → Connected Services → Zoom.

This works because the assistant deployment has ONE registered Zoom
OAuth app. Every end user connects their own Zoom account against
that single app.

### For end users

1. Open **Settings → Connected Services**.
2. Click **Authorize** on the `zoom` row.
3. Consent on Zoom's page. `gateway.zoom.*` is live.

### For installation admins (one-time setup)

Register a Zoom OAuth app at
<https://marketplace.zoom.us/develop/create>:

1. App type: "OAuth" (user-managed) or "Server-to-Server OAuth" if you
   want unattended access. Most installations want plain OAuth.
2. **Redirect URL**: `https://<your-public-host>/api/v1/oauth/callback/zoom`
3. **Scopes**:
   - `cloud_recording:read`
   - `meeting:read`
   - `meeting_summary:read:summary` *(only if AI Companion is licensed)*
   - `user:read`
4. **Feature → Event Subscriptions** → Add Subscription:
   - Endpoint URL: `https://<your-public-host>/api/v1/webhooks/zoom`
   - Events: `recording.completed`, `recording.transcript_completed`,
     and `meeting.summary_completed` (if using AI Companion).
   - Click **Validate** — Zoom POSTs an `endpoint.url_validation`
     event; the assistant replies synchronously with the CRC response.
5. Copy the **Secret Token** from the Event Subscription panel — this
   is what Zoom uses to sign webhook bodies. **It is not the same as
   the OAuth client secret.**

Then paste both into `<workspace-base>/admin/oauth-apps.yml`:

```yaml
apps:
  zoom:
    client-id: <oauth client id>
    client-secret: <oauth client secret>
    webhook-secret: <Secret Token from Event Subscriptions panel>
```

The assistant uses `webhook-secret` for HMAC verification and the URL
validation handshake; if absent, signature verification falls back to
`client-secret` (which is fine for providers like HubSpot/GitHub where
they happen to be the same value, but **not** for Zoom).

## Why a curated mini-spec?

Zoom's full OpenAPI is multi-MB and 200+ operations. We expose six,
all relevant to the chat-assistant transcript-ingest workflow:

| operation                      | what it does                                        |
|--------------------------------|-----------------------------------------------------|
| `users.me`                     | Authenticated user (`id`, `account_id`)             |
| `users.recordings.list`        | List a user's cloud recordings (date-windowed)      |
| `meetings.recordings.get`      | Get all artefacts for one meeting (incl. transcript)|
| `meetings.get`                 | Meeting metadata (topic, host, agenda)              |
| `meetings.meeting_summary.get` | AI Companion summary (overview, next steps, …)      |
| `past_meetings.get`            | Past meeting instance details (participants, etc.)  |
| `past_meetings.instances`      | List all past instances of a recurring meeting      |

## Things explicitly NOT in scope

Add when concrete need arises:

- Creating/updating/deleting meetings, registrants, polls.
- Live captions / RTMS websocket stream.
- Phone, Contact Center, Whiteboard, Chat APIs.
- Account/billing/admin surface.

## Known limitation: multi-user-per-Zoom-account

Zoom webhooks identify the *tenant* (`account_id`), not the user. The
assistant uses the same identifier for OAuth identity AND webhook
tenant routing, so:

- **Single user per Zoom org** (most common): everything works.
- **Multiple users from the same Zoom org each connect separately**:
  the last user to authorize "wins" the webhook tenant slot. Inbound
  webhooks for that org route to the most-recent user only. Other
  users' UIs still see their recordings via OAuth, but post-meeting
  webhooks won't fan out.

The fix is a framework change (decouple OAuth identity from webhook
tenant key); not in `v0.1`. Track upstream.

## Versioning

`v0.1.0` — initial: recordings, transcripts, AI summary, three webhooks,
one skill.
