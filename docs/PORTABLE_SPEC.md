# PR AI Scoring — Build Spec

Build a PR intelligence platform for a corporate comms/PR team (YTLCC) to track industry news and social media chatter — starting with "Sovereign AI" as the flagship keyword — in one place, scored for PR relevance, without manual scanning.

## Core concept
Two intake paths feeding one scored database:
1. **Automated daily feed** — pulls news + social posts on tracked keywords automatically.
2. **Manual coverage intake** — user pastes a specific article/social link (especially for Instagram/Facebook, which have no open API for automated monitoring) and classifies how it originated: **Original** (agency-pitched), **Amplifier** (derivative of another story), or **Reactive** (responding to a news moment).

Every item — from either path — gets AI-scored and stored the same way.

## Data sources (automated path)
- **Google News RSS** — free, no auth, keyword search per tracked term.
- **Reddit public search JSON API** — free, no auth needed for read-only search.
- **YouTube Data API v3** — free tier (10k units/day), needs an API key.
- Do NOT attempt automated Instagram/Facebook scraping — Meta's APIs only expose accounts you own, not public search. Route IG/FB through the manual paste-a-link intake instead.

## Scoring
For every item (from either intake path), have an LLM (Claude or GPT) return:
- `pr_score` (0–100): PR relevance/impact, not just importance
- `sentiment`: Positive / Neutral / Negative
- `summary`: 1-2 plain sentences
- For manual intake: also classify origin type (Original/Amplifier/Reactive)

## Clustering (important, don't skip)
When the same real-world story is covered by multiple outlets, group them. Have the LLM compare a new item's headline/summary against recent items and flag matches. The UI should show "also covered by N other sources" — this is what turns noise into signal (8 outlets on one story vs. 1 lone Reddit post).

## Storage
A structured database (Notion, Airtable, or Postgres) with fields: Headline, Keyword/Tag, Type (News/Social/Manual), Platform, Source, Date Published, Summary, Sentiment, PR Score, Engagement, URL, Origin Type (for manual intake), Status, Date Logged, Cluster ID.

## Frontend — four views, light mode, built for non-technical users
1. **Today / Briefing** — single top story (highest score + most outlets), then a ranked list of other story clusters.
2. **Browse / Feed** — every individual item, filterable by keyword.
3. **Trending** — social items ranked by engagement velocity (rate of change), not raw volume.
4. **Saved** — user-flagged items compiled into a shareable daily digest.

Plus a **story detail drawer**: click any item → see full summary, score breakdown, and "Also covered by" list of every other outlet on the same story with their individual score/sentiment.

Design constraints: plain language over jargon (e.g. "Newsworthy" not "PR Score"), generous whitespace, light color palette, minimal technical vocabulary, mobile-friendly for on-the-go checking.

## Refresh cadence
Be honest about "real-time": true instant push needs paid streaming APIs. Realistic target is polling every 15–30 min for news, 5–10 min for social — same-hour freshness, which matches how real PR monitoring tools actually operate.

## Optional: conversational layer
An "ask about today's coverage" chat interface that answers questions from the live scored data (e.g. "any negative coverage today?", "what's trending on sovereign AI?").

## What NOT to do
- Don't attempt to auto-scrape Instagram/Facebook — it breaks constantly and risks account bans.
- Don't over-promise millisecond real-time — set expectations at "checked every X minutes."
- Don't score every item the same way regardless of source — a Reddit thread and a Reuters article need different weighting logic for engagement vs. reach.
