# Managing Microsoft Outlook connections in membrane

A **connection** in membrane is one authenticated link to one external account (here: one Microsoft Outlook account). It holds OAuth tokens, membrane refreshes them transparently, and every `action run` / `request` call references the connection by its `connectionId`.

## Find the Outlook connector

The connector is the *type* (Outlook in general); the connection is your *instance* of it. Find the connector ID:

```bash
membrane search microsoft-outlook --elementType=connector --json
```

Take the connector ID from the first matching item:

```bash
CONNECTOR_ID=$(membrane search microsoft-outlook --elementType=connector --json | jq -r '.items[0].element.id // .items[0].id')
echo "$CONNECTOR_ID"
```

(The exact JSON path depends on membrane's response shape — `.items[0].element.id` is the documented field; some responses inline it as `.items[0].id`. The `// ` fallback in `jq` handles both.)

## Create a new connection

```bash
membrane connect --connectorId="$CONNECTOR_ID" --json
```

This prints a URL and (on a GUI box) opens it in a browser. Sign into the Microsoft account you want to connect, approve the requested scopes. The output contains the new `connectionId`:

```bash
CONN=$(membrane connect --connectorId="$CONNECTOR_ID" --json | jq -r '.connectionId // .id')
echo "$CONN"
```

Headless flow is analogous to `membrane login` — open the URL on another device and copy the completion code back.

## List existing connections

```bash
membrane connection list --json
```

Returns all your connections (across all connectors). Filter to just Outlook ones:

```bash
membrane connection list --json | jq '.items[] | select(.connectorKey == "microsoft-outlook")'
```

Find the connectionId for a specific account by email:

```bash
membrane connection list --json | jq -r '.items[] | select(.email == "you@outlook.com") | .id'
```

(Adjust the field names per the actual response — `connection list --json` shows you the shape.)

## Reconnect / rotate

If a connection expires or is revoked (user revoked access in Microsoft's account settings, password changed, etc.), the next action call will fail with an auth error. To recover, create a fresh connection:

```bash
membrane connect --connectorId="$CONNECTOR_ID" --json
```

Whether the membrane CLI exposes a way to delete the stale connection first (e.g. a `connection delete <id>` subcommand) should be checked with `membrane connection --help`. If no delete command is available, recovery is simply creating a fresh connection (above) and using the new connectionId.

The new connection may have a different `connectionId`. Long-running scripts should resolve the connectionId dynamically via `connection list` rather than hardcoding it.

## Multiple Outlook accounts

Each Microsoft account is a separate connection. Repeat the `membrane connect` flow once per account, sign into the right Microsoft account in the browser flow, store the resulting connectionIds:

```bash
CONN_PERSONAL=$(membrane connection list --json | jq -r '.items[] | select(.email == "you@outlook.com") | .id')
CONN_WORK=$(membrane connection list --json | jq -r '.items[] | select(.email == "you@company.example") | .id')

membrane request "$CONN_PERSONAL" '/me/messages?$top=5&$select=subject'
membrane request "$CONN_WORK"     '/me/messages?$top=5&$select=subject'
```

## Lifecycle

- **OAuth refresh is automatic** — membrane refreshes the access token before it expires. You don't need to do anything.
- **A connection is per-membrane-account** — if you log out of membrane (`membrane logout` or clearing creds), connections survive in membrane's server-side state and re-appear on next login.
- **Revocation outside membrane** — if the user revokes the CLI's access from Microsoft's side (account.microsoft.com → Privacy → App permissions), the next call fails with an auth error. Recovery: re-run `membrane connect --connectorId="$CONNECTOR_ID"`.

## Troubleshooting

- **`Connection not found`** — wrong connectionId, or it was deleted. Run `connection list` to confirm.
- **`unauthorized` / `auth refresh failed`** — the OAuth token can't be refreshed. Most common cause: user revoked from Microsoft's side. Run `membrane connect --connectorId="$CONNECTOR_ID"` to make a new one.
- **`scope not granted`** — the connection was created with fewer scopes than the action needs. Re-run `membrane connect --connectorId="$CONNECTOR_ID"`; in the browser approval, accept all requested scopes.
