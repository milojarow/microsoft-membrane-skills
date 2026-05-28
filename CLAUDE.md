# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

This is the **microsoft-membrane-skills** repository — a Claude Code marketplace that ships one skill for driving Microsoft Outlook (email, calendar, contacts, tasks) via the [membrane](https://getmembrane.com) CLI's `microsoft-outlook` connector.

**Repository**: https://github.com/milojarow/microsoft-membrane-skills

## Repository Structure

```
microsoft-membrane-skills/
├── .claude-plugin/          # marketplace.json + plugin.json
├── CLAUDE.md                # This file
├── README.md                # Project overview
├── LICENSE                  # MIT
├── evaluations/             # Test scenarios for the skill
└── skills/
    └── microsoft-membrane/
        ├── SKILL.md          # Entry point (lean)
        └── reference/        # installation.md, connections.md, graph-api-patterns.md
```

## The skill

### microsoft-membrane
Drive `membrane` against the Microsoft Outlook connector. `SKILL.md` is the lean entry point with the discover-before-build principle, a quick-reference table, and cross-links to:
- `reference/installation.md` — install + first login (headless variant included).
- `reference/connections.md` — connection create / list / reconnect.
- `reference/graph-api-patterns.md` — common Graph queries (`/me/messages`, `/me/mailFolders`, `$search`, `$select`, calendar/contacts) for when no pre-built action exists.

## Skill Activation

Activates when reading or modifying Microsoft Outlook data (email, calendar, contacts, tasks) via the `membrane` CLI — connection setup, action discovery, running actions, raw Graph API requests.

## Updating this skill

After any session that discovers a new action, gotcha, or Graph API pattern. Keep entries **generic** — no real connectionIds, no real account addresses, no internal Microsoft tenant data. The git log of this repo is the diary.
