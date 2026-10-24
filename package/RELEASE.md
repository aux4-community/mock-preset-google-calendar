# Release notes

## 0.0.3

- Stateful effects (needs aux4/mock 0.1.0+): creating, updating or deleting an event now
  reprograms the GET endpoints, so a later read of the event and of the events list
  reflects it (a deleted event 404s and drops out of the list).
- An update returns the event with the sent fields applied; fields it does not send keep
  their preset values (via the mock's `${body.x:-default}` fallback).
- Scratch files the preset writes are removed after use.
