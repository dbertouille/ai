---
name: google-calendar-manager
description: View, create, update, and delete Google Calendar events and calendars using the Google Workspace CLI. Use this skill whenever the user wants to check their schedule, list upcoming events, create/edit/delete events, manage calendars, check availability, or do any Google Calendar operation.
---

# Google Calendar Manager

Manage Google Calendar via `nix run github:googleworkspace/cli` (the `gws` CLI).

The CLI alias used throughout this skill:

```bash
alias gws='nix --extra-experimental-features "nix-command flakes" run github:googleworkspace/cli --'
```

Always run commands with that prefix. For brevity, examples use `gws` to represent the full invocation.

## Authentication

Before any Calendar operation, confirm credentials are set up:

```bash
gws auth status
```

If not authenticated, use the [[google-oauth2-playground-auth]] skill to obtain a token with the required scopes, then set:

```bash
export GOOGLE_WORKSPACE_CLI_TOKEN=<access_token>
```

Required scopes by operation:

| Operation | Scope |
|-----------|-------|
| Read events / calendars | `https://www.googleapis.com/auth/calendar.readonly` |
| Read events only | `https://www.googleapis.com/auth/calendar.events.readonly` |
| Create / edit / delete events | `https://www.googleapis.com/auth/calendar.events` |
| Full calendar access | `https://www.googleapis.com/auth/calendar` |

---

## Calendar List

### List all calendars

```bash
# All calendars (default 100 results)
gws calendar calendarList list --format table

# Only calendars where user is owner or writer
gws calendar calendarList list --params '{"minAccessRole": "writer"}' --format table

# JSON for scripting
gws calendar calendarList list | jq '.items[] | {id, summary, primary}'
```

### Get a specific calendar

```bash
gws calendar calendarList get --params '{"calendarId": "primary"}'
gws calendar calendarList get --params '{"calendarId": "CALENDAR_ID"}'
```

Use `"primary"` to refer to the user's main calendar throughout all commands.

---

## Events

### List upcoming events

```bash
# Next 10 events on primary calendar
gws calendar events list --params '{"calendarId": "primary", "maxResults": 10, "singleEvents": true, "orderBy": "startTime", "timeMin": "2026-06-10T00:00:00Z"}' --format table

# Events in a time range
gws calendar events list --params '{"calendarId": "primary", "timeMin": "2026-06-10T00:00:00Z", "timeMax": "2026-06-17T23:59:59Z", "singleEvents": true, "orderBy": "startTime"}' --format table

# Search by keyword
gws calendar events list --params '{"calendarId": "primary", "q": "standup", "singleEvents": true, "orderBy": "startTime", "timeMin": "2026-06-10T00:00:00Z"}' --format table

# All results (paginated)
gws calendar events list --params '{"calendarId": "primary", "singleEvents": true, "orderBy": "startTime", "timeMin": "2026-06-10T00:00:00Z"}' --page-all --format table
```

Always pass `"singleEvents": true` and `"orderBy": "startTime"` for chronological results. Timestamps must be RFC3339 with timezone offset (e.g. `2026-06-10T00:00:00Z` or `2026-06-10T09:00:00-07:00`).

### Get a single event

```bash
gws calendar events get --params '{"calendarId": "primary", "eventId": "EVENT_ID"}'
```

### Create an event

```bash
# Simple all-day event
gws calendar events insert --params '{"calendarId": "primary"}' --json '{
  "summary": "Team offsite",
  "start": {"date": "2026-06-20"},
  "end": {"date": "2026-06-21"}
}'

# Timed event with timezone
gws calendar events insert --params '{"calendarId": "primary"}' --json '{
  "summary": "1:1 with Manager",
  "start": {"dateTime": "2026-06-15T10:00:00-07:00", "timeZone": "America/Los_Angeles"},
  "end": {"dateTime": "2026-06-15T10:30:00-07:00", "timeZone": "America/Los_Angeles"},
  "description": "Weekly check-in"
}'

# Event with attendees (sends invites by default: use sendUpdates="none" to suppress)
gws calendar events insert --params '{"calendarId": "primary", "sendUpdates": "all"}' --json '{
  "summary": "Project sync",
  "start": {"dateTime": "2026-06-16T14:00:00Z"},
  "end": {"dateTime": "2026-06-16T15:00:00Z"},
  "attendees": [
    {"email": "colleague@example.com"}
  ]
}'

# Event with Google Meet
gws calendar events insert --params '{"calendarId": "primary", "conferenceDataVersion": 1}' --json '{
  "summary": "Video call",
  "start": {"dateTime": "2026-06-16T14:00:00Z"},
  "end": {"dateTime": "2026-06-16T15:00:00Z"},
  "conferenceData": {
    "createRequest": {"requestId": "unique-id-123", "conferenceSolutionKey": {"type": "hangoutsMeet"}}
  }
}'

# Recurring event (weekly every Monday)
gws calendar events insert --params '{"calendarId": "primary"}' --json '{
  "summary": "Weekly standup",
  "start": {"dateTime": "2026-06-15T09:00:00-07:00", "timeZone": "America/Los_Angeles"},
  "end": {"dateTime": "2026-06-15T09:30:00-07:00", "timeZone": "America/Los_Angeles"},
  "recurrence": ["RRULE:FREQ=WEEKLY;BYDAY=MO"]
}'
```

### Update an event

```bash
# Patch (partial update — only provided fields change)
gws calendar events patch --params '{"calendarId": "primary", "eventId": "EVENT_ID"}' --json '{
  "summary": "Updated title",
  "description": "New description"
}'

# Move to a different time
gws calendar events patch --params '{"calendarId": "primary", "eventId": "EVENT_ID"}' --json '{
  "start": {"dateTime": "2026-06-16T15:00:00Z"},
  "end": {"dateTime": "2026-06-16T16:00:00Z"}
}'

# Full replace (put — all fields must be provided)
gws calendar events update --params '{"calendarId": "primary", "eventId": "EVENT_ID"}' --json '{
  "summary": "...",
  "start": {"dateTime": "..."},
  "end": {"dateTime": "..."}
}'
```

Prefer `patch` over `update` to avoid accidentally clearing fields.

### Delete an event

```bash
gws calendar events delete --params '{"calendarId": "primary", "eventId": "EVENT_ID"}'

# Delete without notifying attendees
gws calendar events delete --params '{"calendarId": "primary", "eventId": "EVENT_ID", "sendUpdates": "none"}'
```

### Move an event to another calendar

```bash
gws calendar events move --params '{"calendarId": "SOURCE_CALENDAR_ID", "eventId": "EVENT_ID", "destination": "DEST_CALENDAR_ID"}'
```

---

## Free/Busy Query

Check availability across calendars:

```bash
gws calendar freebusy query --json '{
  "timeMin": "2026-06-10T00:00:00Z",
  "timeMax": "2026-06-10T23:59:59Z",
  "items": [{"id": "primary"}, {"id": "OTHER_CALENDAR_ID"}]
}'
```

---

## Extracting Event IDs and Parsing

```bash
# Get IDs and summaries of upcoming events
gws calendar events list --params '{"calendarId": "primary", "singleEvents": true, "orderBy": "startTime", "timeMin": "2026-06-10T00:00:00Z", "maxResults": 20}' | jq '.items[] | {id, summary, start}'

# Find event ID by title
gws calendar events list --params '{"calendarId": "primary", "q": "standup", "singleEvents": true, "timeMin": "2026-06-10T00:00:00Z"}' | jq '.items[0].id'

# Get start time of an event
gws calendar events get --params '{"calendarId": "primary", "eventId": "EVENT_ID"}' | jq '.start'
```

---

## Checking the Schema

```bash
gws schema calendar.events.list
gws schema calendar.events.insert
gws schema calendar.events.patch
gws schema calendar.calendarList.list
gws schema calendar.freebusy.query
```

---

## Error Reference

| Exit code | Meaning |
|-----------|---------|
| 0 | Success |
| 1 | API error (Google returned an error) |
| 2 | Auth error — run `gws auth login` or set `GOOGLE_WORKSPACE_CLI_TOKEN` |
| 3 | Validation error — bad arguments |
| 4 | Discovery error — schema fetch failed |
| 5 | Internal error |
