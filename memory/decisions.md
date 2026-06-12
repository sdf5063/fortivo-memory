# Decisions & Architecture Log

## DFR Voice Tier 1 Upgrade (2026-05-02)
Built on top of the 2026-04-03 DFR overhaul. Four upgrades, all backwards-compatible — no existing data touched.

### 1. Sonnet 4.5 + few-shot examples
- Relay (`fortivo-dfr-api/api/structure.js`) switched from `claude-haiku-4-5-20251001` → `claude-sonnet-4-5-20250929`
- System prompt now includes 3 few-shot examples (water mit + safety incident, restoration with sub on site, refinement correction example)
- max_tokens raised 1800 → 2500 to fit the larger examples + confidence object
- Cost: ~3x Haiku per call but DFR parse runs ≤2x per report. Negligible monthly.

### 2. Confidence scoring per field
- Relay returns `_confidence: {fieldName: "high"|"med"|"low"}` for every field it populates
- `applyAutofillHighlight(el, confidence)` paints fields:
  - `field-conf-high` → green border (fades after 8s)
  - `field-conf-med`  → blue border (fades after 8s)
  - `field-conf-low`  → yellow border + "VERIFY" badge that **persists until the user edits the field**
- Banner now shows "X fields filled — N low-confidence (yellow) — please verify"

### 3. Yesterday's DFR pre-fill
- New `Views.reportForm` banner appears when a prior DFR exists for the selected job
- One-tap `App.useYesterdayDFR()` pre-fills: `crewSize`, `crewNames`, `arrivalTime`, `departureTime`, `workCategory`, `location`, `equipment`, `materialsUsed`, `subcontractors`
- **Today's `workPerformed` seeds from yesterday's `nextDayPlan`** (continuity)
- Read-only on prior reports — prior data is never modified
- Skips fields the user has already typed into

### 4. Voice refinement / merge mode
- After first successful parse, `_dfrParsedOnce = true` and parse button switches to purple "Refine with new dictation"
- User taps mic again, dictates corrections/additions, taps Refine
- Relay receives `{mode: 'refine', existingDFR: <current form state>, transcript: <new content only>}`
- Sonnet keeps existing values unchanged unless explicitly contradicted; updates confidence to "high" on changed fields
- Refine mode allows un-checking checkboxes (initial mode only sets true)
- `clearTranscript()` resets refine state → next parse is initial again

### State machine
- `_dfrTranscriptFinal` — accumulated final transcript (live)
- `_dfrPrevTranscript` — what was sent in last parse (used to extract NEW content for refine)
- `_dfrParsedOnce` — flag toggling button mode
- All three reset on `Views.reportForm` open and on `clearTranscript()`

### Files touched
- `~/fortivo-dfr-api/api/structure.js` (rewritten — backup at `structure.js.bak`)
- `Fortivo Operations - Site Assets/fortivo_app.html`
  - Backup: `_backups/fortivo_app_2026-04-03_tier1.html`
  - CSS: confidence variants, yesterday banner, refine button styling
  - `Views.reportForm`: state reset + yesterday banner
  - `parseNarrative`: refine logic + confidence handling
  - `applyAutofillHighlight(el, confidence)`: 3-tier styling
  - New: `_dfrCollectFormState()`, `useYesterdayDFR()`
  - New const: `DFR_FIELD_MAP` (single source of truth for all 19 fields)

### Deployment
- Vercel: `fortivo-dfr-api.vercel.app/api/structure` redeployed 2026-05-02
- App: not yet deployed to SharePoint (`SitePages/fortivo_app.aspx`) — pending Scott's deploy step

## Property Type + Residential Client Rollup (2026-04-03)

### New SP Column: Property_Type on Jobs_Master
- Choice field: Residential, Commercial, Industrial, Multifamily, Government, Healthcare
- Add via browser JS (see snippet below — run once in SP authenticated console)
- Maps to `propertyType` in JS model; also writes `Client_Category` for backward compat
- `jobFromSP` reads `Property_Type` first, falls back to `Client_Category`
- `jobToSP` writes both `Property_Type` and `Client_Category`

### SP Column Creation Snippet (run once in browser console on SP site)
```javascript
const d = await fetch('/sites/FortivoOperations/_api/contextinfo',{method:'POST',headers:{'Accept':'application/json;odata=nometadata'}}).then(r=>r.json());
await fetch("/sites/FortivoOperations/_api/web/lists/getbytitle('Jobs_Master')/fields", {
  method:'POST', headers:{'Accept':'application/json;odata=verbose','Content-Type':'application/json;odata=verbose','X-RequestDigest':d.FormDigestValue},
  body: JSON.stringify({'__metadata':{'type':'SP.FieldChoice'},'Title':'Property_Type','FieldTypeKind':6,'Required':false,'Choices':{'results':['Residential','Commercial','Industrial','Multifamily','Government','Healthcare']}})
});
```

### Create "Residential Client" Master Account (run once in browser console)
```javascript
const d = await fetch('/sites/FortivoOperations/_api/contextinfo',{method:'POST',headers:{'Accept':'application/json;odata=nometadata'}}).then(r=>r.json());
await fetch("/sites/FortivoOperations/_api/web/lists/getbytitle('CRM_Accounts')/items", {
  method:'POST', headers:{'Accept':'application/json;odata=verbose','Content-Type':'application/json;odata=verbose','X-RequestDigest':d.FormDigestValue},
  body: JSON.stringify({'__metadata':{'type':'SP.Data.CRM_x005f_AccountsListItem'},'Title':'Residential Client','Account_Name':'Residential Client','Account_Type':'Client','Account_Status':'Active'})
});
```

### Job Manager (fortivo_app.html) Changes
- `Property Type` dropdown replaces `Client Category` label in job detail view (always visible as badge in view mode, select in edit mode)
- CRM note shown inline: Residential → "grouped under Residential Client"; others → "tracked under [Client Name]"
- `_onPropertyTypeChange(val, clientName)` updates note divs in both job detail and new job form
- New job form: `njPropertyType` (was `njClientCategory`), same options
- Phase-to-rebuild carries `propertyType` forward from original job
- `_autoCreateResidentialContact` checks `propertyType` first, falls back to `clientCategory`

### CRM (fortivo_crm.html) Changes
- Property Type chip row added below account type chips: All | Residential | Commercial | Multifamily | Industrial | Government | Healthcare
- `_activePropTypeFilter` state + `_propTypeChip(val)` function
- Residential chip → filters to only "Residential Client" master account
- Other type chips → filter accounts whose linked jobs have that Property_Type
- Residential Client account Jobs tab → shows all Jobs_Master rows with Property_Type=Residential (live aggregation, not job links), with individual client names, ROM, status
- `_autoImportClients` skips residential job client names (they don't get individual accounts)
- `fClientCategory` dropdown hidden (kept in DOM for compat)

## 2026-04-03 Session: DFR Filename, Field_Reports Columns, Search Bar

### DFR PDF Filename Format (fixed)
- Changed from `Fortivo DFR_MM.DD.YYYY.pdf` → `[Job Number]; DFR: MM.DD.YYYY.pdf`
- Example: `26-01-00015; DFR: 04.03.2026.pdf`
- Changed in `generateDFRPdf()` (~line 6370). Both `_doDFRDownload` and `generateAndUploadDFR` use the return value, so one change fixes both.

### Field_Reports SP Columns Added (via browser JS)
Added 5 new columns to `Field_Reports` list to support DFR overhaul fields:
- `Weather_Conditions` (FieldTypeKind 3 = Note/multiline)
- `Materials_Used` (Note)
- `Subcontractors` (Note)
- `Additional_Notes` (Note)
- `Report_Number` (FieldTypeKind 2 = Text)

### Universal Search Bar — Phone Search Fix + Deploy
- Search bar was already implemented across all 4 apps (done in prior session)
- Fixed gap: phone number search was fetched but not matched — added `match(c.Phone,q)` to contacts filter in all 4 apps
- Backups: `_backups/*_2026-04-03_search.html`
- All 4 apps deployed and end-to-end tested:
  - "Shah" → Jobs + Accounts + Invoices ✓
  - "Warfield" → Jobs + Invoices ✓
  - "Bob Patten", "Tiffany" → Contacts ✓
  - 1-char minimum enforced ✓, Escape closes ✓, keyboard nav ✓
  - Touch targets 51px (>44px req.) ✓, full-width on mobile ✓
  - Cross-app URL format: `fortivo_crm.aspx#/accounts/[spId]` verified correct

## DFR Voice-to-Text Overhaul (2026-04-03)

### Primary API relay (dedicated)
- **Project**: `~/fortivo-dfr-api/` → deployed at `https://fortivo-dfr-api.vercel.app`
- **Endpoint**: `POST /api/structure` — accepts `{transcript, jobContext}`, returns `{ok, dfr}` with 19 structured fields
- **Env vars on Vercel**: `ANTHROPIC_API_KEY` (Claude key), `API_KEY=fortivo-voice-2026`
- **Model**: Claude Haiku (`claude-haiku-4-5-20251001`)

### Secondary relay (in voice-email project)
- `fortivo-voice-email.vercel.app/api/structure-dfr` — also deployed, same logic, uses `CLAUDE_API_KEY` env var

### App changes (`fortivo_app.html`)
- **App URL**: `DFR_API = 'https://fortivo-dfr-api.vercel.app/api/structure'`; `DFR_API_KEY = 'fortivo-voice-2026'`
- **No user API key setup needed** — removed LLM settings UI entirely; all calls go through Vercel relay
- **New DFR form sections**: Weather, Materials Used, Subcontractors, Additional Notes, Report Number (auto DFR-NNN)
- **New SP mapper fields**: `Weather_Conditions`, `Materials_Used`, `Subcontractors`, `Additional_Notes`, `Report_Number` — must add these columns to `Field_Reports` SP list before they round-trip
- **Voice UI**: 72px pulsing mic button, live transcript panel, recording timer (M:SS), ■ stop icon
- **Transcript**: IIFE-scoped `_dfrTranscriptFinal` (not a textarea); saved to `narrative` field on submit
- **PDF**: `_doDFRDownload(reportId)` (exposed as `App.generateDFRPdf`) → jsPDF, navy header/footer, all 12 DFR sections, filename `Fortivo DFR_MM.DD.YYYY.pdf`
- **SP auto-upload**: `App.generateAndUploadDFR(reportId)` → downloads PDF + searches Active/Closed job folders for one starting with job number, uploads to `13_Daily Field Reports` subfolder
- **Backup**: `_backups/fortivo_app_2026-04-03.html`

### Remaining (not yet done)
- Add new Field_Reports SP columns in SharePoint admin (`Weather_Conditions`, `Materials_Used`, `Subcontractors`, `Additional_Notes`, `Report_Number`)
- Deploy updated `fortivo_app.html` → `fortivo_app.aspx` on SharePoint

## Quick Wins Batch (2026-04-03)

### Universal App Nav Bar
- Slim 32px fixed nav bar added to all 4 apps: Dashboard, Job Manager, CRM, Invoicing
- CSS: `.fv-appnav` (position:fixed, top:0, z-index:9999, #19345E bg, #1292D1 bottom border)
- Active app highlighted with `.fv-active` class (accent blue background)
- SP links: `/sites/FortivoOperations/SitePages/[appname].aspx`
- Sidebar apps: `.app-shell` changed to `height:calc(100vh - 32px); margin-top:32px`
- Dashboard: `body { padding-top:32px }` instead

### Python Script Backups
- `fv_qb.py`, `fv_sync.py`, `fv_server.py` copied to `Site Assets/_python_backups/`
- Source: Site Assets root (fv_qb.py) and `_backups/` timestamped copies

### PWA Service Worker (Dashboard, CRM, Invoicing)
- Shared `sw.js` rewritten as `fortivo-ops-v1` cache — covers CDN libs for all 3 apps
- Strategy: cache-first for CDN (cdnjs, cdn.jsdelivr, unpkg, fonts, msal), network-first for HTML/ASPX, skip SP API calls
- Per-app manifests: `manifest_dashboard.json`, `manifest_crm.json`, `manifest_invoicing.json`
- SW registered with scope `/sites/FortivoOperations/` — registration failure is silent (catches error)
- Note: Job Manager explicitly unregisters its SW (comment: "no longer using PWA SW") — not touched

### CRM Health Score — Account List Column
- Health score badge now shown in account LIST view (was only in account detail)
- `_filterAccounts()` now calls `computeHealthScore()` per row using pre-fetched contacts+activities
- Added `<th>Health</th>` column to account table header
- Algorithm was already correctly implemented per spec (30+20+20+15+15 pts)

### Dashboard Date Filters + CSV Export
- Added From/To date inputs to Jobs tab filters bar (filter on `j.date` = dateReceived)
- `getJobTableFiltered()` updated to apply date range
- Export CSV button in Jobs table header → `exportJobsCSV()` downloads current filtered view
- CSV columns: Job ID, Client, Address, State, Phase, Status, ROM, QB Revenue, COGS, Expenses, Margin%, Invoiced, Paid, Balance, Date Received
- Filename format: `fortivo_jobs_YYYY-MM-DD.csv`

### Invoice Delete Fix
- `spDeleteItem(list, itemId)` function was missing from `fortivo_invoicing.html`
- Added after `spUploadFile()` — uses DELETE method with `IF-MATCH:*` and `X-HTTP-Method:DELETE` headers
- `deleteInvoice()` already had the confirmation dialog and correct call — only the function definition was missing

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

## CRM Phase 1 Overhaul (2026-04-02)
- **New SP List: CRM_Deals** — 14 fields: Title, Account_Id, Account_Name, Contact_Id, Contact_Name, Deal_Stage (Choice: Lead/Qualified/Proposal/Negotiation/Won/Lost), Deal_Value (Number), Expected_Close_Date (DateTime), Probability (Integer), Source (Text), Notes (Note), Created_Date (DateTime), Last_Updated (DateTime), Owner (Text), Lost_Reason (Text)
- Entity type: `SP.Data.CRM_x005f_DealsListItem`
- In-app SP list creation: `_setupCRMDealsList()` / `_runCRMDealsSetup()` — run once from Pipeline view (⚙ button)
- **Pipeline Kanban**: drag-drop across stages, per-stage value totals, stale-deal alerts (14+ days), weighted forecast
- **Computed Health Score**: replaces static `healthScore=50` — 5 dimensions (activity recency 30pt, follow-up compliance 20pt, revenue 20pt, contact completeness 15pt, activity volume 15pt). Displayed as colored badge on account overview. Score NOT written back to SP (computed at render time to avoid write load)
- **Dashboard widgets**: pipeline funnel, weighted forecast KPI, closing-this-month count, stale deals, quick-log templates
- **360° Contact Timeline**: unified events list (activities + deals + job links + last-contacted) sorted reverse-chronological. Deal mini-list shown on contact detail above timeline.
- **Quick Log**: 6 one-tap templates from dashboard (voicemail, email, meeting, capability statement, site visit, note). Full account/contact picker + custom templates inline.
- **Deal CRUD routes**: `#/pipeline` (kanban), `#/deals/new`, `#/deals/:id` (detail + stage progress bar + move buttons)
- **Backward compatible**: all existing routes (#/accounts, #/contacts, #/referrals, #/followups, etc.) unchanged. CRM_Deals load uses `.catch(() => [])` so SP list missing = graceful empty state.
- Backup taken: `_backups/fortivo_crm_2026-04-02.html`

## CRM Outlook Calendar Sync (2026-04-01)
- Built into `fortivo_crm.html` — no separate service, all client-side
- Uses MSAL Browser v2 (already loaded) + Microsoft Graph API `POST/PATCH /me/events`
- Auth: delegated `Calendars.ReadWrite` scope via `acquireTokenSilent` → `acquireTokenPopup` fallback
- Uses existing Azure AD app "Fortivo Voice Email" (ID: cbec554b-4222-4c5b-a1b8-e51133955bb3)
- Tenant resolved at runtime from `_spPageContextInfo.aadTenantId` (falls back to `organizations`)
- Event ID stored in new SP field `Calendar_Event_Id` (Text) on `CRM_Contacts` list
- Syncs on: save contact, save follow-up, snooze; deletes on: complete follow-up
- Calendar events: 9 AM ET, 15 min duration, 15 min reminder, subject "BD Follow-Up: [Name] — [Account]"
- Fire-and-forget after SP save — calendar failure shows warning toast but doesn't block save
- Requires Azure AD app changes before going live (see Azure AD requirements doc)

## Invoice Generator (2026-04-02)
- **File: `fortivo_invoicing.html`** → deploy as `fortivo_invoicing.aspx`
- **SP List: `Invoices`** — 21 fields: Title (=invoice #), Job_Number, Job_Name, Client_Name, Property_Address, Billing_Type (Choice: Xactimate/T&M/Fixed Price), Invoice_Date, Due_Date, Sent_Date, Paid_Date (DateTime), Amount, Tax_Amount, Total (Integer), Status (Choice: Draft/Sent/Paid/Overdue/Cancelled), Payment_Reference, QB_Invoice_Number, Notes, PDF_URL, Line_Items_JSON (Note — JSON), Created_Date, Last_Updated
- Entity type: `SP.Data.InvoicesListItem`
- In-app SP list creation: `runSetup()` — run once from sidebar "Setup SP List"
- **Invoice numbering**: INV-YYMM-NNN (e.g., INV-2604-001), auto-increments by querying last INV-YYMM-* record
- **Billing types**: Xactimate, T&M, Fixed Price — all use same line items JSON structure
- **4-step wizard**: billing type → job & dates → line items → review/save
- **PDF**: jsPDF 2.5.1 + jspdf-autotable 3.8.2 (CDN), LOGO_WHITE on navy header, autoTable for line items, auto-uploads to job's `08_Invoices` SP folder
- **QBO Phase 1**: CSV export only (manual import), filters: all/not-in-QBO/selected
- **Mark Paid** flow records paid date, payment ref, QB invoice # back to SP
- **Hash routing**: `#/dashboard`, `#/invoices`, `#/create`, `#/invoice/:id`, `#/qbo`
- **Job Manager deep-link**: `fortivo_app.aspx#/invoices` to navigate from Job Manager
- **Company info for PDFs**: Fortivo Property Services, LLC — P.O. Box 83161, Gaithersburg, MD 20883-83161 — (301) 822-2266 — AR@Fortivo.com — Federal ID 39-3559334 — www.Fortivo.com
- Default payment terms: **Net 25** (not Net 30)
- CC surcharge notice: "Payments by credit cards are assessed a 3% processing fee"

### Overhaul Planned (2026-04-02)
- Full plan file: `Fortivo Operations - Site Assets/INVOICING_OVERHAUL_PLAN.md`
- **PDF overhaul**: Two distinct PDF builders — Xactimate/Fixed Price matches `Fortivo Invoice Template.pdf`, T&M matches `Invoice Details_Final Template.pdf` (both in `02_Finance/04_Invoicing/`)
- **T&M integration**: Category-based entry (Billable Labor, Materials, Equipment, Subcontractors, Fuel, Admin) — future pull from T&M Tracker PWA
- **QBO one-time import**: Bulk import existing QBO invoices for numbering continuity (CSV approach)
- **Bug fixes**: Wizard back-nav state loss, em dash rendering (use literal chars not entities), sync dot status
- **Deploy rule**: Always write `/tmp/` first, then `cp` to OneDrive (avoids sync conflicts that corrupt deploys)
- **Escaped script tags killed rendering** — `<\/script>` in the file broke the browser; fixed 2026-04-02

## Universal Search Bar (2026-04-03)
- Added to all 4 apps: `fortivo_app.html`, `fortivo_crm.html`, `fortivo_invoicing.html`, `Fortivo_Dashboard.html`
- Self-contained `<style>` + `<div id="fv-search-wrap">` + `<script>` block inserted after `</nav>` in each file
- Position: `fixed; top:32px` (directly below nav strip); background #19345E matching nav
- `app-shell` margin-top updated 32px → 70px (and `calc(100vh - 32px)` → 70px) in all 3 SPA files; Dashboard `body padding-top` updated same
- Data: fetches Jobs_Master, CRM_Accounts, CRM_Contacts, Invoices on first focus; in-memory cache
- Groups results: 🏗 Jobs, 🏢 Accounts, 👤 Contacts, 📄 Invoices (max 5/4/4/4 per type)
- Keyboard: ArrowUp/Down, Enter to navigate, Escape closes
- Navigation URLs: Jobs→fortivo_app.aspx#/jobs/[Job_Number], Accounts→fortivo_crm.aspx#/accounts/[Id], Contacts→fortivo_crm.aspx#/contacts/[Id], Invoices→fortivo_invoicing.aspx#/invoice/[Id]
- Tested: "26-01" job results ✓, "Shah" client match ✓, Escape ✓, ArrowDown highlight ✓

## DFR Filename Fix (2026-04-03)
- Changed `'Fortivo DFR_MM.DD.YYYY.pdf'` → `'[JobNumber]; DFR: MM.DD.YYYY.pdf'`
- e.g. `26-01-00015; DFR: 04.03.2026.pdf`
- Uses `r.jobNumber` from the report object; falls back to `'Unknown'`

## Field_Reports SP Columns Added (2026-04-03)
- Added 5 new columns to `Field_Reports` list via REST API:
  - `Weather_Conditions` (Note/FieldTypeKind:3)
  - `Materials_Used` (Note/FieldTypeKind:3)
  - `Subcontractors` (Note/FieldTypeKind:3)
  - `Additional_Notes` (Note/FieldTypeKind:3)
  - `Report_Number` (Text/FieldTypeKind:2)

## CRM Mobile UX + Reports Fix (2026-04-03)
Backup: `_backups/fortivo_crm_2026-04-03_uxfix.html`

### Issues Fixed
1. **Jobs tab count for Residential Client** — was always showing (0) because it counted `CRM_Job_Links` (empty for that account). Now computes count from `Jobs_Master` using the same residential filter logic as the jobs tab itself. Count computed inline before `el.innerHTML` in `accountDetail`.
2. **Mobile scroll cutoff** — Added `-webkit-overflow-scrolling:touch` + `overscroll-behavior:contain` to `.page-container`. Added `padding-bottom:calc(var(--bottom-nav-height)+16px+env(safe-area-inset-bottom))` in mobile media query so content isn't hidden behind bottom nav. Wrapped residential jobs table in `<div class="table-wrap">` for proper horizontal scroll.
3. **Account filter inconsistency** — Bug: `_filterAccounts()` used `fType.value || _activeChipFilter` — when fType was reset to "" but `_activeChipFilter` still held a stale value, filtering persisted incorrectly. Fixed: removed `|| _activeChipFilter` fallback; `_chipFilter()` already syncs `fType.value` so the dropdown is always authoritative.
4. **Clickable job numbers** — Both residential jobs table and linked jobs table now have `<a href="fortivo_app.aspx#/jobs/[jobNum]">` on the job number cell (plus `onclick` still on the whole row). `event.stopPropagation()` prevents double-navigation.
5. **Reports section** — New `Views.reports` view + `_exportReport(type)` function. Route: `#/reports`. Nav item: "Export Reports" added under Reports section in sidebar. 5 report types: All Accounts, All Contacts, Pipeline/Deals, Activity Log (date range), Follow-Up Report (date range). Each exports a timestamped `.csv` file via `URL.createObjectURL`. `_exportReport` exposed in App object.
6. **Mobile tap targets** — In mobile media query: `.btn-sm { min-height:44px }`, `.tab-btn { padding:14px 16px }`, `.clickable-row { -webkit-tap-highlight-color }` for visual tap feedback.

## T&M Tracker — Phase 1 "Stop the Bleeding" (2026-06-12)
- Full ecosystem review (15-agent workflow) produced `T&M HTML/ROADMAP.md` — 6-phase plan; Phase 1 implemented same day (v3.33/v3.34)
- **Git repo:** private https://github.com/sdf5063/fortivo-tm-tracker (baseline = v3.32); commit + push after every change set
- **Tests:** `tests/phase1.test.js` (22 assertions) runs via `npm test` AND blocks `npm run build`/deploy on failure
- Equipment D/W/M override now persists in project JSON (was silently reverting to auto)
- Rate-version safety: no silent fallback to 2026-V1.4; blocking modal + `ratesSnapshot` embedded in every project file (backfilled on open); unknown line items preserved as `orphaned:true` with saved totals, never dropped
- iPhone session resume: green Resume banner on setup screen (localStorage buffer now carries dirty flag + lastSavedAt)
- `.history/` folder inside each job's T&M dir: last 10 project-file versions auto-kept before each save; deleted days' legacy files archived there too
- `changeRateVersion`/`_recomputeAllDayTotals` are transactional (rollback on failure)
- Save button shows orange "Save •" when unsaved; sticky error toasts with Retry
- deploy.sh archives index.html to `.backups/` (keep 20) and aborts on build failure

## T&M Tracker — Phases 2-4 shipped (2026-06-12, v3.36)
- **Phase 4 (rates, the crucial one):** rate update = 2 steps: (1) browser Rate Editor → Publish to Production (reaches all trackers in seconds via runtime hydration from /api/rates-current); (2) double-click `03_Rate Sheets/Update Rate Sheets.command` → tests → deploy → client Word/PDF regenerated from `_Template/` master → Finder opens output. Live blob activated + loop verified end to end.
- Client doc format master: `03_Rate Sheets/_Template/Fortivo T&M Rate Sheet_TEMPLATE.docx` (edit wording/layout HERE). Generator finds tables by header signature; year-rollover automatic; meta.json + .backups in each Version folder.
- **Published rate versions are immutable** (publish.js rejects modify/remove with 409 — duplicate to new version instead).
- Phase 2: migrateProject() validates every load; _extra fields round-trip (older client can't strip newer data); verifyDayTotals drift toast; day PDFs stamp locked rate version; totals.grandTotalExclAdmin added.
- Phase 3: crew carry-forward to new days; device roster for name suggestions; scoped UNDO on day-delete/rate-change; offline draft outbox + auto-flush; day-add popover (any date, auto-resequence).
- Tests now 34 assertions; still gate every deploy. Remaining: Phase 3 leftovers (hour preset chips, job picker search, iPhone job validation), Phase 5 (dry-run diff, rename detection, TEMPLATE_CONTRACT.md, Excel updater), Phase 6 (multi-PM locks, identity, invoice freeze, QBO CSV).

## T&M Tracker — Phases 5+6 + Phase 3 leftovers shipped (2026-06-12, v3.37) — ROADMAP COMPLETE
- **Phase 5:** generator is verifiable — dry-run diff w/ y/N approval before any Word write; idempotent value-agnostic regexes (2nd run on own output = zero changes); rename detection blocks duplicate-item sheets (SYNONYMS = escape hatch); `--check-template` contract check runs before every generation; `TEMPLATE_CONTRACT.md` documents anchors/tables; run-boundary text replacement preserves mixed formatting; PDF verified (exit code + page count).
- **Phase 6:** identity-lite (one-time device-user prompt; attribution everywhere + 50-entry changeLog in project meta); optimistic concurrency on save (conflict copy default / overwrite / abort); advisory `.lock.json` (10-min refresh, 4h stale) warns on simultaneous open; **invoice freeze** — Export Project Report snapshots totals to `meta.invoices[]`, drift banners on summary; **Export Invoice Data (CSV)** in save menu (QBO-ready); Needs-invoice/Invoiced badges on setup preview; drafts carry user/deviceId/jobId.
- Phase 3 leftovers: 8/10/12 hr chips, searchable recency-first job picker, job-number validation, affordance captions.
- Deferred (revisit when needed): per-job localStorage buffers, draft PATCH-assignment, /api/jobs phone list, Excel auto-updater, key rotation (FALLBACK_KEY still in functions).
