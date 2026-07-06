# SAS Competitive Intelligence — Battle Cards

A static, Vercel-deployable dashboard of competitor battle cards, structured to mirror
the retired Klue export. The dashboard is two files: a shell that never needs to
change, and a data file you edit whenever a competitor needs updating.

## Files

```
index.html    the dashboard shell — layout, styles, rendering logic. You shouldn't
              need to touch this unless you want to change the design.
data.json     everything else — every competitor, every section, all content.
              This is the file you edit to update or add competitors.
```

`index.html` fetches `data.json` at page load (`fetch('./data.json')`) and renders
whatever it finds. There's no build step, no framework, no database — just two
static files.

## Deploying to Vercel

1. Put `index.html` and `data.json` in the same folder at the root of your repo
   (or in `/public`, depending on how you set up the Vercel project).
2. Import the repo into Vercel as a static project — no framework preset, no
   build command needed.
3. Deploy. Every time you push a change to `data.json`, Vercel redeploys and the
   dashboard reflects it.

**One constraint to know:** `fetch('./data.json')` requires being served over
http(s). Opening `index.html` directly from disk (`file://...`) will fail with a
CORS-style error. To test locally, run any static file server from the folder
(e.g. `npx serve` or `python3 -m http.server`) and open the localhost URL —
don't just double-click the file.

## How to update an existing competitor

Open `data.json`, find the competitor by `id`, edit whatever fields need
changing, save, and redeploy. There's no schema migration or code change
required — the shell just renders whatever is in the file.

Update `lastUpdated` and add an entry to `changelog` when you make a
substantive change, so anyone opening the card can see what changed and when
without comparing old versions:

```json
{"date": "Jul 13, 2026", "note": "Refreshed pricing section after Q3 earnings call."}
```

## How to add a new competitor

Append a new object to the `competitors` array. There are two valid shapes:

**Minimum (shows up as "pending" in the sidebar, no card content):**
```json
{
  "id": "snowflake",
  "name": "Snowflake",
  "category": "Cloud data platform",
  "status": "pending",
  "pressure": null
}
```
`id` should be a short lowercase slug, unique across the file — it's used
internally to track which card is selected. `category` is the one-line
descriptor shown under the name everywhere in the UI.

**Full ("ready") card:** everything above, plus `lastUpdated`, `curator`,
`changelog`, and a `sections` object. See the schema reference below, or just
copy Databricks' entry as a template and replace the content.

## `sections` schema reference

Every field below is required for a competitor to render as "ready." If a
field is missing, that part of the card will silently render empty or throw —
easiest to always copy an existing full card as a starting template rather
than building a `sections` object from scratch.

| Field | Type | Renders as |
|---|---|---|
| `background` | string (HTML allowed, e.g. `<b>`) | Approach to Market → Background paragraph |
| `ma` | array of `{name, date, value}` (`value` can be `null`) | M&A list |
| `marketStrategy` | string | Market Strategy paragraph |
| `positioningEvolution` | array of strings | Numbered list under Market Strategy — use this only for genuine chronological sequences |
| `customers` | array of strings | Pill tags under Customers |
| `verticals` | string | Verticals Served paragraph |
| `partners` | string | Partners paragraph |
| `top` | array of `{point, subs: [strings]}` | Top Things to Know — each item is a headline point with supporting sub-bullets |
| `offerings.overview` | string | Product Offerings intro paragraph |
| `offerings.items` | array of `{name, desc}` | Product Offerings card grid |
| `claims.theirView` | array of strings | Product Claims — the competitor's public marketing claims |
| `claims.sasView` | array of strings | Product Claims — SAS's rebuttal |
| `claims.productRebuttal.product` | string | Name of the specific SAS product being compared (e.g. "SAS Viya Workbench") |
| `claims.productRebuttal.theirView` / `.sasView` | arrays of strings | Head-to-head rebuttal against that specific product |
| `dimensions` | array of `{name, verdict}` where `verdict` is `"sas"` or `"them"` | Strengths & Weaknesses — comparison dimension list with a colored advantage tag |
| `swot.strengths` / `swot.weaknesses` | arrays of strings | Strengths & Weaknesses — supporting bullet lists |
| `tactics.questions` | array of `{q, a}` | Sales Tactics — discovery questions and the pivot/answer for each |
| `tactics.analystNote` | `{tag, text}` | Sales Tactics — amber "needs verification" callout, for analyst/award claims that go stale (Gartner MQ standing, etc.) |
| `tactics.attack` | array of strings | Sales Tactics — angles reps should proactively raise |
| `tactics.defend` | array of strings | Sales Tactics — rebuttals to what the competitor's reps will say about SAS |
| `reframe` | string | Changing the Conversation — the main reframe/redirect paragraph |
| `betterTogether` | string | Changing the Conversation — complementary-sale messaging for accounts already committed to the competitor |
| `trustworthyAI.definition` / `.theirs` / `.ours` | strings | Changing the Conversation — Trustworthy AI messaging block |
| `trustworthyAI.needsLinks` | array of strings | Listed as "needs input" — named internal assets (sales sheets, decks) that don't have a live URL yet. Doesn't render as clickable; it's a to-do list, not a link list. |

Top-level fields alongside `sections`:

| Field | Type | Notes |
|---|---|---|
| `pressure` | integer 1–5, or `null` | Drives the "Competitive pressure" meter (segments + HIGH/MODERATE/LOW label) |
| `lastUpdated` | string | Shown as the "stamp" in the card header |
| `curator` | string | Small italic attribution line under the card title |
| `changelog` | array of `{date, note}` | Rendered oldest-to-newest as written — put newest entries first in the array |

## What's intentionally not here

Earlier versions of this dashboard had live, in-browser "Insights from the
Field" and "Recent Wins" boxes that saved automatically and were shared across
whoever opened the file. That relied on a storage API that only exists inside
Claude's own artifact environment — it does not exist on a plain static host
like Vercel, so it was removed rather than shipped broken.

If you want that functionality back on Vercel, it needs a real backend (even a
minimal one — a Vercel KV store or small database behind an API route) since a
static file can't accept writes from a browser. Until then, the equivalent is
just editing `data.json` directly, the same as everything else on this
dashboard.

## Weekly refresh workflow

This file has no automation of its own — nothing runs on a schedule. The
intended loop is:

1. Open a new chat with Claude (or continue this one).
2. Ask Claude to refresh a specific competitor, or all of them — Claude will
   re-research and hand you an updated `data.json` (or a diff/patch against it).
3. Replace `data.json` in your repo and redeploy.

If you have a live Klue export (or any other internal battlecard doc) for a
competitor that's still `"pending"`, upload it in the same chat — Claude can
merge internal-only sections (Product Claims, Sales Tactics discovery
questions, Trustworthy AI messaging) that public research alone can't produce,
the same way the Databricks card was built.
