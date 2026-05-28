# Microsoft Graph API patterns via membrane's proxy

Use the proxy only when **no pre-built action covers your case**. For most everyday operations, `membrane action list --intent="…"` returns an action with pagination + field mapping already handled.

The proxy command:

```bash
membrane request <connectionId> <path> [flags]
```

Membrane prepends the Graph base URL (`https://graph.microsoft.com/v1.0` for the current connector) and injects the auth header automatically. You write the path starting at `/`.

## Proxy flags

| Flag | Purpose |
|---|---|
| `-X, --method` | HTTP method (GET, POST, PUT, PATCH, DELETE). Default GET. |
| `-H, --header` | Add a header (repeatable). `-H "Accept: application/json"`. |
| `-d, --data` | Request body (string). |
| `--json` | Shorthand: send body as JSON + set `Content-Type: application/json`. |
| `--rawData` | Send body as-is, no processing. |
| `--query` | Query-string parameter (repeatable). `--query "limit=10"`. |
| `--pathParam` | Path parameter (repeatable). `--pathParam "id=123"`. |

## Common Graph query parameters

| Param | What it does | Example |
|---|---|---|
| `$top=N` | Page size | `$top=10` |
| `$skip=N` | Skip N (paging) | `$skip=20` |
| `$select=a,b,c` | Return only these fields (drastically smaller response) | `$select=subject,from,receivedDateTime` |
| `$filter=<expr>` | Filter results | `$filter=isRead eq false` |
| `$orderby=<field>` | Sort | `$orderby=receivedDateTime desc` |
| `$search="<terms>"` | Full-text search | `$search="invoice"` |
| `$expand=<rel>` | Expand a related entity | `$expand=attachments` |
| `$count=true` | Include total count | `$count=true` |

## Email patterns

### List recent inbox messages

```bash
membrane request "$CONN" '/me/messages?$top=10&$select=subject,from,receivedDateTime'
```

### List unread

```bash
membrane request "$CONN" '/me/messages?$filter=isRead eq false&$top=20&$select=subject,from,receivedDateTime'
```

### Search inbox by keyword

```bash
membrane request "$CONN" '/me/messages?$search="invoice"&$top=10&$select=subject,from,receivedDateTime'
```

`$search` is full-text and returns relevance-ordered (you can't combine `$search` with `$orderby`). For exact field matches use `$filter`.

### Read a single message in full

```bash
membrane request "$CONN" '/me/messages/<messageId>'
```

The `messageId` comes from the `id` field in any `/me/messages` listing response.

### List mail folders

```bash
membrane request "$CONN" '/me/mailFolders'
```

### Messages in a specific folder

```bash
membrane request "$CONN" '/me/mailFolders/<folderId>/messages?$top=20&$select=subject,from'
```

Use `displayName` from `/me/mailFolders` to find the folder — common ones are `Inbox`, `SentItems`, `Drafts`, `DeletedItems`, `JunkEmail`, `Archive`.

### Reply to a message

```bash
membrane request "$CONN" '/me/messages/<messageId>/reply' \
  -X POST --json -d '{"comment":"Sounds good — sending the deck now."}'
```

### Send a new message

```bash
membrane request "$CONN" '/me/sendMail' \
  -X POST --json -d '{
    "message": {
      "subject": "Status update",
      "body": {"contentType":"Text","content":"All on track."},
      "toRecipients": [{"emailAddress":{"address":"recipient@example.com"}}]
    },
    "saveToSentItems": true
  }'
```

(For automation, prefer the pre-built `send mail` action — `action list --intent="send mail"` — which has cleaner ergonomics.)

### Mark as read

```bash
membrane request "$CONN" '/me/messages/<messageId>' \
  -X PATCH --json -d '{"isRead":true}'
```

### Move to a folder

```bash
membrane request "$CONN" '/me/messages/<messageId>/move' \
  -X POST --json -d '{"destinationId":"<folderId>"}'
```

## Calendar patterns

### List upcoming events

```bash
membrane request "$CONN" '/me/events?$top=10&$select=subject,start,end&$orderby=start/dateTime'
```

### Events in a date range (calendarView)

```bash
membrane request "$CONN" '/me/calendarView?startDateTime=2026-02-01T00:00:00Z&endDateTime=2026-02-28T23:59:59Z&$select=subject,start,end'
```

`calendarView` expands recurring events into instances; `/me/events` returns the master record only.

### Create an event

```bash
membrane request "$CONN" '/me/events' \
  -X POST --json -d '{
    "subject": "1:1 with Alice",
    "start": {"dateTime":"2026-02-15T15:00:00","timeZone":"America/Mexico_City"},
    "end":   {"dateTime":"2026-02-15T15:30:00","timeZone":"America/Mexico_City"},
    "attendees": [{"emailAddress":{"address":"alice@example.com","name":"Alice"},"type":"required"}]
  }'
```

## Contacts patterns

### List contacts

```bash
membrane request "$CONN" '/me/contacts?$top=50&$select=displayName,emailAddresses'
```

### Search contacts

```bash
membrane request "$CONN" '/me/contacts?$search="alice"&$top=10'
```

## Tasks (To-Do) patterns

### List task lists

```bash
membrane request "$CONN" '/me/todo/lists'
```

### List tasks in a list

```bash
membrane request "$CONN" '/me/todo/lists/<listId>/tasks?$top=20&$filter=status ne completed'
```

### Create a task

```bash
membrane request "$CONN" '/me/todo/lists/<listId>/tasks' \
  -X POST --json -d '{"title":"Follow up with Alice","dueDateTime":{"dateTime":"2026-02-20T09:00:00","timeZone":"America/Mexico_City"}}'
```

## Pagination

### Proxy responses

Graph paginates with `@odata.nextLink` in the response:

```bash
membrane request "$CONN" '/me/messages?$top=10' --json | jq '."@odata.nextLink"'
```

Follow the link as a full URL (it's already complete):

```bash
NEXT=$(membrane request "$CONN" '/me/messages?$top=10' --json | jq -r '."@odata.nextLink"')
# NEXT is e.g. "https://graph.microsoft.com/v1.0/me/messages?$skip=10&$top=10"
# Strip the base URL and pass the rest to `membrane request`:
PATH_ONLY="${NEXT#https://graph.microsoft.com/v1.0}"
membrane request "$CONN" "$PATH_ONLY"
```

### Pre-built actions

Pre-built actions handle pagination automatically — most return all results, transparently following `nextLink` server-side, and present a flat list to you. If a list-style action ever exposes paging at the CLI level, it'll show up as parameters in the `inputSchema` (look for `cursor`, `nextPageToken`, `pageSize` keys when you run `action list --intent="…" --json`). That's another reason to prefer actions when one exists.

## Rate limits / throttling

Microsoft Graph throttles both proxy calls and actions:

- Per-user and per-app limits apply. Hot loops will hit `429 Too Many Requests`.
- Response includes a `Retry-After` header (seconds). Wait at least that long before retrying — do NOT retry sooner, it resets the cooldown.
- For batch reads/writes, prefer Graph's `$batch` endpoint (up to 20 requests per call) over a hot loop of singletons.
- For automation, sleep briefly between proxy calls (`sleep 1` is usually enough for non-tight loops) and consider exponential backoff on 429.

## Gotchas

- **`$search` doesn't combine with `$orderby` or `$filter` on many resources** — Graph returns 400 if you try.
- **Times in `$filter` need ISO 8601 + `Z` (UTC)** — `receivedDateTime ge 2026-01-01T00:00:00Z`. No timezone offsets.
- **Folder names in `$filter` are case-sensitive** — match `displayName` exactly.
- **`$select` on a single resource still helps** — the full `/me/messages/<id>` response is huge; trim to what you actually use.
- **Body content type** — `"contentType":"Text"` for plain, `"HTML"` for HTML. HTML emails need properly escaped JSON.
- **Attachment uploads** are a multi-step Graph flow — prefer the pre-built `send mail with attachment` action over hand-rolling it.
