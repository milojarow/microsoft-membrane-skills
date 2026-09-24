---
name: microsoft-membrane
description: Use when reading or modifying Microsoft Outlook data (email, calendar, contacts, tasks) from the terminal via the `membrane` CLI's microsoft-outlook connector — installing membrane, logging in, creating/listing connections, discovering and running pre-built actions, or reaching Microsoft Graph endpoints (the raw `membrane request` proxy is deprecated and 503s, so Graph calls go through `action run`).
---

# Microsoft Outlook via membrane

Drive Microsoft Outlook (email, calendar, contacts, tasks) by proxying the Microsoft Graph API through the [membrane](https://getmembrane.com) CLI. Membrane handles OAuth and token refresh transparently — you never see API keys.

> **📬 ACTIVE-SKILL MARKER:** While `microsoft-membrane` is active, begin every reply with 📬 so the operator sees at a glance that this skill is engaged.

## Overview

`membrane` is a generic CLI proxy to many SaaS APIs (Microsoft Outlook, Slack, Notion, …). This skill focuses on the **`microsoft-outlook` connector** specifically. The mental model has three layers:

| Layer | What it is | When to use |
|---|---|---|
| **Connections** | Per-account auth — created once via browser, refreshed automatically | One-time setup per Outlook account |
| **Actions** | Pre-built operations with pagination + field mapping handled | Default — discover with `action list` |
| **Proxy** | Raw HTTP to the Graph API endpoint | Fallback when no action covers your case |

The discipline: **discover an action first, fall back to proxy only when nothing fits.**

## ⚠️ `membrane request` is deprecated and currently 503s — read this first

The proxy layer prints a deprecation notice and then fails with **503**, while the connection itself reports healthy (`connected: true`, `state: READY`, `errors: []`). The subcommand is broken, not the integration.

**The failure mode is silent and looks like success.** The error body is valid JSON that echoes the request back, with no `value` key — so a parser doing `d.get('value', [])` reports **"0 messages"**. A transport failure reads as an empty inbox. **If an account that normally has traffic returns 0, re-run with no filters; if it's still 0, the instrument is broken, not the mailbox.** (The error output also prints the full bearer token — never paste it into logs or tickets.)

**Use `action run` for everything**, including cases you'd previously have proxied. Payload lands at **`.output.data.value`** (not `.value`), and `/me/messages` already spans all folders. Full recipe: [reference/graph-api-patterns.md](reference/graph-api-patterns.md).

## When to use

- Reading recent / search-by-keyword Outlook email from the terminal.
- Listing or creating calendar events.
- Reading contacts.
- Managing tasks (To-Do).
- Iterating mail folders.
- Anything else exposed by the Microsoft Graph `/me/…` endpoints — read or write.

**Not for:** administering a Microsoft 365 tenant (user provisioning, group policies, license assignment, audit logs). Those need the Graph admin endpoints with a tenant-admin token, which is outside membrane's per-user connector model and outside this skill's scope.

**Not for:** non-Microsoft connectors (Slack, Notion, etc.) — membrane supports them, but this skill is scoped to `microsoft-outlook`. The patterns transfer, but the action IDs and connector ID differ.

## Prerequisites

- The `membrane` CLI on `$PATH`. Verify with `membrane --version`.
- A logged-in membrane session (`membrane login --tenant`).
- A Microsoft Outlook connection in your membrane account.

If membrane isn't installed or you haven't logged in yet, see [reference/installation.md](reference/installation.md) — it also covers the **headless / SSH-only** login flow (no browser on the box).

## Workflow — the four-question flow

```
   ┌─────────────────────────────┐
   │ membrane installed?         │  → reference/installation.md
   └─────────────┬───────────────┘
                 │ yes
   ┌─────────────▼───────────────┐
   │ logged into membrane?       │  → membrane login --tenant  (or headless variant)
   └─────────────┬───────────────┘
                 │ yes
   ┌─────────────▼───────────────┐
   │ outlook connection exists?  │  → reference/connections.md
   └─────────────┬───────────────┘
                 │ yes  (note the connectionId)
   ┌─────────────▼───────────────┐
   │ is there a pre-built action │
   │ for what I want to do?      │  → action list --intent="..."
   └──┬───────────────┬──────────┘
      │ yes           │ no
      ▼               ▼
   action run     broaden the intent and look again —
                  the proxy is deprecated and 503s
                  → reference/graph-api-patterns.md
```

## Discover-before-build (the core principle)

Pre-built actions handle pagination, field mapping, error shapes, and Graph API quirks for you. Hand-rolling a `/me/messages?$top=N&$select=…` call from memory is fine, but you're re-implementing what `action run` already does — and you lose the rich `inputSchema` that documents valid parameters.

**Always start with:**

```bash
membrane action list --intent="<what you want to do>" --connectionId=$CONN --json
```

Replace `<what you want to do>` with natural language: `"list recent emails"`, `"create calendar event"`, `"search inbox"`. The response is a list of action objects each with an `id` and an `inputSchema`. Pick the one that fits and:

```bash
membrane action run --connectionId=$CONN <action_id> --json [--input '{"key":"value"}']
```

Only when no action covers the case → fall back to the proxy (see [reference/graph-api-patterns.md](reference/graph-api-patterns.md)).

**If `action list` returns nothing for your intent**, try broadening the natural-language phrase first (`"send"` instead of `"send email with attachment"`). If still empty after broadening, your connection may lack the required Graph scope — re-run `membrane connect --connectorId=$CONNECTOR_ID` and accept ALL scopes in the browser flow, then retry. As a last resort, drop to the proxy and call the Graph endpoint directly.

**Microsoft Graph rate limits** apply to both actions and proxy calls — Graph throttles per-app and per-user. In tight loops, expect `429 Too Many Requests` with a `Retry-After` header (in seconds). Respect it; don't retry sooner. For batch operations, prefer the `$batch` endpoint or a pre-built bulk action over a hot loop of single calls.

## Quick reference

| Want to… | Command |
|---|---|
| First login | `membrane login --tenant` |
| Find the Outlook connector ID | `membrane search microsoft-outlook --elementType=connector --json` |
| Create a connection | `membrane connect --connectorId=<connectorId> --json` |
| List existing connections | `membrane connection list --json` |
| Discover actions | `membrane action list --intent="..." --connectionId=$CONN --json` |
| Run an action | `membrane action run --connectionId=$CONN <action_id> --json [--input '{"k":"v"}']` |
| Raw Graph API | `membrane request $CONN /me/messages?$top=10` — **deprecated, 503s; use `action run`** |

For full connection lifecycle (create, list, reconnect after rotation): [reference/connections.md](reference/connections.md).
For Graph API patterns (mailFolders, $search, $select, calendar, contacts, To-Do tasks) and the proxy flag table: [reference/graph-api-patterns.md](reference/graph-api-patterns.md).

## Common mistakes

- **Going straight to the proxy without discovering actions** — re-implements pagination + field mapping for nothing. `action list` first.
- **Forgetting `--json`** — output is human-formatted and unparseable in scripts. Add it to every command meant for automation.
- **Asking the user for API keys or tokens** — never. Membrane manages OAuth + refresh. The only credential involved is the membrane account login.
- **Hardcoding a connectionId that has rotated** — after re-auth, the connectionId may change. Use `connection list` at the top of long-running scripts to resolve it dynamically.
- **Omitting `--tenant` on `membrane login`** for work/school multi-tenant accounts — login can fail in unhelpful ways.
- **Passing `--input` JSON with shell-unfriendly quoting** — wrap the whole payload in single quotes and use double quotes inside, OR pass it via a heredoc to a file and `--inputFile`.
- **Forgetting that `request` URLs need URL-encoded query params** — `$select=subject` and `$search="keyword"` work but watch the quoting. (Moot while `request` is deprecated — pass OData params in the action's `--input` instead.)
- **Reading `.value` off an `action run` response** — the payload is at `.output.data.value`. Reading `.value` yields nothing and looks like an empty mailbox.
- **Trusting a "0 messages" result** — on a connection that normally has traffic, zero means "verify the transport", not "the inbox is empty". Re-query with no filters before reporting it.
