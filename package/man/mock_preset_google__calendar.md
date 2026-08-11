#### Description

The `calendar` command registers the Google Calendar API on a running `aux4/mock` server, so a
Calendar client can run against a local mock instead of Google. It stubs the calendar list, the
events collection and item, `quickAdd`, and `freeBusy` in one call.

It registers credentials for nothing — run `aux4 mock preset google auth` first. That command
writes the token file and stubs the OAuth endpoints; this one only serves the API.

The stub paths are templates, so any calendar id and any event id match: `--calendarId` and
`--eventId` change what the response bodies say, not which requests are matched. The `--calendar`,
`--event`, `--start`, and `--end` flags shape the bodies the same way. When a flag does not reach
the field you need, pass the whole body instead with `--calendarListBody`, `--eventBody`,
`--eventsBody`, or `--freeBusyBody`.

The preset is a base layer. Apply it, then call `aux4 mock stub` for any endpoint you want to
behave differently — a later stub for the same method and path replaces the preset's.

#### Usage

```bash
aux4 mock preset google calendar [--port <port>] [--calendar <name>] [--event <title>]
```

--port             Port of the running mock server, which derives the state directory (default: `7070`)
--name             Stable handle of the server, used instead of `--port`
--stateDir         Explicit state directory, taking precedence over `--port` and `--name`
--calendar         Display name of the calendar returned by `calendarList` (default: `Sally`)
--calendarId       Calendar id written into the response bodies (default: `primary`)
--event            Title of the event returned by the event endpoints (default: `Team sync`)
--eventId          Id of that event (default: `evt1`)
--start            Event start time, RFC3339 (default: `2026-01-15T09:00:00Z`)
--end              Event end time, RFC3339 (default: `2026-01-15T10:00:00Z`)
--calendarListBody Raw JSON body for `calendarList`
--eventBody        Raw JSON body for get, create, update, and quickAdd
--eventsBody       Raw JSON body for the events list
--freeBusyBody     Raw JSON body for `freeBusy`

#### Example

```bash
aux4 mock start --port 18901
aux4 mock preset google auth --port 18901
aux4 mock preset google calendar --port 18901 --calendar "Work" --event "Design review"
```

```text
stub GET /users/me/calendarList -> 200
stub GET /calendars/{calendarId}/events -> 200
stub POST /calendars/{calendarId}/events -> 200
stub POST /calendars/{calendarId}/events/quickAdd -> 200
stub GET /calendars/{calendarId}/events/{eventId} -> 200
stub PATCH /calendars/{calendarId}/events/{eventId} -> 200
stub DELETE /calendars/{calendarId}/events/{eventId} -> 204
stub POST /freeBusy -> 200
```

A Calendar client pointed at the mock then returns the stubbed event:

```bash
aux4 google calendar events list --tokenFile google-token.json --apiUrl http://127.0.0.1:18901/api
```

```text
{"kind":"calendar#events","summary":"Work","items":[{"kind":"calendar#event","id":"evt1","status":"confirmed","summary":"Design review","start":{"dateTime":"2026-01-15T09:00:00Z"},"end":{"dateTime":"2026-01-15T10:00:00Z"}}]}
```
