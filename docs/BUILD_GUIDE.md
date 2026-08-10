# Signal — PR Intelligence System
## Complete Build & Operations Guide

**Project:** PR AI Scoring (YTL Creative Communications)
**Owner:** Bradley Tan
**Last updated:** 7 August 2026

---

## 1. What this system is

A PR intelligence platform that automatically collects, scores, and surfaces news and social media coverage about YTL Group, its competitors, and its industries — so the comms team stops manually scanning and starts working from a ranked, filtered feed.

**Three components:**

| Component | What it does | Status |
|---|---|---|
| **Ingestion engine** | n8n workflows pulling from news + social sources on a schedule | Built, needs credentials |
| **Storage + scoring** | Claude scores each item, results land in Notion | Notion DB built, scoring built |
| **Dashboard** | Web interface — Today / Browse / Us vs Them / Trends / Settings | Built as prototype |
| **Manual intake** | Paste-a-link tool for IG/FB coverage | Live at pr-ai-scoring.vercel.app |

---

## 2. Architecture

```
                    ┌─────────────────────────────┐
                    │   SCHEDULER (n8n, daily/15m) │
                    └──────────────┬───────────────┘
                                   │
       ┌───────────────┬───────────┼───────────┬────────────────┐
       ▼               ▼           ▼           ▼                ▼
  Google News     Reddit       YouTube      GDELT        Google Alerts
    (RSS)      (public JSON)  (Data API)  (free API)      (via email)
       │               │           │           │                │
       └───────────────┴───────────┴───────────┴────────────────┘
                                   │
                          ┌────────▼─────────┐
                          │  DEDUPLICATION   │  skip anything seen before
                          └────────┬─────────┘
                                   │
                          ┌────────▼─────────┐
                          │  RELEVANCE GATE  │  score 0-100, drop below threshold
                          └────────┬─────────┘
                                   │
                          ┌────────▼─────────┐
                          │  CLAUDE SCORING  │  PR score, sentiment, summary, cluster
                          └────────┬─────────┘
                                   │
                          ┌────────▼─────────┐
                          │  NOTION DATABASE │  permanent archive
                          └────────┬─────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
      DASHBOARD (web)      TEAMS ALERT (PA)      ILMU BRIDGE (n8n)
       polls every 15m      when score 80+        answers questions
```

**Manual path (for Instagram/Facebook):**
```
Someone spots a post → pastes link into vercel intake tool
  → classifies origin (Original / Amplifier / Reactive)
  → scored → same Notion database
```

---

## 3. Every credential you need

| # | Credential | Where to get it | Cost | Used by |
|---|---|---|---|---|
| 1 | **Anthropic API key** | console.anthropic.com | Pay per use (~$5-20/mo at this volume) | Scoring + clustering nodes |
| 2 | **Notion integration token** | notion.so/my-integrations | Free | Storage write + dashboard read |
| 3 | **YouTube Data API v3 key** | console.cloud.google.com | Free (10k units/day) | YouTube search node |
| 4 | **ILMU bearer credential** | Already exists — `CRRgScOmqPWvpnwG` | — | ILMU bridge |
| 5 | **Dashboard shared secret** | You invent it | Free | Bridge auth |
| 6 | **Awario account** *(optional)* | awario.com | $89-249/mo | X/IG/FB coverage |
| 7 | **Azure subscription** *(optional)* | portal.azure.com | Free tier sufficient | Hosting + real-time push |

**No key needed:** Google News RSS, Reddit public search, GDELT, Hacker News Algolia.

### Notion setup checklist
- Create the integration at notion.so/my-integrations
- Copy the **Internal Integration Token**
- Open the tracker database → `...` menu → **Connections** → add your integration
- **This last step is the one people forget** — without it the API returns 404 on a database you can plainly see

---

## 4. Data sources — honest capabilities

### What works free and automated

| Source | Auth | Coverage | Latency | Notes |
|---|---|---|---|---|
| **Google News RSS** | None | Global news, all major outlets | ~15 min | Server-side only — CORS-blocked in browsers |
| **GDELT** | None | Global news, 2017→present, full-text search | ~15 min | Works in browsers. Supports `startdatetime`/`enddatetime` for historical backfill |
| **Reddit** | None | All public subreddits, real upvote/comment counts | Real-time | `reddit.com/search.json?q=X&sort=new` |
| **YouTube Data API v3** | Free key | Video search, view/comment counts | Real-time | 10,000 units/day quota |
| **Hacker News (Algolia)** | None | Tech discussion, real points/comments | Real-time | Browser-friendly |
| **Google Alerts** | None | Anything Google indexes | ~hourly | Delivered by email — n8n parses the inbox |

### What does NOT work free — and why

| Platform | Why not | The only real options |
|---|---|---|
| **Instagram** | Graph API only reads accounts you own. No public hashtag/keyword search across the platform. Basic Display API deprecated. | Paid listening tool, or manual paste-link |
| **Facebook** | Same — Graph API is your own Pages only. CrowdTangle (the free research tool) was shut down in 2024; its replacement, Meta Content Library, is academic-access only. | Paid listening tool, or manual paste-link |
| **X / Twitter** | Free tier cannot search recent tweets by keyword. Basic tier starts ~$200/mo. | Paid listening tool, or X's own saved-search notifications (manual) |
| **LinkedIn** | No public API for keyword or trend search at all. | Manual only. Anything claiming otherwise is scraping — breaks constantly, risks the account |

> **Do not attempt to scrape Meta or LinkedIn.** It breaks on every layout change and can get accounts banned. The paste-link intake tool is the correct architectural answer for these platforms, not a compromise.

---

## 5. The scoring system

### Relevance score (the noise gate)

Runs *before* anything enters the feed. Four factors, 0-100:

| Factor | Weight | Logic |
|---|---|---|
| **Source authority** | 0-40 | Reuters/Bloomberg = 95, The Edge = 88, Bernama = 85, unknown blog = 45. Multiply by 0.4 |
| **Keyword match** | 0-30 | Does the text actually *name* the tracked entity? 15 points per matched term, capped |
| **Engagement** | 0-15 | `engagement / 30`, capped — so virality alone can't push junk through |
| **Verified news bonus** | 0-15 | Real fetched article beats simulated/social chatter |

**Default threshold: 45.** Below that, the item never enters the feed. In the dashboard this is the "Noise control" slider.

Plain-English bands shown to users:
- **75+** → "Must read"
- **55-74** → "Worth a look"
- **Below 55** → "Low priority"

### Source authority table

```
reuters.com          95    theedgemalaysia.com  88
bloomberg.com        95    bernama.com          85
ft.com               92    channelnewsasia.com  85
nikkei.com           90    businesstimes.com.sg 84
straitstimes.com     84    thestar.com.my       82
nst.com.my           80    malaymail.com        70
freemalaysiatoday    68    (unknown)            45
```

Extend this list as you find outlets that matter to YTL.

### PR score (Claude's judgment)

Separate from relevance. Claude reads each item and returns:
- `pr_score` (0-100) — PR relevance/impact for a corporate comms team
- `sentiment` — Positive / Neutral / Negative
- `summary` — 1-2 plain sentences

### Story clustering

The feature that turns noise into signal. Claude compares each new item's headline against recent items and flags matches, so the dashboard can show **"also covered by 8 outlets."**

One Reddit thread ≠ eight national outlets running the same story. Without clustering you can't tell them apart.

---

## 6. What to track — the YTL map

### YTL business segments and their competitors

| Segment | YTL entity | Main competitors |
|---|---|---|
| **Utilities & Power** | YTL Power International, Wessex Water (UK), PowerSeraya (SG), Ranhill Utilities | Tenaga Nasional (TNB), Malakoff, Sunway Energy |
| **Data Centre & AI** | YTL Green Data Center Park (Kulai), NVIDIA partnership, sovereign AI compute | TNB DC ventures, Bridge Data Centres, AirTrunk, Princeton Digital Group |
| **Cement & Building Materials** | Malayan Cement, YTL Cement | Hume Cement, Cement Industries of Malaysia, Tasek Corporation |
| **Construction** | YTL Construction, Express Rail Link | Gamuda, IJM Corporation, Sunway Construction, WCT Holdings |
| **Property** | YTL Land & Development | SP Setia, Sime Darby Property, IOI Properties, Mah Sing |
| **Hospitality** | YTL Hospitality REIT, YTL Hotels | Genting Malaysia, Shangri-La Malaysia, Sunway Hospitality |
| **Telco & Digital Banking** | YTL Communications (Yes 4G), Ryt Bank | Maxis, CelcomDigi, U Mobile, TM, GXBank, Boost Bank, AEON Bank |

**Context:** YTL Corp (Bursa: 4677, Tokyo: 1773), founded 1955 by Yeoh Tiong Lay. Combined market cap of listed entities ~RM73.4bn. Operations in ~11 countries. Utilities is the largest revenue segment.

### Suggested tracked topics (10-topic budget)

1. YTL Corporation
2. YTL Power International
3. YTL data centre / Kulai
4. Sovereign AI Malaysia
5. Malayan Cement
6. Yes 4G / Ryt Bank
7. Tenaga Nasional *(competitor)*
8. Gamuda *(competitor)*
9. Malaysia AI infrastructure *(industry)*
10. Malaysia data centre policy *(industry)*

### What a PR person should track beyond mentions

- **Share of voice** vs named competitors — raw counts mean nothing without a denominator
- **Spokesperson/executive mentions** — separate alert tier from general brand mentions
- **Message pull-through** — did the phrase your team pitched actually appear in coverage?
- **Journalist relationships** — who covers YTL repeatedly, and how they lean
- **Negative velocity, not volume** — a volume spike alert fires on good news too
- **Origin classification** — Original (agency-pitched) / Amplifier (derivative) / Reactive (responding to a moment)
- **Response time** — your team's own performance metric
- **Geographic split** — Malaysian press vs international wires need different responses

---

## 7. Notion database schema

**Database:** 🌐 AI News & Social Trends Tracker (under YTLCC Task Management Hub)
**Data source ID:** `b45e8d7e-2b7b-4660-8ab4-09aec625da34`

| Field | Type | Notes |
|---|---|---|
| Headline / Topic | Title | |
| Keyword Tracked | Multi-select | Which tracked topic matched |
| Type | Select | News / Social Post / Forum / Manual intake |
| Platform | Select | Google News / X / Instagram / Facebook / Reddit / YouTube / Other |
| Source / Author | Rich text | |
| Date Published | Date | |
| Summary | Rich text | Claude-generated |
| Sentiment | Select | Positive / Neutral / Negative |
| PR AI Score | Number | 0-100 |
| Engagement | Number | Likes/shares/upvotes where available |
| URL | URL | |
| Status | Select | New / Reviewed / Actioned |
| Date Logged | Created time | Auto |

**Fields to add for full functionality:**
- `Relevance Score` (Number) — the noise-gate score
- `Cluster ID` (Text) — groups items covering the same story
- `Origin Type` (Select) — Original / Amplifier / Reactive
- `Segment` (Select) — which YTL business area

---

## 8. The ILMU bridge

### Why a separate workflow

**Never point the dashboard at ILMU's main Teams webhook.** That endpoint runs the full action pipeline — intent classification, task creation, Notion writes. A dashboard question starting with "tell me about..." can trip the stakeholder-notify pattern and silently write to the Task Board. This has already happened once in production.

The bridge (`ILMU_Signal_Bridge_v2.json`, workflow ID `0yW5U2BdTtmtOsKI`) is read-only by construction: no Notion nodes, no action router, no write path.

### Setup

1. Import via **Import from File** — never API push (n8n's update API drops credentials from HTTP nodes)
2. **Ask ILMU node** — the payload uses an OpenAI-style `messages` array, so the endpoint is almost certainly `/v1/chat/completions`, not a bare `/v1`. Copy the exact URL from your live ILMU workflow's HTTP node.
3. **Model must be `ilmu-v3.1`** — *not* `nemo-super`, which is only a node display name. This exact mistake made v7.6 inert on first deploy.
4. Set `SHARED_SECRET` in the Build Read-Only Prompt node. Set `REQUIRE_AUTH = true`.
5. Verify the ILMU credential is still attached after import.
6. Activate.

### Testing (three stages)

**Stage 1 — inside n8n.** Click the webhook node → *Listen for test event*, then:

```bash
curl -X POST https://btym-wflow.shop/webhook-test/signal-dashboard \
  -H "Content-Type: application/json" \
  -H "x-agent-token: YOUR_SECRET" \
  -d '{"message":"Say hello","context":{"itemsToday":5,"topics":[],"topItems":[]}}'
```

Watch the canvas — each node shows its real input and output. Most problems surface here.

**Stage 2 — production URL.** Same command with `webhook` instead of `webhook-test`. Expect `{"botReply":"..."}`. Then re-run *without* the token header — you should get `Not authorised`. If you get a real answer, auth isn't enforcing.

**Stage 3 — dashboard.** Settings → paste URL + token → Connect → Test connection. This is the only stage that tests CORS.

### Error decoder

| Message | Cause |
|---|---|
| `ILMU call failed: HTTP 404` | Wrong endpoint URL — try `/v1/chat/completions` |
| `ILMU call failed: HTTP 401/403` | Credential missing or unauthorised |
| `unrecognised response shape` | Connected, but different JSON structure — the error lists actual keys |
| `Not authorised` despite sending token | Token mismatch — check for stray spaces |
| Works in curl, fails in dashboard | CORS |
| No response at all | Workflow inactive, or wrong path |

### Security reality

Anything in a client-side page is readable by anyone who opens it. The token stops strangers who find the URL; it does not stop anyone with page access. Fine for local testing and internal-only pages.

**For production:** put an Azure Function or a second n8n webhook in between, so the real secret never reaches the browser. Also tighten `Access-Control-Allow-Origin` from `*` to your actual site origin.

> **Outstanding:** the main ILMU workflow has a hardcoded plaintext secret (`ytlcc-agent-test-2026-long-secret`) flagged for rotation. Rotating it requires coordinating with the Power Automate side, which also calls it.

---

## 9. Historical archive strategy

### What's actually available

| Period | Source | Cost |
|---|---|---|
| **2017 → now** | GDELT full-text (supports date-range parameters) | Free |
| **~2000 → now** | Malaysian outlet archives (The Star, NST, The Edge, Bernama) — on-site search, no API | Free to browse |
| **1990s → now** | Factiva, LexisNexis, ProQuest | Enterprise (thousands/year) |
| **1955 → 1990s** | National Library Malaysia microfilm; YTL annual reports; Bursa filings | Manual research |

### The honest answer on 1955

No free or affordable digital news archive reaches 1955. For YTL's own history, **Bursa Malaysia's announcement archive and YTL's annual reports are better sources than news anyway** — authoritative and first-party rather than second-hand.

### The archive that actually matters

From day one of running the automation, every scored item is stored permanently. In six months you have six months of searchable, scored, sentiment-tagged YTL coverage scored against *your* criteria — something no paid tool offers.

**Two actions:**
1. **Never prune.** Storage is cheap; history is not recoverable.
2. **Backfill on day one.** Run a one-time GDELT pull with `startdatetime=20170101000000` on your key terms. Single n8n run, free, seeds years of history before you start.

---

## 10. Tool investment decision

### The deciding factor is API access, not features

You need data flowing into *your* database and dashboard, not sitting in someone else's UI.

| Plan | Monthly | Topics | API | Feeds your stack? |
|---|---|---|---|---|
| Awario Starter | $49 ($29 annual) | 3 | No | No — CSV export only |
| **Awario Pro** | **$149 ($89 annual)** | **15** | **No** | **Yes, indirectly — n8n parses its alert emails** |
| Awario Enterprise | $399 ($249 annual) | 100 | Yes | Yes, directly |
| BrandMentions | $79-99 entry | 5-50 | Enterprise only | Not affordably |

### Recommendation: Awario Pro (~$89/mo annual)

**Reasoning:** Awario's API is Enterprise-only at $249/mo — a steep jump. But Pro includes email and Slack alerts, and you already have the pattern for parsing alert emails in n8n (it's identical to the Google Alerts approach). Point Awario's alerts at a dedicated inbox, parse in n8n, and X/Instagram/Facebook data flows into your stack for a third of the price.

**Trade-offs to accept:** alerts are batched (near-real-time, not instant); HTML email parsing breaks when they change templates.

**Upgrade to Enterprise when:** email lag hurts crisis response, you outgrow 15 topics, or parsing fragility becomes someone else's maintenance problem.

**Before committing:** use the 7-day free trial specifically to measure actual latency on your keywords. Sources conflict — Awario claims non-stop monitoring on all plans, but at least one review reports slower refresh on lower tiers.

### Tools evaluated and rejected

- **Tavily** — a search API for AI agents, not a listening tool. No monitoring, no IG/FB, no trending concept. Useful *inside* n8n as a web-search step; not a competitor to the above.
- **Syften** ($19.95) — Reddit/forums focused. X is a paid add-on, no Meta coverage.
- **Xpoz** ($20) — every favourable review traced back to its own blog. Unverifiable claims; check independent reviews (G2, Capterra) before considering.
- **Brand24** ($199+), **Mention** ($599), **Meltwater/Brandwatch** ($800-3,000+) — overkill for this scope.

---

## 11. Deployment on your stack

| Layer | Tool | Notes |
|---|---|---|
| Ingestion + scoring | **n8n** (btym-wflow.shop) | Dev instance caps at 750 runs/month — move to Sandbox/Production before team rollout |
| Storage | **Notion** → **Azure Table Storage** | Migrate when Notion rate limits bite |
| Real-time push | **Azure SignalR** (free tier: 20 connections) | Replaces 15-min polling with instant updates |
| Hosting | **Azure Static Web Apps** (free tier) | Keeps everything in one tenant vs Vercel |
| Alerts | **Power Automate → Teams** | Reuse the ILMU relay pattern |
| Email digest | **Power Automate → Outlook** | PA is on a 3-month Premium trial |

**Division of labour:** n8n gathers and scores → Azure stores and pushes → PA notifies in Teams.

### Known n8n gotchas (learned the hard way)

- `update_workflow` API drops credentials from HTTP nodes — always **Import from File**
- Disabling an IF node does **not** skip its branch — it blind-forwards to the first output
- Pasting into an open canvas duplicates rather than replaces — wipe first or import to a blank workflow
- n8n places pasted content relative to viewport zoom — reset zoom before importing
- After removing any node, run a **reachability check** on the whole graph. Orphaned subgraphs are silent and produce confidently wrong "nothing found" answers

---

## 12. Build phases

### Phase 1 — Make ingestion real *(highest value, do first)*
- [ ] Add Anthropic, Notion, YouTube credentials to the n8n tracker workflow
- [ ] Add the extra Notion fields (Relevance Score, Cluster ID, Origin Type, Segment)
- [ ] Test end-to-end: one keyword → scored row appears in Notion
- [ ] Run the GDELT historical backfill to 2017
- [ ] Set the schedule (daily to start, tighten to 15-min once stable)

### Phase 2 — Connect the dashboard to real data
- [ ] Replace the dashboard's in-memory array with Notion API reads
- [ ] Point search at the database, not the session
- [ ] Verify the ILMU bridge end-to-end (three-stage test above)

### Phase 3 — Add clustering
- [ ] Claude compares new headlines against the last 48h of items
- [ ] Write Cluster ID back to Notion
- [ ] Dashboard "Also covered by" reads real cluster members

### Phase 4 — Alerts and distribution
- [ ] PA flow: Teams message when PR score 80+
- [ ] PA flow: negative-velocity alert (not raw volume)
- [ ] Daily digest email from flagged items

### Phase 5 — Paid social *(when budget approved)*
- [ ] Awario Pro trial — measure real latency first
- [ ] Configure the 10 topics + competitor keywords
- [ ] Route alerts to a dedicated inbox
- [ ] n8n parses that inbox into the same pipeline

### Phase 6 — Production hardening
- [ ] Move off the n8n dev instance
- [ ] Proxy the ILMU call (Azure Function) so no secret reaches the browser
- [ ] Tighten CORS to the real origin
- [ ] Rotate the main ILMU shared secret (needs IT + PA coordination)
- [ ] Migrate storage to Azure if Notion limits bite

---

## 13. Current status — real vs simulated

**Be precise about this when demoing.** Credibility depends on not overclaiming.

| Element | Status |
|---|---|
| News articles (GDELT + Hacker News) | **Real, live**, fetched every 90 seconds, clickable to source |
| Google News RSS ingestion | Real — but server-side only (CORS-blocked in browser; lives in n8n) |
| Reddit / YouTube ingestion | Real capability, built in the n8n workflow, needs credentials |
| X / Instagram / Facebook posts | **Simulated** — labelled DEMO in the UI |
| Relevance scoring + noise gate | Real logic, running on both real and simulated items |
| PR score / sentiment / summary | Real logic, needs the Anthropic key wired |
| Story clustering | **Not yet built** — dashboard uses same-topic grouping as a stand-in |
| AI Trends page data (launches, funding, regulatory) | **Static mock rows** — needs real sourcing |
| IG/FB trends data (hashtags, formats, velocity) | **Static mock rows** — needs a paid tool |
| Competitor volumes | **Static** — each rival needs to become a tracked keyword |
| Assistant | Real intent parsing over live feed data; ILMU bridge built, untested end-to-end |

---

## 14. Known limitations

1. **"Real-time" means 15-minute polling**, not instant push. True streaming needs paid APIs. This matches what commercial monitoring tools actually deliver — nobody pays for millisecond latency on PR monitoring.
2. **Meta and X cannot be automated free.** This is a platform policy constraint, not a technical one. No amount of engineering solves it.
3. **The dashboard's assistant is pattern-matching**, not reasoning — until the ILMU bridge is live.
4. **Client-side secrets are visible.** Any token in the page is readable. Proxy before sharing.
5. **The n8n dev instance caps at 750 runs/month.** A 15-minute schedule alone is ~2,880 runs/month. Move to Sandbox/Production before tightening the schedule.
6. **Sentiment analysis is imperfect** on Malaysian English and Bahasa content. Spot-check before trusting aggregate sentiment in a report.

---

## 15. Files in this project

| File | What it is |
|---|---|
| `signal_final.html` | The dashboard — 5 sections, live news, search, assistant, ILMU settings |
| `ai_news_social_trends_tracker.json` | n8n ingestion + scoring workflow |
| `ILMU_Signal_Bridge_v2.json` | Read-only n8n bridge for assistant answers |
| `pr_ai_scoring_build_spec.md` | Portable spec for handing to another AI/developer |
| pr-ai-scoring.vercel.app | Manual paste-link coverage intake (Bradley's build) |

---

## 16. The one-paragraph version (for your boss)

> Signal collects everything published about YTL and its competitors — from news wires, Reddit, YouTube and (with a paid tool) Instagram, Facebook and X — scores each item for how much it matters to us, and presents it as a ranked morning brief instead of a firehose. It runs automatically on our existing n8n and Azure infrastructure, stores everything permanently so we build our own searchable archive, and answers plain-English questions through ILMU. The free sources are live today; adding Instagram, Facebook and X coverage costs roughly USD 89/month.
