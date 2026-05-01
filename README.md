# Agentnetes Status Page

Live health for the Agentnetes platform at <https://status.agentnetes.io>.

A server-side prober runs once per minute against the 6 component endpoints and writes a snapshot to a public GCS bucket; the page polls the snapshot every 10s and renders it. No per-visitor amplification on the production endpoints during an incident.

## Architecture

```
Cloud Scheduler (every 60s)
  └─ Cloud Function "status-probe" (Node 22, 256MB, max-instances 1)
       └─ HTTP-probes 6 endpoints in parallel
            └─ writes JSON to gs://agentnetes-status-public/status.json (public-read, 30s cache)

Browser at status.agentnetes.io
  └─ polls https://storage.googleapis.com/agentnetes-status-public/status.json (every 10s)
       └─ renders the snapshot
```

| File | Purpose |
|---|---|
| `index.html` | Page itself; vanilla HTML/CSS/JS, no build step |
| `incidents.json` | Manually-curated incident list, polled every 60s |
| `CNAME` | Binds GitHub Pages to `status.agentnetes.io` |

The probe function source lives at `<repo>/cmd/status-probe/` of the main repo (deployed via `gcloud functions deploy status-probe`).

## Why server-side probes

A page that probes from the browser amplifies load N× during an actual outage (every concerned visitor pings the failing API). Server-side probing keeps load constant: 6 probes per minute regardless of visitor count, and the snapshot is served from Google's CDN.

The probe also reads real HTTP status codes (no CORS constraint), so we can tell a 500 from a 200 — something browser-side `mode: "no-cors"` can't do.

## Refresh granularity

- **Probe frequency:** every 60s (Cloud Scheduler's minimum). Free tier covers it.
- **Snapshot upload:** within ~2s of each probe.
- **Page poll:** every 10s. Snapshot CDN cache is 30s, so the page may show data up to 40s old at any instant.

For sub-1s freshness you'd need a long-running container. Industry norm is 60s; Statuspage's free tier is also 60s.

## Posting an incident

Edit `incidents.json` in this repo:

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

## Manually triggering a probe

Cloud Scheduler runs the probe automatically. To force-refresh between schedule ticks:

```bash
curl https://us-central1-a8s-mission-control.cloudfunctions.net/status-probe
```

The function is `--allow-unauthenticated` because the GCS write is its only effect and the response is non-sensitive (`{ ok, overall, took_ms }`).

## Why not Instatus

Free tier doesn't allow custom domains; Hobby is $20/mo. This setup is $0.
