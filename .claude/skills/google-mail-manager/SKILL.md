---
name: google-mail-manager
description: Read, search, triage, send, and manage Gmail messages and threads using the Google Workspace CLI. Use this skill whenever the user wants to read emails, search their inbox, list unread messages, send emails, reply to threads, label/trash messages, or do any Gmail operation.
---

# Gmail Manager

Manage Gmail via `nix run github:googleworkspace/cli` (the `gws` CLI).

The CLI alias used throughout this skill:

```bash
alias gws='nix --extra-experimental-features "nix-command flakes" run github:googleworkspace/cli --'
```

Always run commands with that prefix. For brevity, examples use `gws` to represent the full invocation.

## Authentication

Before any Gmail operation, confirm credentials are set up:

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
| Read / search | `https://www.googleapis.com/auth/gmail.readonly` |
| Send / reply | `https://www.googleapis.com/auth/gmail.send` |
| Modify labels, trash | `https://www.googleapis.com/auth/gmail.modify` |
| Full access | `https://www.googleapis.com/auth/gmail` |

---

## Common Operations

### Triage — show unread inbox summary

```bash
# Default: up to 20 unread messages (table format)
gws gmail +triage

# Limit results
gws gmail +triage --max 10

# Filter by query
gws gmail +triage --query 'from:boss@example.com'
gws gmail +triage --query 'is:unread after:2024/01/01'

# Include label names
gws gmail +triage --labels

# JSON output for scripting
gws gmail +triage --format json | jq '.[].subject'
```

### Read a message

```bash
# Plain text body
gws gmail +read --id MESSAGE_ID

# Include From/To/Subject/Date headers
gws gmail +read --id MESSAGE_ID --headers

# HTML body
gws gmail +read --id MESSAGE_ID --html

# JSON output
gws gmail +read --id MESSAGE_ID --format json
```

`+read` handles base64 decoding, multipart/alternative, and HTML-to-text conversion automatically.

### List messages

```bash
# All inbox messages (default 20)
gws gmail users messages list --params '{"userId": "me"}' --format table

# Search by Gmail query
gws gmail users messages list --params '{"userId": "me", "q": "is:unread"}' --format table
gws gmail users messages list --params '{"userId": "me", "q": "from:someone@example.com subject:invoice"}' --format table

# Limit results
gws gmail users messages list --params '{"userId": "me", "maxResults": 50}' --format table

# All results (paginated)
gws gmail users messages list --params '{"userId": "me", "q": "is:unread"}' --page-all --format table
```

Common Gmail search operators: `is:unread`, `is:starred`, `from:`, `to:`, `subject:`, `has:attachment`, `after:YYYY/MM/DD`, `before:YYYY/MM/DD`, `label:`, `in:inbox`, `in:sent`

### Get full message metadata

```bash
gws gmail users messages get --params '{"userId": "me", "id": "MESSAGE_ID", "format": "metadata"}'
gws gmail users messages get --params '{"userId": "me", "id": "MESSAGE_ID", "format": "full"}'
```

Formats: `minimal`, `full`, `raw`, `metadata`

### Send an email

```bash
gws gmail +send --to recipient@example.com --subject "Hello" --body "Message body here"

# With CC / BCC
gws gmail +send --to a@example.com --cc b@example.com --bcc c@example.com --subject "Hello" --body "Body"

# HTML body
gws gmail +send --to a@example.com --subject "Hello" --body "<h1>Hello</h1>" --html
```

### Reply to a message

```bash
# Reply (preserves thread)
gws gmail +reply --id MESSAGE_ID --body "My reply text"

# Reply-all
gws gmail +reply-all --id MESSAGE_ID --body "My reply text"
```

### Forward a message

```bash
gws gmail +forward --id MESSAGE_ID --to recipient@example.com --body "FYI:"
```

### Watch for new emails (stream)

```bash
gws gmail +watch
```

Streams new emails as NDJSON to stdout. Ctrl-C to stop.

---

## Labels

### List labels

```bash
gws gmail users labels list --params '{"userId": "me"}' --format table
```

### Apply / remove labels on a message

```bash
# Add a label (get label ID from list above)
gws gmail users messages modify --params '{"userId": "me", "id": "MESSAGE_ID"}' --json '{"addLabelIds": ["LABEL_ID"]}'

# Remove a label
gws gmail users messages modify --params '{"userId": "me", "id": "MESSAGE_ID"}' --json '{"removeLabelIds": ["LABEL_ID"]}'

# Mark as read (remove UNREAD label)
gws gmail users messages modify --params '{"userId": "me", "id": "MESSAGE_ID"}' --json '{"removeLabelIds": ["UNREAD"]}'

# Mark as unread
gws gmail users messages modify --params '{"userId": "me", "id": "MESSAGE_ID"}' --json '{"addLabelIds": ["UNREAD"]}'

# Star a message
gws gmail users messages modify --params '{"userId": "me", "id": "MESSAGE_ID"}' --json '{"addLabelIds": ["STARRED"]}'
```

### Bulk label modification

```bash
gws gmail users messages batchModify --params '{"userId": "me"}' --json '{"ids": ["ID1", "ID2"], "addLabelIds": ["LABEL_ID"], "removeLabelIds": ["UNREAD"]}'
```

---

## Threads

```bash
# List threads
gws gmail users threads list --params '{"userId": "me", "q": "is:unread"}' --format table

# Get a thread
gws gmail users threads get --params '{"userId": "me", "id": "THREAD_ID"}'

# Trash a thread
gws gmail users threads trash --params '{"userId": "me", "id": "THREAD_ID"}'
```

---

## Trash and Delete

```bash
# Move to trash (recoverable)
gws gmail users messages trash --params '{"userId": "me", "id": "MESSAGE_ID"}'

# Untrash
gws gmail users messages untrash --params '{"userId": "me", "id": "MESSAGE_ID"}'

# Permanent delete (irreversible)
gws gmail users messages delete --params '{"userId": "me", "id": "MESSAGE_ID"}'

# Bulk delete
gws gmail users messages batchDelete --params '{"userId": "me"}' --json '{"ids": ["ID1", "ID2"]}'
```

---

## Profile

```bash
gws gmail users getProfile --params '{"userId": "me"}'
```

---

## Output and Parsing Tips

- Use `--format table` for human-readable output
- Use `--format json` (default) for scripting
- Pipe to `jq` for field extraction:

```bash
# Extract IDs from a list
gws gmail users messages list --params '{"userId": "me", "q": "is:unread"}' | jq '.messages[].id'

# Get subject of a specific message
gws gmail users messages get --params '{"userId": "me", "id": "ID", "format": "metadata", "metadataHeaders": ["Subject","From","Date"]}' | jq '.payload.headers[] | select(.name=="Subject") | .value'
```

---

## Checking the Schema

```bash
gws schema gmail.users.messages.list
gws schema gmail.users.messages.get
gws schema gmail.users.messages.send
gws schema gmail.users.threads.list
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
