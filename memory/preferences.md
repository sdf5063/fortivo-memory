# Preferences & Working Style

## Communication
- Prefers proceeding without excessive permission asks
- Only confirm before destructive actions (deletions, force pushes)
- Keep responses concise — skip summaries of what was just done
- Phase approach: core features first, then iterate

## Development
- Favors single-file HTML apps (inline CSS + JS) for SharePoint compatibility
- Brand colors: --primary (#19345E dark blue), --accent (#1292D1 light blue)
- Fortivo logo should be base64-embedded in all client-facing HTML/PDF
- iPhone home screen shortcuts must keep working (same URLs after deploys)
- Equipment status terms: Standby (not "Available"), Deployed, In Repair, Retired
- Job status choices: Open, In Progress, On Hold, Closed, Cancelled

## Deployment
- Netlify: `npx netlify-cli deploy --dir=. --prod` or `./deploy.sh`
- SharePoint: save master .html to synced "Fortivo Operations - Site Assets" folder, wait for OneDrive sync, then REST CopyTo → SitePages/*.aspx + Publish (run from authenticated browser console)
- DenyAddAndCustomizePages must be Disabled (1) for custom pages
- **⚠ SP auto-reverts DenyAddAndCustomizePages every ~24h (Microsoft policy since late 2024).** Symptom: all custom .aspx pages return "File Not Found" and CopyTo returns 403. Fix before every deploy — POST this CSOM XML to `https://fortivopropertyservices-admin.sharepoint.com/_vti_bin/client.svc/ProcessQuery` (with admin-site X-RequestDigest) from an authenticated admin-center tab:
  `<Request AddExpandoFieldTypeSuffix="true" SchemaVersion="15.0.0.0" LibraryVersion="16.0.0.0" ApplicationName="Javascript Library" xmlns="http://schemas.microsoft.com/sharepoint/clientquery/2009"><Actions><ObjectPath Id="2" ObjectPathId="1"/><ObjectPath Id="4" ObjectPathId="3"/><SetProperty Id="5" ObjectPathId="3" Name="DenyAddAndCustomizePages"><Parameter Type="Enum">1</Parameter></SetProperty><Method Name="Update" Id="6" ObjectPathId="3"/></Actions><ObjectPaths><Constructor Id="1" TypeId="{268004ae-ef6b-4e9b-8425-127220d84719}"/><Method Id="3" ParentId="1" Name="GetSitePropertiesByUrl"><Parameters><Parameter Type="String">https://fortivopropertyservices.sharepoint.com/sites/FortivoOperations</Parameter><Parameter Type="Boolean">true</Parameter></Parameters></Method></ObjectPaths></Request>`
  Allow 1–5 min propagation (write path recovers first, page-serving path after).

## Product Priorities (2026-07-13)
- CRM is currently dormant — keep it wired into new systems (nav links, graceful data reads) but never make it a dependency; skip CRM sections when its lists are empty

## Tools & Services
- GitHub account: sfutrovsky@gmail.com
- Netlify site: `fortivo-tm-tracker` (T&M Tracker PWA)
- Azure AD App: "Fortivo Voice Email" (ID: cbec554b-4222-4c5b-a1b8-e51133955bb3)
- Anthropic API: Claude Haiku for voice email formatting
