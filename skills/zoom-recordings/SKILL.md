---
name: zoom-recordings
description: Zoom Cloud Meeting recordings, transcripts (VTT), and AI Companion summaries — listing a user's recordings, fetching the artefacts for a single meeting, downloading the transcript or summary, and reacting to `recording.completed` / `recording.transcript_completed` / `meeting.summary_completed` webhooks. Activate this skill BEFORE any Zoom call when the user asks about meetings, recordings, transcripts, captions, summaries, action items, or "what happened in the X meeting".
---

# Zoom Recordings, Transcripts & AI Summaries

## Namespace

Calls go through `gateway.zoom.<method>(args)` from inside `execute_javascript`
or `execute_python`. Never call them as top-level tools.

Method names mirror the spec's operation ids with dots → underscores
(framework canonicalisation):

- `gateway.zoom.users_me({})`
- `gateway.zoom.users_recordings_list({ userId: "me", from: "2026-04-01" })`
- `gateway.zoom.meetings_recordings_get({ meetingId: "<id-or-uuid>" })`
- `gateway.zoom.meetings_get({ meetingId: "<id>" })`
- `gateway.zoom.meetings_meeting_summary_get({ meetingUUID: "<uuid>" })`
- `gateway.zoom.past_meetings_get({ meetingUUID: "<uuid>" })`
- `gateway.zoom.past_meetings_instances({ meetingId: "<id>" })`

If a call returns `gateway.zoom.foo is not a world tool`, the error lists every
valid method — pick from it. Never re-send the same call.

## Cardinal rules

1. **Don't fabricate UUIDs.** Meeting IDs and UUIDs come from `users_recordings_list`,
   webhook payloads, or `past_meetings_instances`. Don't guess them.
2. **UUIDs with `/` need double URL-encoding.** Zoom's documented quirk: if a
   meeting `uuid` starts with `/` or contains `//`, encode it twice before
   passing as a path param. Otherwise pass as-is.
3. **Prefer `meetingId: "me"` for "my recordings".** Saves a `users_me` round-trip.
4. **Always pass `from`/`to` for `users_recordings_list`.** Default is the last 7
   days; use ISO `YYYY-MM-DD`. Max range is one month per request — paginate
   month-by-month for longer windows.
5. **Don't poll for transcripts.** Wait for the `recording.transcript_completed`
   webhook; transcripts can take 30+ minutes after the meeting ends.

## Listing a user's recordings

```javascript
const r = await gateway.zoom.users_recordings_list({
  userId: "me",
  from: "2026-04-01",
  to: "2026-04-30",
  page_size: 100,
});
for (const m of r.meetings) {
  console.log(m.start_time, m.topic, m.uuid, m.recording_count, "files");
}
```

Pagination: response carries `next_page_token`. Pass it back as `next_page_token`
on the next call until empty.

## Getting the artefacts for one meeting

```javascript
const rec = await gateway.zoom.meetings_recordings_get({
  meetingId: "<meeting-id-or-uuid>",
});

// Find the transcript:
const transcript = rec.recording_files.find(f => f.file_type === "TRANSCRIPT");
const video      = rec.recording_files.find(f => f.file_type === "MP4");
const audio      = rec.recording_files.find(f => f.file_type === "M4A");
const chat       = rec.recording_files.find(f => f.file_type === "CHAT");
```

`file_type` values: `MP4` (video), `M4A` (audio), `TRANSCRIPT` (closed-caption
VTT), `CC` (raw captions text), `CHAT` (chat log), `TIMELINE`, `CSV`, `SUMMARY`.

## Downloading a transcript or any recording file

The `download_url` requires authentication. Two paths:

**(a) From a webhook handler.** The `recording.transcript_completed` and
`recording.completed` event payloads include a short-lived `download_token`
(Zoom calls it `download_access_token` on some endpoints). Append it as a
query string — no OAuth bearer needed:

```javascript
// Inside a webhook action, payload shape:
//   { event, payload: { account_id, object: { recording_files: [...] }}, download_token }
const transcriptFile = payload.object.recording_files.find(f => f.file_type === "TRANSCRIPT");
const url = `${transcriptFile.download_url}?access_token=${download_token}`;
const vtt = await fetch(url).then(r => r.text());
```

**(b) From an interactive call.** Use the world's authenticated HTTP
fetch (the OAuth bearer is added automatically for `https://*.zoom.us`):

```javascript
const vtt = await fetch_authenticated({
  url: transcript.download_url,
  provider: "zoom",
  responseType: "text",
});
```

## AI Companion summaries

If the host has Zoom AI Companion enabled AND turned on the per-meeting
summary toggle, Zoom generates a structured post-meeting summary
(overview, chapter-level details, next steps, key questions). The right
moment to fetch it is the `meeting.summary_completed` webhook:

```javascript
const s = await gateway.zoom.meetings_meeting_summary_get({
  meetingUUID: payload.object.meeting_uuid,
});
console.log(s.summary_title);
console.log(s.summary_overview);
for (const step of s.next_steps) console.log("•", step);
```

The summary is also editable in Zoom's UI after the fact, so re-fetching
with this endpoint is the way to pick up host edits — the webhook payload
captures only the initial AI version.

When AI Companion is **not** licensed (or summary was off for the meeting),
this endpoint returns 404. Fall back to summarising the VTT yourself.

## Pattern: ingest a transcript when it completes

The right shape for a `recording.transcript_completed` action:

1. Pull the meeting's `topic`, `start_time`, host email from
   `payload.object` (already in the webhook body — no extra call needed
   unless you want richer detail; if so, `meetings_get` or
   `past_meetings_get` by UUID).
2. Find the `TRANSCRIPT` file in `payload.object.recording_files`.
3. Fetch the VTT using `download_url + ?access_token=<download_token>`.
4. Strip VTT timestamps if you want plain text, or keep them for
   speaker/time alignment.
5. Persist into the assistant's documents store (one Document per meeting
   is the usual shape; topic + date as the title).

## Pattern: fold the AI summary onto the same Document

When `meeting.summary_completed` later fires for the same meeting:

1. Look up the existing transcript Document by `meeting_uuid`.
2. Call `meetings_meeting_summary_get({ meetingUUID })`.
3. Prepend the structured summary to the Document body, keep the VTT
   below as the source-of-truth body.

This way "what happened in the X meeting" returns the summary; deeper
queries (literal quotes, who said what) still reach the transcript.

## Past-meeting metadata

For richer context (duration, participant count) on a recurring meeting's
specific instance:

```javascript
const inst = await gateway.zoom.past_meetings_get({ meetingUUID: "<uuid>" });
console.log(inst.duration, inst.participants_count);
```

For a list of all past instances:

```javascript
const all = await gateway.zoom.past_meetings_instances({ meetingId: "<numeric-id>" });
for (const m of all.meetings) console.log(m.uuid, m.start_time);
```

## Things that are NOT in this realm

Add when there is a concrete need:

- Creating/updating/deleting meetings.
- Registrant management.
- Polls, surveys, Q&A, breakout rooms.
- Live in-meeting captions stream (websocket/RTMS — completely separate API).
- Phone, Contact Center, Whiteboard, Chat APIs.
- Account/billing/admin APIs.
