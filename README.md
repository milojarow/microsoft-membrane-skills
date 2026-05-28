# microsoft-membrane-skills

**Drive Microsoft Outlook (email, calendar, contacts, tasks) from Claude Code via the [membrane](https://getmembrane.com) CLI.**

## What is this?

A Claude Code marketplace with one skill, `microsoft-membrane`. It teaches Claude how to use the `membrane` CLI as a proxy to the Microsoft Graph API — connecting to Outlook / Hotmail accounts, discovering pre-built actions, running them, and falling back to raw Graph API requests when no action exists. Membrane handles OAuth and credential refresh transparently, so the skill never asks the user for tokens.

### Why this skill exists

- **Membrane abstracts away OAuth + token refresh** for Microsoft Graph. The skill teaches the right mental model: connections (auth + lifecycle), actions (pre-built operations), proxy (raw API when no action exists).
- **Discover-before-build** — `action list --intent=QUERY` returns pre-built actions with pagination + field mapping already handled. Skipping discovery and going straight to raw Graph URLs burns tokens and re-implements what membrane already does.
- **The headless-login path is non-obvious** — `membrane login --tenant` opens a browser on a desktop, but on a remote SSH-only box you need `membrane login complete <code>` after opening the printed URL elsewhere. The install reference spells this out.
- **`--json` is not the default** — without it, output is human-formatted and a pain to parse. The skill keeps reminding.
- **`connection list` is the recovery path** when a connectionId rotates after re-auth — without this, scripts hardcoded to old IDs break silently.

## The skill

| Skill | Description |
|-------|-------------|
| **microsoft-membrane** | Drive `membrane` against the Microsoft Outlook connector — connection lifecycle, action discovery, action runs, proxy requests to Microsoft Graph, with auth/refresh handled transparently. |

## Installation

Add this marketplace in Claude Code:

```
/plugin → Marketplaces → Add Marketplace → milojarow/microsoft-membrane-skills
```

Then install:

```
/plugin → Discover → microsoft-membrane-skills → Install
```

## Requirements

- The **membrane** CLI on `$PATH` (Node 18+, `npm install -g @membranehq/cli`). See `skills/microsoft-membrane/reference/installation.md`.
- A Microsoft account (personal Outlook/Hotmail, or work/school) you can OAuth into.
- A Membrane account (Free tier is enough).

## License

MIT
