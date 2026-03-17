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
- SharePoint: upload to SiteAssets, copy to SitePages as .aspx, then Publish
- DenyAddAndCustomizePages must be Disabled (1) for custom pages

## Tools & Services
- GitHub account: sfutrovsky@gmail.com
- Netlify site: `fortivo-tm-tracker` (T&M Tracker PWA)
- Azure AD App: "Fortivo Voice Email" (ID: cbec554b-4222-4c5b-a1b8-e51133955bb3)
- Anthropic API: Claude Haiku for voice email formatting
