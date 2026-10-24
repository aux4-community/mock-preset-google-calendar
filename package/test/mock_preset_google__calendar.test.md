# community/mock-preset-google-calendar

Proves that installing this package makes `aux4 mock preset google calendar` available through
aux4's profile merge — the `aux4/mock` core knows nothing about Calendar — and that the stubs it
applies answer every endpoint a Calendar client calls.

The server is started inside the first test's `execute` block (not `beforeAll`, which the harness
would kill) and torn down in `afterAll` on a unique port. It is a detached process, so it survives
across the sibling `##` scenarios; each override scenario clears the stub store first. All checks
use plain `curl`, with no external network.

```afterAll
aux4 mock stop --port 18902 2>/dev/null
pkill -f "18902" 2>/dev/null
true
```

## the calendar preset on a running mock

### should start the server and apply the calendar preset

```execute
aux4 mock start --port 18902
sleep 1
aux4 mock preset google calendar --port 18902
```

```expect:partial
stub GET /users/me/calendarList -> 200
```

### should list one calendar

```execute
curl -s http://localhost:18902/api/users/me/calendarList
```

```expect
{"kind":"calendar#calendarList","items":[{"kind":"calendar#calendarListEntry","id":"primary","summary":"Sally","primary":true,"accessRole":"owner"}]}
```

### should list the events of any calendar id

```execute
curl -s http://localhost:18902/api/calendars/primary/events
```

```expect:partial
"summary":"Team sync"
```

### should match a calendar id other than the default

The stub path is a template, so a client using a real calendar address still matches.

```execute
curl -s -o /dev/null -w "%{http_code}" http://localhost:18902/api/calendars/sally@example.com/events
```

```expect
200
```

### should return a single event by id

```execute
curl -s http://localhost:18902/api/calendars/primary/events/evt1
```

```expect
{"kind":"calendar#event","id":"evt1","status":"confirmed","summary":"Team sync","start":{"dateTime":"2026-01-15T09:00:00Z"},"end":{"dateTime":"2026-01-15T10:00:00Z"}}
```

### should create an event

```execute
curl -s -X POST http://localhost:18902/api/calendars/primary/events -H 'Content-Type: application/json' -d '{"summary":"anything"}'
```

```expect:partial
"id":"evt1"
```

### should quick-add an event

```execute
curl -s -o /dev/null -w "%{http_code}" -X POST "http://localhost:18902/api/calendars/primary/events/quickAdd?text=Lunch+at+noon"
```

```expect
200
```

### should update an event

```execute
curl -s -X PATCH http://localhost:18902/api/calendars/primary/events/evt1 -H 'Content-Type: application/json' -d '{"summary":"changed"}'
```

```expect:partial
"kind":"calendar#event"
```

### should delete an event with no content

```execute
curl -s -o /dev/null -w "%{http_code}" -X DELETE http://localhost:18902/api/calendars/primary/events/evt1
```

```expect
204
```

### should report a busy block

```execute
curl -s -X POST http://localhost:18902/api/freeBusy -H 'Content-Type: application/json' -d '{"timeMin":"2026-01-15T00:00:00Z"}'
```

```expect
{"kind":"calendar#freeBusy","calendars":{"primary":{"busy":[{"start":"2026-01-15T09:00:00Z","end":"2026-01-15T10:00:00Z"}]}}}
```

## overrides: name the calendar and the event

### should re-apply the preset with overrides

```execute
aux4 mock reset --port 18902 --stubs true
aux4 mock preset google calendar --port 18902 --calendar "Work" --event "Design review" --eventId evt99 --start 2026-03-01T14:00:00Z --end 2026-03-01T15:00:00Z
```

```expect:partial
stub POST /freeBusy -> 200
```

### should return the overridden calendar name

```execute
curl -s http://localhost:18902/api/users/me/calendarList
```

```expect:partial
"summary":"Work"
```

### should return the overridden event

```execute
curl -s http://localhost:18902/api/calendars/primary/events/evt99
```

```expect
{"kind":"calendar#event","id":"evt99","status":"confirmed","summary":"Design review","start":{"dateTime":"2026-03-01T14:00:00Z"},"end":{"dateTime":"2026-03-01T15:00:00Z"}}
```

### should carry the overridden times into freeBusy

```execute
curl -s -X POST http://localhost:18902/api/freeBusy -H 'Content-Type: application/json' -d '{}'
```

```expect:partial
"start":"2026-03-01T14:00:00Z"
```

## an empty result set

### should return no events when the list body is replaced

```execute
aux4 mock reset --port 18902 --stubs true
aux4 mock preset google calendar --port 18902 --eventsBody '{"kind":"calendar#events","summary":"Sally","items":[]}'
```

```expect:partial
stub GET /calendars/{calendarId}/events -> 200
```

### should serve the empty list

```execute
curl -s http://localhost:18902/api/calendars/primary/events
```

```expect
{"kind":"calendar#events","summary":"Sally","items":[]}
```

## stateful effects: GET reflects an earlier create/update/delete

### should re-apply a fresh preset

```execute
aux4 mock reset --port 18902 --stubs true
aux4 mock preset google calendar --port 18902
```

```expect:partial
stub DELETE /calendars/{calendarId}/events/{eventId} -> 204
```

### should return the original event before any mutation

```execute
curl -s http://localhost:18902/api/calendars/primary/events/evt1
```

```expect
{"kind":"calendar#event","id":"evt1","status":"confirmed","summary":"Team sync","start":{"dateTime":"2026-01-15T09:00:00Z"},"end":{"dateTime":"2026-01-15T10:00:00Z"}}
```

### should reflect a reschedule on the same event

```execute
curl -s -X PATCH http://localhost:18902/api/calendars/primary/events/evt1 -H 'Content-Type: application/json' -d '{"summary":"Team sync (rescheduled)","start":{"dateTime":"2026-01-16T11:00:00Z"},"end":{"dateTime":"2026-01-16T12:00:00Z"}}'
curl -s http://localhost:18902/api/calendars/primary/events/evt1
```

```expect:partial
{"kind":"calendar#event","id":"evt1","status":"confirmed","summary":"Team sync (rescheduled)","start":{"dateTime":"2026-01-16T11:00:00Z"},"end":{"dateTime":"2026-01-16T12:00:00Z"}}
```

### should reflect the reschedule in the events list too

```execute
curl -s http://localhost:18902/api/calendars/primary/events
```

```expect:partial
"summary":"Team sync (rescheduled)"
```

### should 404 a deleted event and drop it from the list

```execute
curl -s -o /dev/null -X DELETE http://localhost:18902/api/calendars/primary/events/evt1
curl -s -w "\n%{http_code}" http://localhost:18902/api/calendars/primary/events/evt1
```

```expect
{"error":{"errors":[{"domain":"global","reason":"notFound","message":"Not Found"}],"code":404,"message":"Not Found"}}
404
```

### should return an empty list after the delete

```execute
curl -s http://localhost:18902/api/calendars/primary/events
```

```expect
{"kind":"calendar#events","summary":"Sally","items":[]}
```

### should make a newly created event retrievable by the fixed event id

```execute
aux4 mock reset --port 18902 --stubs true
aux4 mock preset google calendar --port 18902
curl -s -X POST http://localhost:18902/api/calendars/primary/events -H 'Content-Type: application/json' -d '{"summary":"New Event","start":{"dateTime":"2026-02-01T08:00:00Z"},"end":{"dateTime":"2026-02-01T09:00:00Z"}}'
curl -s http://localhost:18902/api/calendars/primary/events/evt1
```

```expect:partial
{"kind":"calendar#event","id":"evt1","status":"confirmed","summary":"New Event","start":{"dateTime":"2026-02-01T08:00:00Z"},"end":{"dateTime":"2026-02-01T09:00:00Z"}}
```

## layering: an error path over the preset

### should apply the preset then override one endpoint

A later stub for the same method and path replaces the preset's, which is how an error path is
tested without hand-stubbing the rest of the API.

```execute
aux4 mock reset --port 18902 --stubs true
aux4 mock preset google calendar --port 18902
aux4 mock stub --port 18902 --method GET --path '/calendars/{calendarId}/events' --status 403 --body '{"error":{"code":403,"message":"Rate Limit Exceeded","status":"PERMISSION_DENIED"}}'
```

```expect:partial
stub GET /calendars/{calendarId}/events -> 403
```

### should serve the error on the overridden endpoint

```execute
curl -s -w "\n%{http_code}" http://localhost:18902/api/calendars/primary/events
```

```expect
{"error":{"code":403,"message":"Rate Limit Exceeded","status":"PERMISSION_DENIED"}}
403
```

### should still serve the untouched endpoints from the preset

```execute
curl -s http://localhost:18902/api/users/me/calendarList
```

```expect:partial
"accessRole":"owner"
```
