# community/mock-preset-google-calendar

Google Calendar preset for [aux4/mock](https://hub.aux4.io/r/public/packages/aux4/mock).

One command replaces the Google Calendar API with a local mock: the calendar list, the events
collection and item (`GET`/`POST`/`PATCH`/`DELETE`), `quickAdd`, and `freeBusy`. Point a Google
Calendar client at the mock with `--apiUrl` and it behaves like the real service, offline.

## Installation

```bash
aux4 aux4 pkger install community/mock-preset-google-calendar
```

Installing pulls in `community/mock-preset-google-auth`, which supplies the credentials every
Google API call needs. Requires `aux4/mock` 0.1.0 or later (for the stateful-effects `--thenFile`
flag used by this preset's create/update/delete endpoints).

## Usage

Start a mock, apply the credentials, then apply the calendar endpoints:

```bash
aux4 mock start --port 18901
aux4 mock preset google auth --port 18901
aux4 mock preset google calendar --port 18901
```

`auth` writes `google-token.json` and stubs the OAuth endpoints; `calendar` stubs the API. A
Calendar client then runs entirely against the mock:

```bash
aux4 google calendar events list --tokenFile google-token.json --apiUrl http://127.0.0.1:18901/api
```

```json
{"kind":"calendar#events","summary":"Sally","items":[{"kind":"calendar#event","id":"evt1","status":"confirmed","summary":"Team sync","start":{"dateTime":"2026-01-15T09:00:00Z"},"end":{"dateTime":"2026-01-15T10:00:00Z"}}]}
```

### Endpoints

| Method | Path | Returns |
|---|---|---|
| `GET` | `/users/me/calendarList` | one calendar, from `--calendar` |
| `GET` | `/calendars/{calendarId}/events` | a one-event list |
| `POST` | `/calendars/{calendarId}/events` | the created event |
| `POST` | `/calendars/{calendarId}/events/quickAdd` | the created event |
| `GET` | `/calendars/{calendarId}/events/{eventId}` | the event |
| `PATCH` | `/calendars/{calendarId}/events/{eventId}` | the updated event |
| `DELETE` | `/calendars/{calendarId}/events/{eventId}` | `204`, no body |
| `POST` | `/freeBusy` | one busy block, from `--start`/`--end` |

The paths are templates: any calendar id and any event id match, whatever `--calendarId` and
`--eventId` are set to. Those flags control what the **bodies** say.

### Stateful effects: GET reflects an earlier create/update/delete

The preset now wires each mutating endpoint to reprogram the GET endpoints when it serves a
response (via [`aux4/mock`'s `--thenFile`](https://hub.aux4.io/r/public/packages/aux4/mock)), so a
test that acts and then reads back sees the effect of the action instead of the original fixture:

- **`DELETE`** the event -> a later `GET` of that event returns Google's `404 Not Found` envelope,
  and the events list no longer includes it.
- **`PATCH`** the event -> a later `GET` of that event, and the events list, reflect the
  `summary`/`start`/`end` the `PATCH` request sent. A field the request didn't send resolves to an
  empty string — send the fields you want reflected (a reschedule typically sends `start` and
  `end`).
- **`POST`** create -> a later `GET` by the fixed `--eventId` returns the created event's
  `summary`/`start`/`end`.

```bash
aux4 mock preset google calendar --port 18901

curl -s http://localhost:18901/api/calendars/primary/events/evt1
# {"kind":"calendar#event","id":"evt1","summary":"Team sync", ...}

curl -s -X DELETE http://localhost:18901/api/calendars/primary/events/evt1

curl -s -w "\n%{http_code}" http://localhost:18901/api/calendars/primary/events/evt1
# {"error":{"errors":[{"domain":"global","reason":"notFound","message":"Not Found"}],"code":404,"message":"Not Found"}}
# 404
```

This is single-fixture state: the mock always starts with exactly one event (`--eventId`), so the
events list toggles between that one item and empty — it does not track an arbitrary number of
created events.

### Shaping the responses

Name the calendar, the event, and its times:

```bash
aux4 mock preset google calendar --port 18901 \
  --calendar "Work" --event "Design review" \
  --start 2026-03-01T14:00:00Z --end 2026-03-01T15:00:00Z
```

| Option | Description | Default |
|---|---|---|
| `--port` | Port of the running mock server | `7070` |
| `--name` | Stable handle of the server, instead of `--port` | |
| `--stateDir` | Explicit state directory | |
| `--calendar` | Calendar display name | `Sally` |
| `--calendarId` | Calendar id in the response bodies | `primary` |
| `--event` | Event title | `Team sync` |
| `--eventId` | Event id | `evt1` |
| `--start` | Event start, RFC3339 | `2026-01-15T09:00:00Z` |
| `--end` | Event end, RFC3339 | `2026-01-15T10:00:00Z` |

For a response the flags do not reach, replace a whole body: `--calendarListBody`, `--eventBody`,
`--eventsBody`, and `--freeBusyBody` each take raw JSON. Use `--eventsBody` to return several
events, or an empty `items` array to test the no-results path.

### Layering your own stubs

A preset is a starting point, not a wall. Apply it, then override any single endpoint with
`aux4 mock stub` — the later stub wins:

```bash
aux4 mock preset google calendar --port 18901
aux4 mock stub --port 18901 --method GET --path '/calendars/{calendarId}/events' --status 403 \
  --body '{"error":{"code":403,"message":"Rate Limit Exceeded","status":"PERMISSION_DENIED"}}'
```

That is how to test an error path: take the whole API from the preset and bend the one endpoint
under test.

## See also

- [community/mock-preset-google-auth](https://hub.aux4.io/r/public/packages/community/mock-preset-google-auth) — the credentials every Google preset needs
- [aux4/mock](https://hub.aux4.io/r/public/packages/aux4/mock) — the mock server itself
