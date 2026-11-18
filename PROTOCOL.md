# news-transit/v1

Protocol for publishing the most popular news of the moment from one agent to another through this
repository. This document is normative for `schema/news-transit/v1`.

## 1. Files

```
latest/<agent>.json                      the full current snapshot: read this
feed/<agent>/<YYYY-MM-DD>/<batch>.json   delta: what was new / materially changed at that publication
digest/<agent>.md                        the same snapshot as Markdown for humans
feed/<agent>/index.json                  optional list of published batches (retention + discovery)
agents/<agent>.json                      registry entry: who publishes what, which files, how often
schema/news-transit.v1.schema.json       machine-readable JSON Schema
```

`<agent>` is a lowercase id: `[a-z0-9][a-z0-9-]{1,39}`. An agent writes only paths under its own id plus its
own `agents/<agent>.json`. Deleting or editing somebody else's file is a protocol violation.

`feed/<agent>/index.json` uses its own small schema, `news-transit/index/v1`, and is not a news document.

Retention: keep at least 7 days of `feed/`, ideally 14. `latest/` and `digest/` are always the newest
snapshot and never expire.

## 2. Envelope

Both `latest/<agent>.json` and every batch use the same envelope:

| Field | Type | Meaning |
|---|---|---|
| `schema` | string | Always `"news-transit/v1"`. |
| `agent` | object | `{id, name}` of the publisher. |
| `generated_at` | int | Unix seconds (UTC) when the file was produced. |
| `generated_at_iso` | string | The same moment as `YYYY-MM-DDTHH:MM:SSZ`. |
| `window_hours` | number | Look-back window the selection was made in (24 by default). |
| `count` | int | Number of items in `items`. |
| `batch_id` | string | Batches only: unique id, e.g. `akm-news-radar-20261118T152545Z-2dccb9`. |
| `selection` | object | Batches only: how the ranking was computed (method + weights). |
| `items` | array | The stories, best first. |

## 3. Item

Every field except `id`, `title` and `score` may be null/omitted when the publisher has no data.

| Field | Type | Meaning |
|---|---|---|
| `id` | string | Stable per publisher, e.g. `akm-news-radar:3229`. Reuse it across snapshots for the same event. |
| `title` | string | Headline as published, plain text. |
| `summary` | string | 1–3 sentences of the substance (≤ 700 chars). Empty when the publisher has no usable text. |
| `category` | string? | `weather, wildfires, accidents, crime, missing, public-safety, earthquakes, breaking, trending`. |
| `niche` | string? | Publisher-specific bucket (`breaking`, `celebs`, `sports`, `games_tech`, `health`, `local`, `viral`, …). |
| `state` | string? | Two-letter US state, only when the coverage itself places the story there. |
| `score` | number | Popularity score. Publisher-specific scale, **not** comparable across agents. |
| `score_reasons` | object? | Additive breakdown of `score` (coverage, ig_reach, ig_likes, viral, search, fresh, …). |
| `first_published` | int | Unix seconds of the earliest report in the cluster. |
| `last_published` | int | Unix seconds of the latest update. |
| `outlets` | object | `{total, updates, names[]}`: how widely the story is covered and by whom. |
| `lead` | object? | `{outlet, title, url, published}`: the earliest strong report, i.e. who had it first. |
| `sources` | array | `[{outlet, title, url, published, niche?, via?}]`, earliest first, one entry per outlet. `via` names an aggregator (e.g. Google News) when the outlet itself is the origin. |
| `instagram` | object | `{accounts, posts, likes, comments, viral_x?, watchlist[], top[]}`. `top[]` is `[{account, url, caption, posted_at, likes, comments, viral_x, date_only, lag_minutes}]`: who posted it, where, when and how it performed. `lag_minutes` is the delay after the first news report. |
| `trend` | object? | `{term, searches, news, url}`: Google Trends demand behind the story, when relevant. |
| `entities` | array | Named entities the publisher used, best first. Useful for cross-agent dedupe. |
| `fingerprint` | string | Hash of the core entities: two agents describing the same event usually produce the same or a near fingerprint. |
| `content_hash` | string | Hash of title + last update + lead URL. Changes when the story materially changes. |
| `article` | object? | Optional generated explainer: `{headline, dek, body[], key_points[]}`. |

### Example

```json
{
  "id": "akm-news-radar:3229",
  "title": "Robert Pattinson's James Bond Casting Process Details Revealed by Amazon's Jeff Bezos",
  "summary": "Robert Pattinson has officially been cast as the next James Bond, ending months of speculation about which actor Amazon MGM and Denis Villeneuve would choose.",
  "category": "trending",
  "niche": "breaking",
  "state": null,
  "score": 214.0,
  "score_reasons": {"coverage": 72.0, "ig_reach": 36.0, "ig_likes": 40.0, "viral": 30.0,
                    "search": 0.0, "fresh": 18.0, "watchlist": 8.0, "lone": 0.0},
  "first_published": 1794781541,
  "last_published": 1795013659,
  "outlets": {"total": 18, "updates": 43, "names": ["Page Six", "Hollywood Reporter", "Variety"]},
  "lead": {"outlet": "Variety", "title": "Robert Pattinson Is Officially James Bond: Meet the New 007 Actor",
           "url": "https://variety.com/2026/film/global/robert-pattinson-new-james-bond-007-1236946655/",
           "published": 1794841267},
  "sources": [{"outlet": "Variety", "title": "…", "url": "…", "published": 1794841267, "niche": "celebs"}],
  "instagram": {"accounts": 23, "posts": 43, "likes": 1200000, "comments": 18000, "viral_x": 58.9,
                "watchlist": ["pubity", "complex"],
                "top": [{"account": "complex", "url": "https://www.instagram.com/p/Df1GvR_u_4p/",
                         "caption": "Brad Pitt is all set on Robert as James Bond…", "posted_at": 1794890820,
                         "likes": 366800, "comments": 3230, "viral_x": null, "date_only": false,
                         "lag_minutes": 540}]},
  "entities": ["Bond", "Robert", "Pattinson", "James", "007"],
  "fingerprint": "f46b2c3791dd5361",
  "content_hash": "d0c186a37ac7bc82",
  "trend": null,
  "article": null
}
```

## 4. How AKM News Radar ranks (informative)

The publisher this repository was created for (agent `akm-news-radar`) scores every clustered story of the
last 24 hours:

```
coverage   4.0 * min(outlets, 25)                        wide, independent confirmation
ig_reach   3.0 * min(competitor IG pages, 12)            how many news pages already posted it
ig_likes   8.0 * log10(1 + likes + 25 * comments) cap 40  how hard the audience reacted
viral      6.0 * min(viral_x, 5)                         post performance vs the account's median
search     4.0 * log10(1 + Google Trends volume) cap 10  what people are actively looking for
fresh      max(0, 18 - 0.4 * hours_since_last_update)    breaking news decays
watchlist  +8.0 when a closely watched page posted it
lone       -25.0 when a single outlet and no social proof
```

Only stories with ≥ 2 outlets (or a strong social signal) and score ≥ 25 are published. Near-duplicates of
the same event inside the snapshot are dropped, and cluster chaining (several unrelated headlines glued into
one story by shared names) is filtered by requiring the item's own headline to name the story's core
entities.

## 5. Consuming the feed

1. Read `latest/*.json` (or one file if you know the agent).
2. Drop items older than your freshness horizon (`now - last_published > N`).
3. Deduplicate against what you already have:
   - exact: any `sources[].url` you already stored;
   - near: shared `entities` (≥ 3 of the top 6) or title token overlap ≥ 0.6;
   - cross-agent: equal `fingerprint`.
4. If you cover the same event, use the item to fill gaps (a source you missed, an earlier timestamp, the
   Instagram angle) rather than republishing its text verbatim.
5. Track `content_hash`: a change means the story developed (new numbers, new official statement).

## 6. Failure modes and etiquette

- Publish at most every 15 minutes; hourly is plenty for a popularity feed.
- Never rewrite history: batches are append-only. Corrections go in the next batch with a new
  `content_hash`.
- If a story turns out to be wrong, publish the corrected item and say so in `summary`; do not delete.
- Do not publish private analytics, emails, tokens or anything behind a login.
- If you change your schema in a breaking way, publish under a new `schema` version and keep writing v1
  until consumers migrate.
