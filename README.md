# Signal — PR Intelligence Dashboard

A PR monitoring tool for YTL Creative Communications: collects news and social
coverage on YTL, its competitors, and its industries, scores each item for
PR relevance and "strength" (how good or bad it is), and surfaces it as a
ranked daily briefing instead of a firehose.

## What's in this repo

```
index.html                          The dashboard — open directly, or deploy as a static site
workflows/
  tracker-ingestion-scoring.json    n8n workflow: pulls news/Reddit/YouTube, scores via ILMU, writes to Notion
  ilmu-dashboard-bridge.json        n8n workflow: read-only bridge so the dashboard's assistant can query ILMU
docs/
  BUILD_GUIDE.md                    Full build reference — architecture, credentials, scoring logic, deployment
  PORTABLE_SPEC.md                  Standalone spec for handing this project to another AI/developer
```

## Current status

The dashboard runs today with **real live news** (GDELT + Hacker News,
fetched client-side, no key needed) mixed with **simulated social posts**
clearly labeled DEMO. Everything else — the n8n ingestion pipeline, Notion
storage, ILMU-powered scoring and Q&A — is built and ready to connect, but
needs credentials wired in before it's live. See `docs/BUILD_GUIDE.md`
section 13 for the exact real-vs-simulated breakdown.

## Setup

1. Import both files in `workflows/` into n8n (**Import from File**, not the
   API — n8n's update API can drop credentials from HTTP nodes).
2. In `ilmu-dashboard-bridge.json`, open the **Build Read-Only Prompt** node
   and set `SHARED_SECRET` to your own value — the placeholder
   `REPLACE_WITH_YOUR_OWN_SECRET` must not be used as-is.
3. In `tracker-ingestion-scoring.json`, confirm the ILMU and Notion
   credentials are attached (re-attach if the import dropped them).
4. Open `index.html` locally, or deploy it (Vercel recommended) and paste
   your production webhook URL + secret into the dashboard's Settings page.

Full details, including the three-stage test procedure and an error
decoder, are in `docs/BUILD_GUIDE.md`.

## Security note

Any secret entered into the dashboard's Settings page is visible to anyone
who opens that page — it's client-side JavaScript. This is acceptable for
local testing but **not for anything shared or hosted publicly** without
putting a proxy (e.g. an Azure Function) between the browser and the real
ILMU credential. See `docs/BUILD_GUIDE.md` section 8.
