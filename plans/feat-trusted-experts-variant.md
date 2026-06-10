# feat: `/experts` — Trusted-Sources Expert Opinion Variant

## Overview

A personal variant of last30days that answers one question: **"What do the highly
qualified people I already trust think about topic X?"** — for use when writing a
piece. Instead of fanning out to Reddit/X/YouTube/TikTok, it searches only the
user's curated trust graph:

1. **Raindrop bookmarks** — things the user explicitly saved
2. **Gmail newsletters** — subscribed authors, under the `news feed` label
3. **RSS subscriptions** — managed in Newsroom
4. **Engineer blogs list** — hand-picked high-value blogs kept in a repository

Output is a **writer's brief**: per-expert stances with verbatim quotes,
consensus vs. disagreement mapping, and pull-quote candidates with attribution.

**Key principle inversion vs. last30days**: last30days surfaces *what the crowd
says right now* (engagement-weighted, 30-day window). This variant surfaces
*what trusted individuals think* (trust-weighted, relevance-first, no hard
recency cutoff — a 2-year-old bookmarked essay by a great engineer beats
yesterday's mediocre newsletter).

## Architecture: Hybrid (query-time + personal index)

The four sources split cleanly by whether they are searchable:

| Source | Server-side search? | Strategy |
|---|---|---|
| Raindrop | ✅ Full-text search API (incl. highlights + cached copies) | Query-time API call |
| Gmail (`news feed` label) | ✅ Gmail query syntax | Delegate to the user's existing **gws Gmail skill** at the SKILL.md layer — do NOT reimplement OAuth in Python |
| Newsroom RSS feeds | ❌ | Local **corpus index** (SQLite FTS5), incrementally refreshed |
| Blogs repository | ❌ | Same corpus index; RSS autodiscovery per blog |

```
┌────────────────────────────────────────────────────────────────┐
│ SKILL.md orchestration (Claude)                                │
│  1. Parse topic            3. In parallel: gws Gmail search    │
│  2. Run experts.py ────────┐  (label:"news feed" + topic)      │
│  4. Synthesize brief ◄─────┴── merge both result sets          │
└────────────────────────────────────────────────────────────────┘
          │
┌─────────▼──────────── scripts/experts.py ──────────────────────┐
│  refresh (if stale) ──► lib/feeds.py ──► corpus.db (FTS5)      │
│                          ▲                                     │
│         lib/feed_sources.py (Newsroom OPML + blogs repo)       │
│                                                                │
│  query ──┬─► lib/raindrop.py (API search)                      │
│          └─► lib/corpus.py (FTS5 BM25 over RSS + blogs)        │
│  rank ───► trust tier + relevance (+ mild recency tiebreak)    │
│  enrich ─► lib/fulltext.py (fetch top-N articles, extract      │
│            quotable passages + author attribution)             │
│  emit ───► compact markdown for Claude synthesis               │
└────────────────────────────────────────────────────────────────┘
```

### Why hybrid, not pure query-time

RSS feeds only expose the last ~10–50 entries; a query-time fetch would miss
everything older and re-download every feed on every question. An incremental
index accumulates the full archive over time and makes queries instant.
`scripts/store.py` already implements the exact pattern needed (SQLite, WAL,
FTS5 porter+unicode61 tokenizer, BM25 ranking) — `lib/corpus.py` is a
straightforward adaptation of it.

### Why Gmail stays at the SKILL.md layer

The user already has a working local Gmail skill (`gws`). Gmail OAuth in Python
means credential files, token refresh, and a Google Cloud project — all
duplicated effort for a worse experience. Instead, SKILL.md instructs Claude to
run the gws query in parallel with `experts.py` and merge results during
synthesis. (Fallback noted in Open Questions if gws output proves too lossy.)

## Repo layout

```
variants/trusted/
  SKILL.md                  # deployed as ~/.claude/skills/experts/
  references/
    synthesis.md            # writer's-brief synthesis instructions
scripts/
  experts.py                # new orchestrator (thin; reuses lib/)
  lib/
    raindrop.py             # NEW — Raindrop REST API search
    feeds.py                # NEW — RSS/Atom fetch + parse (stdlib only)
    feed_sources.py         # NEW — load Newsroom OPML + blogs repo, autodiscover feeds
    corpus.py               # NEW — FTS5 personal index (adapted from store.py)
    fulltext.py             # NEW — article text extraction for quote mining
    # reused as-is: http.py, relevance.py, dedupe.py, dates.py, env.py,
    #               cache.py, ui.py, render.py patterns
```

Follows the existing rules: `lib/__init__.py` stays a bare marker; deploy via
`scripts/sync.sh` (extend it to also deploy the `experts` skill dir).

## Source modules

### 1. `lib/raindrop.py`

- Endpoint: `GET https://api.raindrop.io/rest/v1/raindrops/0?search={topic}`
  (collection `0` = all bookmarks). Auth: `RAINDROP_TOKEN` test token from
  app.raindrop.io/settings/integrations — single env var, no OAuth dance.
- Pull: title, excerpt, link, created date, tags, domain, and **highlights**
  (passages the user marked — these are gold: pre-curated quotes).
- Paginate up to ~100 results; normalize to the common item schema with
  `source: "raindrop"`.

### 2. `lib/feed_sources.py` + `lib/feeds.py`

- **Newsroom**: read the subscribed feed list from Newsroom's data. Config key
  `NEWSROOM_FEEDS` points at either an OPML export or Newsroom's own
  feed-list file (path confirmed during setup — see Open Questions). Parser
  accepts OPML, JSON, or plain URL list, so whichever format exists works.
- **Blogs repo**: config key `BLOGS_REPO` = local path (or git URL, cloned to
  `~/.cache/experts/blogs`). Tolerant parser: scan `.md`/`.txt`/`.opml`/`.json`
  files and extract URLs. For each blog URL without an explicit feed, do RSS/Atom
  **autodiscovery** (`<link rel="alternate" type="application/rss+xml">`),
  caching resolved feed URLs in the corpus DB so discovery runs once per blog.
- `feeds.py` fetches feeds concurrently (ThreadPoolExecutor, same pattern as
  `last30days.py`), parses RSS 2.0 + Atom with stdlib `xml.etree` (no new deps),
  extracts: title, link, author, published date, summary/content.

### 3. `lib/corpus.py` — the personal index

Adapted from `store.py`:

```sql
sources(id, kind, feed_url, site_url, title, author_default, trust_tier, last_fetched)
entries(id, source_id, url, title, author, published, summary, content, fetched_fulltext)
entries_fts USING fts5(title, author, summary, content, tokenize='porter unicode61')
```

- `experts.py refresh` ingests new entries (dedupe on URL); `--refresh` forces
  it, otherwise auto-refresh when `last_fetched` > 24h old. Refresh of ~100
  feeds takes well under a minute with 8 workers.
- Query: FTS5 BM25 with the topic, plus `relevance.token_overlap_relevance`
  re-scoring (already in lib) to filter incidental matches.

### 4. `lib/fulltext.py` — quote mining

RSS summaries are often truncated; quotes must be verbatim. For the **top N
(default 8–10) ranked items** that lack full content:

- Fetch the page via existing `http.py` (+ `cache.py`).
- Readability-lite extraction in stdlib: strip nav/script/style, prefer
  `<article>`/largest text block, decode entities — same approach as
  `hackernews._strip_html` but page-scale. No bs4/trafilatura dependency
  (consistent with the repo's stdlib-only stance).
- Extract **author** (meta tags `author`/`og:article:author`, byline patterns,
  fallback to the source's default author from the blogs list).
- Emit 2–4 "quotable excerpts": paragraphs with highest topic-term density,
  trimmed to quote length — the same role transcript highlights play for
  YouTube in last30days.

## Ranking: trust replaces engagement

No upvotes/likes exist here, and they'd be the wrong signal anyway. Score:

```
score = 0.55*relevance + 0.30*trust + 0.15*recency
```

| Trust tier | Sources | trust value |
|---|---|---|
| T1 — hand-picked individuals | Blogs repo, Raindrop items with user **highlights** | 100 |
| T2 — explicit curation | Other Raindrop bookmarks, Gmail newsletters | 80 |
| T3 — subscriptions | Newsroom RSS entries | 65 |

- **Recency is a tiebreaker, not a filter**: default lookback is *unbounded*
  for Raindrop and the corpus; `--days=N` available when the user wants only
  fresh takes (e.g., reacting to news). Recency component uses gentle decay
  (half-life ~1 year), so evergreen essays survive.
- Per-author cap (max 3 items per author) so one prolific blogger doesn't
  drown out the rest; dedupe via existing `dedupe.py` (same article bookmarked
  AND in RSS collapses, keeping the higher trust tier).

## SKILL.md flow (`/experts <topic>`)

1. **Parse intent**: topic + optional angle (`--days=N`, `--for="the piece I'm
   writing"` context blurb that sharpens synthesis).
2. **Run** `python3 scripts/experts.py "{topic}" --emit=compact` (foreground,
   5-min timeout; auto-refreshes index if stale).
3. **In parallel**, query Gmail via the gws skill:
   `label:"news feed" {topic}` (plus 1–2 reformulations), pull matching
   newsletter bodies, note sender → author mapping.
4. **Synthesize** per `references/synthesis.md`:

```
# What your trusted sources think about {TOPIC}

## The short version
[2-3 sentences: where the experts land, and the main fault line]

## Consensus
- [Point] — held by {Author A}, {Author B} (with one quote)

## Where they disagree
**{Author A}** ({blog/newsletter}, {date}): "verbatim quote…" — [1-line stance]
**{Author B}** …

## Voices, one by one
### {Author} — {source name} ({date})
> "Direct quote…"
[1-2 sentences of their argument; link]

## Pull-quotes for your piece
1. "…" — {Author}, {source}, {date} [link]

## Gaps
[Angles none of your sources cover — flag before you write]

---
📚 Sources: {N} Raindrop bookmarks │ {N} newsletters │ {N} RSS posts │ {N} blog posts
🗣️ Voices: {Author1}, {Author2}, …
```

   Synthesis rules (the variant's equivalent of last30days' citation rules):
   - **Attribute to people, never platforms** — "per Dan Luu", not "per RSS".
   - **Every claim about an expert's view needs a verbatim quote or a link.**
   - Quotes must come from the emitted excerpts — never paraphrase-as-quote.
   - Flag when a quote is old ("(2023 — pre-dates X)") rather than hiding dates.
5. Save brief to `~/Documents/ExpertBriefs/{date}-{slug}.md`; stay in expert
   mode for follow-ups (drill into one author, find the counter-argument, etc.).

## Config — `~/.config/experts/.env`

```
RAINDROP_TOKEN=xxx            # required for bookmarks
NEWSROOM_FEEDS=~/path/to/feeds.opml   # OPML / JSON / URL list
BLOGS_REPO=~/code/engineering-blogs   # local path or git URL
GMAIL_LABEL=news feed         # used by SKILL.md when calling gws
EXPERTS_DB=~/.local/share/experts/corpus.db   # default, optional override
SETUP_COMPLETE=true
```

First-run wizard (much lighter than last30days'): check for the file, ask for
the Raindrop token + the two paths, run an initial `refresh`, report corpus
size ("indexed 1,240 posts from 87 feeds"), then run the first query.
`experts.py --diagnose` reports per-source availability and index freshness.

## Implementation phases

**Phase 1 — Raindrop end-to-end (the thin slice)**
Scaffold `variants/trusted/`, `experts.py`, config loading, `raindrop.py`,
compact emitter, minimal SKILL.md. Value on day one: search bookmarks + Gmail
(via gws) and synthesize. ~Half the plumbing, smallest risk.

**Phase 2 — Corpus index for RSS + blogs**
`corpus.py` (adapt store.py), `feed_sources.py`, `feeds.py`, refresh command,
staleness auto-refresh, blogs-repo parsing + feed autodiscovery.

**Phase 3 — Quote mining + ranking**
`fulltext.py`, author extraction, quotable-excerpt selection, trust-tier
scoring, per-author caps, dedupe across sources.

**Phase 4 — Synthesis polish + deploy**
`references/synthesis.md` quality pass against 3–5 real topics the user would
actually write about; extend `sync.sh` to deploy the `experts` skill; README
section. Optional: a `/loop`-driven or cron `experts.py refresh` so the index
is always warm.

**Tests** (mirroring `tests/` conventions): fixture-based parsers (OPML, RSS,
Atom, blogs-repo markdown), Raindrop response normalization, trust scoring
order, dedupe-across-sources, FTS round-trip.

## Open questions / assumptions

1. **Newsroom feed list location** — assumed it can export OPML or has a
   readable feed-list file. Needs one path confirmed at setup; the parser
   accepts OPML/JSON/plain-URLs so the format is low-risk.
2. **Blogs repo format** — assumed a markdown/text list of URLs; tolerant
   parser covers other shapes. If it's someone else's repo (e.g., a fork of
   `kilimchoi/engineering-blogs`), the same parser works.
3. **Gmail depth** — gws-skill delegation is the plan; if gws can't return
   full newsletter bodies for quoting, fallback is a small `lib/gmail_local.py`
   reading via the Gmail API with the same credentials gws already provisioned.
4. **Skill name** — proposed `/experts`; alternatives: `/trusted`, `/voices`.
5. **Index growth** — RSS only yields entries going forward from first refresh;
   Raindrop + blogs-repo full archives cover the back-catalog. If deeper blog
   history matters later, a one-time sitemap backfill can be added (out of
   scope for v1).
