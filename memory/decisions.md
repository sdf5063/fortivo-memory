# Decisions & Architecture Log

## Data Architecture
- **SP Lists are source of truth** (as of 2026-03-08) — JSON files are fallbacks only
- Job numbering format: YY-PP-NNNNN (e.g., 25-01-00122)
- Jobs_Master financial fields are Integer — must round before writing
- Date_Received is DateTime — must send as ISO 8601
- No ROM column in Jobs_Master — Amount serves as ROM
- Equipment_Inventory uses Description0 (not Description — reserved by SP)

## T&M Tracker PWA (v3.2)
- Single-file offline-capable PWA at `T&M HTML/index.html`
- Live: https://fortivo-tm-tracker.netlify.app
- Multi-day project system with day tabs
- Rate version locked per project
- Admin charges (project doc fee, debris, emergency) applied ONCE per project, not per day
- 3-tier file output: File System Access API → Web Share API → download link
- Service worker caches page + jsPDF CDN for offline use

## Rate Sync Contract
- When rates change, update TWO files: `index.html` (RATE_VERSIONS) + Excel tracker
- Then bump service worker cache version and redeploy

## Voice Email System (Fortivo Dictate)
- Built 2026-03-11, deployed on Vercel
- Production: https://fortivo-voice-email.vercel.app
- No Azure subscription (M365 only) — Vercel for hosting, Azure AD for auth
- Future: Job Manager integration (query Jobs_Master by client name)

## Job Manager SPA
- `fortivo_app.html` → deployed as `fortivo_app.aspx`
- Modular architecture: SP helper, router, offline queue, auth modules
- Hash-based client routing (#/jobs, #/equipment, #/reports, etc.)
- Phase 2 stubs: moisture readings, materials tracker, CRM
