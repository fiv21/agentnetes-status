# Agentnetes Status Page

Live health for the Agentnetes platform at <https://status.agentnetes.io>.

Each visitor's browser probes every component endpoint every 30 seconds. No backend, no polling cron, no static-snapshot lag.

## Architecture

- `index.html` — vanilla HTML/CSS/JS. Probes 6 endpoints in parallel via `fetch(..., { mode: "no-cors" })`. Slow (>1.5s) is yellow; failed/timed-out is red; otherwise green.
- `incidents.json` — manually-curated incident list. Drop in entries when something goes sideways; the page renders the most recent first.
- `CNAME` — binds GitHub Pages to `status.agentnetes.io`.

## Why client-side probes

A static-snapshot page (Statuspage / Instatus / a JSON committed to a repo every 5 min) reports stale state. A page that hits the actual endpoints from each visitor's browser tells you whether the platform is responding *right now*, from where the user is. Fast to roll out, free, and survives any backend going down.

The tradeoff: opaque (`no-cors`) responses don't expose the HTTP status code, so a 500 looks identical to a 200. We mitigate by setting tight per-target timeouts (5s default, 8s for GitHub releases). A network failure or timeout flips the dot red; a successful resolution flips it green.

## Components and targets

| Component | Target |
|---|---|
| Mission Control API | `https://app.agentnetes.io/api/v1/healthz` |
| Auto-update feed | `https://github.com/Fiv21/fv-mission-control/releases/latest/download/latest-mac.yml` |
| Auth (Firebase) | `https://identitytoolkit.googleapis.com/v1/projects` |
| Sync (cloud-service) | `https://app.agentnetes.io/api/v1/healthz` |
| Marketplace | `https://app.agentnetes.io/marketplace/agents` |
| Stripe webhooks | `https://api.stripe.com/healthcheck` |

To add or change a target, edit the `COMPONENTS` array at the top of the `<script>` block in `index.html`. Push to `main`; GitHub Pages rebuilds in ~30s.

## Posting an incident

Edit `incidents.json`:

```json
[
  {
    "title": "Sync intermittent on EU clients",
    "severity": "SEV2",
    "message": "Investigating elevated 5xx on cloud-service from EU regions.",
    "created_at": "2026-05-02T14:23:00Z"
  }
]
```

Push. The page polls `incidents.json` every 60s.

## Why GitHub Pages and not Instatus

Instatus's free tier doesn't allow custom domains; their Hobby tier is $20+/mo. GitHub Pages on a public repo is free, has Let's Encrypt out of the box, and gives us full control over the design.

## Future automation

- Cloud Monitoring webhook → small handler that POSTs an updated `incidents.json` here when a 5xx-rate / p99 / uptime alert fires
- Per-component uptime stats sourced from GCP Monitoring metrics
