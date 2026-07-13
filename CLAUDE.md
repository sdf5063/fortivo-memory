# Fortivo Property Services — Claude Instructions

## Memory System (READ FIRST)
At the **start of every session**, read all files in the `memory/` directory:
- `memory/user.md` — Who Scott is, his company, technical profile
- `memory/preferences.md` — Working style, dev conventions, deployment patterns
- `memory/decisions.md` — Architecture choices, system designs, key decisions made
- `memory/people.md` — Contacts, clients, team members

These files are the shared knowledge base across all Claude sessions (Code and Cowork).
**Do not ask questions that are already answered in these files.**

**GitHub repo:** https://github.com/sdf5063/fortivo-memory
Raw URLs for Cowork access:
- `https://raw.githubusercontent.com/sdf5063/fortivo-memory/main/memory/user.md`
- `https://raw.githubusercontent.com/sdf5063/fortivo-memory/main/memory/preferences.md`
- `https://raw.githubusercontent.com/sdf5063/fortivo-memory/main/memory/decisions.md`
- `https://raw.githubusercontent.com/sdf5063/fortivo-memory/main/memory/people.md`

## Updating Memory
At the **end of a session** (or when significant new information comes up), update the relevant memory file(s):
- New architecture decisions → `decisions.md`
- New contacts or team info → `people.md`
- Changed preferences or conventions → `preferences.md`
- Updated user profile info → `user.md`

Keep entries concise. Use bullet points. Include dates for time-sensitive decisions.
Do not duplicate information across files — each file has a clear purpose.

After updating local memory files, push changes to GitHub:
```bash
cd /tmp/fortivo-memory && cp ~/Library/CloudStorage/OneDrive-FortivoPropertyServices/memory/*.md memory/ && cp ~/Library/CloudStorage/OneDrive-FortivoPropertyServices/CLAUDE.md . && git add -A && git commit -m "Update memory" && git push
```

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
