# Fortivo Property Services — Claude Instructions

## Memory System (READ FIRST)
At the **start of every session**, read all files in the `memory/` directory:
- `memory/user.md` — Who Scott is, his company, technical profile
- `memory/preferences.md` — Working style, dev conventions, deployment patterns
- `memory/decisions.md` — Architecture choices, system designs, key decisions made
- `memory/people.md` — Contacts, clients, team members

These files are the shared knowledge base across all Claude sessions (Code and Cowork).
**Do not ask questions that are already answered in these files.**

## Updating Memory
At the **end of a session** (or when significant new information comes up), update the relevant memory file(s):
- New architecture decisions → `decisions.md`
- New contacts or team info → `people.md`
- Changed preferences or conventions → `preferences.md`
- Updated user profile info → `user.md`

Keep entries concise. Use bullet points. Include dates for time-sensitive decisions.
Do not duplicate information across files — each file has a clear purpose.

## Key Project Paths
- **This directory:** OneDrive-synced Fortivo company documents
- **Active Jobs (SharePoint):** `~/Library/CloudStorage/OneDrive-SharedLibraries-FortivoPropertyServices/Active Jobs - Documents/01_Active Jobs/01_Job Name/`
- **T&M Tracker PWA:** `Fortivo - Documents/11_Templates/01_Client-Facing/03_Rate Sheets/T&M HTML/index.html`
- **Job Manager SPA:** SharePoint Site Assets → `fortivo_app.html`
- **Voice Email:** `~/fortivo-voice-email/`

## Working Style
- Proceed without excessive confirmation — only ask before destructive actions
- Core features first, iterate after
- Keep iPhone PWA URLs stable across deploys
- Single-file HTML apps preferred for SharePoint compatibility
