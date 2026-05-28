# Installing membrane + first login

`membrane` is a Node.js CLI. Install it globally, then complete the one-time login.

## Prerequisites

- **Node.js 18+** and `npm` on `$PATH`.
- A **membrane account** (Free tier is enough). Sign up at https://getmembrane.com if you don't have one.

## Install

```bash
npm install -g @membranehq/cli
```

The package puts a `membrane` binary on `$PATH`. Verify:

```bash
membrane --version
```

If `npm install -g` fails with EACCES on macOS/Linux without `sudo`, your global prefix isn't writable by your user. Either:

- Re-run with `sudo` (quick), or
- Set a user-writable prefix once: `npm config set prefix ~/.npm-global` and add `~/.npm-global/bin` to `$PATH`.

## First login

### Desktop / GUI machine

```bash
membrane login --tenant
```

A browser window opens against your membrane account. Sign in, grant the CLI access. The CLI captures the callback automatically; you're done.

### Headless / SSH-only

If the box has no browser (e.g. you're SSH'd into a server), `membrane login --tenant` prints a URL and a code. Two paths:

**Path A — open URL on another machine:**

1. Copy the printed URL.
2. Open it in a browser on any device where you're signed into membrane.
3. Approve the CLI session.
4. Copy the code shown back to you.
5. On the headless box:

```bash
membrane login complete <code>
```

**Path B — port-forward / SSH tunnel:**

If the login flow tries to open a local URL like `http://localhost:<port>/callback`, forward that port over your SSH session (`-L <port>:localhost:<port>`) and complete the flow in your local browser. This is fiddlier than Path A — prefer the code-paste flow when in doubt.

## Verify

```bash
membrane connection list --json
```

…should return an empty list (`[]` or `{"items":[]}`) if you haven't connected to any service yet. If it returns an auth error, the login didn't take — try `membrane login --tenant` again.

## Upgrading

```bash
npm install -g @membranehq/cli@latest
```

Re-run `membrane --version` to confirm.

## Troubleshooting

- **`command not found: membrane`** — the npm global bin directory isn't on `$PATH`. Find it with `npm bin -g` and add that to `$PATH`.
- **`Error: certificate has expired`** during install — `npm`'s root CAs are out of date; `npm install -g npm@latest` to refresh.
- **`auth error` / `not logged in` after `login` seemed to work** — clear the cached creds (`rm -rf ~/.config/membrane/` or equivalent) and try again. Make sure you didn't pick the wrong membrane account during the browser flow.
- **No `--tenant` flag** — for personal Outlook/Hotmail accounts on the consumer tenant, `--tenant` may not be required. Try without it first; add it back if the login complains.

## Next

Once logged in, create your first Outlook connection — see [connections.md](connections.md).
