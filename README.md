# Agentnetes Status Page

Public status page at <https://status.agentnetes.io>, served from this repo via GitHub Pages.

## Architecture

- `index.html` — the page itself (vanilla HTML/CSS/JS, no build step)
- `status.json` — the live data source the page polls every 60s
- `CNAME` — binds GitHub Pages to `status.agentnetes.io`

## Updating component status

Edit `status.json` and push to `main`. GitHub Pages re-deploys within ~30s; clients see the update on their next 60s poll.

```bash
gh repo clone Fiv21/agentnetes-status
cd agentnetes-status
# edit status.json
git commit -am "ops: degrade Sync to monitoring"
git push
```

### Status values

- `operational` — green
- `degraded`, `partial`, `monitoring`, `investigating`, `identified` — yellow
- `outage`, `down`, `major` — red

## Adding an incident

Push an entry to `incidents` in `status.json`:

```json
{
  "incidents": [
    {
      "title": "Sync intermittent on EU clients",
      "severity": "SEV2",
      "message": "Investigating elevated 5xx on cloud-service from EU regions.",
      "created_at": "2026-05-02T14:23:00Z"
    }
  ]
}
```

The page renders the most recent first. Resolved incidents can be removed once the postmortem lands at `docs/postmortems/`.

## Future automation

- Cloud Monitoring webhook → small handler that POSTs an updated `status.json` here when a 5xx-rate / p99 / uptime alert fires
- Per-component uptime stats from GCP Monitoring metrics

## Why this and not Instatus / Statuspage?

Free tier of Instatus doesn't support custom domains; paid tier is $20+/mo. GitHub Pages on a public repo is free, the design matches the rest of the brand (dark surface, teal accent, JetBrains Mono numbers in the future), and we already had Cloudflare DNS to point the CNAME.
