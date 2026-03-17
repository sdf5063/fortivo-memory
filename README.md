# Fortivo Memory

Shared memory files for Claude sessions (Claude Code + Cowork).

## Usage

### In Cowork
At the start of a session, tell Claude:
> Read my memory files from https://github.com/sdf5063/fortivo-memory

Claude can fetch the raw files:
- `https://raw.githubusercontent.com/sdf5063/fortivo-memory/main/memory/user.md`
- `https://raw.githubusercontent.com/sdf5063/fortivo-memory/main/memory/preferences.md`
- `https://raw.githubusercontent.com/sdf5063/fortivo-memory/main/memory/decisions.md`
- `https://raw.githubusercontent.com/sdf5063/fortivo-memory/main/memory/people.md`

### In Claude Code
The `CLAUDE.md` file instructs Claude to read `memory/` at session start automatically.

## Files
- `CLAUDE.md` — Session instructions
- `memory/user.md` — User profile, company, technical background
- `memory/preferences.md` — Working style, dev conventions, deploy patterns
- `memory/decisions.md` — Architecture choices, system designs
- `memory/people.md` — Contacts, clients, team members
