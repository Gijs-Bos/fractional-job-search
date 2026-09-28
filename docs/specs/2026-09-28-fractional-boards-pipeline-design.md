# Fractional Boards Pipeline — Design

**Date:** 2026-09-28
**Status:** Approved. The review findings (R1–R13, M1–M10) and the 15-day limit on the first scan
(D17) were folded in on 2026-09-28.
**Author:** Claude (with Gijs)
**Related:** `docs/research/2026-09-28-fractional-boards-cowork-brief.md` (board discovery, run in Cowork)

## 1. Goal

Find **fractional, interim, freelance/project and advisory** sales-leadership roles on job boards,
portals, marketplaces and newsletters **other than LinkedIn**, twice a day; evaluate them against
editable criteria; email the new ones; capture feedback — reusing the existing `~/code/Jobs`
pipeline wherever it is source-agnostic.

The exact role criteria arrive later. They must be as easy to change as `filters.json` is for the
LinkedIn profiles. The board list arrives later too (Cowork research). This spec defines the
structure both plug into.

## 2. Decisions

| # | Topic | Decision |
|---|---|---|
| D1 | Deliverable | This spec → implementation plan → built in `~/code/Jobs` |
| D2 | Placement | New profile `fractional-boards` inside `~/code/Jobs` |
| D3 | Relationship to LinkedIn | LinkedIn fractional searches live in a **separate** LinkedIn profile (Gijs builds it). Board results and LinkedIn results go in **separate emails** |
| D4 | Engagement scope | Fractional, interim, freelance/project-based, advisory/NED/board roles |
| D5 | Geography | EU/UK + remote roles workable in CET hours; boards in EN/DE/NL |
| D6 | Source types | Public boards, free-account boards, newsletters/email alerts, freelance marketplaces. Paid boards are researched for information; connecting one is a later, explicit choice |
| D7 | ToS / scraping | Respect robots.txt, low volume, personal use; no bot-check or CAPTCHA circumvention |
| D8 | Paid tools | None — no paid APIs, scraping services or credits |
| D9 | Board discovery | Gijs runs it in Cowork with the brief; output lands in `docs/research/` |
| D10 | Email | Separate `[fractional-boards]` digest per slot, existing card format plus source and facts; **not sent when empty** |
| D11 | Feedback | Existing 👍/👎 + weekly review + triage, plus per-board hit-rate stats |
| D12 | Schedule | Existing 06:00 / 14:00 digest guard |
| D13 | Architecture | **Approach A + C**: generic, config-driven adapters per *access method*; email alerts/newsletters are one adapter type |
| D14 | Cross-profile suppression | **None.** A role already sent by the LinkedIn fractional profile may appear again here. Sent-history stays per profile |
| D15 | "Empty" | `sendWhen: "any"` — email when ≥1 new job (qualified or disqualified). Switch to `"qualified"` once the rubric is tuned |
| D16 | Gmail labels | Claude/the pipeline handles them — see §6.4. No Gmail filters are needed (the Gmail connector cannot create filters, so the design avoids them) |
| D17 | First scan | A source's first scan never puts a job **posted more than 15 days ago**, or with no posting date, into an email (§7.5) |

## 3. Learnings this design is built on

Each item was established in this repo; do not "simplify" it away without re-testing.

| # | Learning | Source | Consequence here |
|---|---|---|---|
| L1 | Evaluator, digest, sent-history and feedback only need `{company, title, location, url, description, evaluation}` keyed by an opaque job ID | code reading 2026-09-28 | New sources are adapters that emit that record; downstream stays unchanged |
| L2 | Two LinkedIn assumptions: numeric job IDs (`jobIdFromUrl`) and "new = URL in today's `*_new.csv`" matched by search prefix | `scrape-descriptions.js:116`, `send-daily-digest.js:680` | Namespaced IDs; adapters write the same dated CSVs; digest learns prefixes from `sources.json` |
| L3 | The Indeed design died on an LLM/MCP-only fetch path, Cloudflare, shared rate limits, fabricated tool calls and connector deauth | `docs/specs/2026-07-16-indeed-daily-digest-design.md` | Fetching is plain code. The LLM is used for evaluation only |
| L4 | Retrying while blocked extends the block (Indeed E14) | same | A bot challenge or robots block stops that source for the day; no retry loop |
| L5 | Silence ≠ healthy: a dead query and a thin market both yield zero rows; a per-region floor (`expectMinRows`) caught dead anchors precisely | Freiburg profile, 2026-09-10 | Per-source `expectMinRows7d` + `parse_zero` detection |
| L6 | "No results" pages are not empty — LinkedIn shows fallback recommendations | CLAUDE.md, 2026-09-09 | `html-list` requires a `noResultsSelector` wherever a board has an empty state |
| L7 | Liveness must never depend on an LLM run | global CLAUDE.md | Fetch, parse, health checks and heartbeat are deterministic Node/bash |
| L8 | Gmail silently drops some self-sent digests; delivery must be verified over IMAP | CLAUDE.md, `lib/mailer.js` | Reuse `deliverDigest` unchanged |
| L9 | Duplicated helpers drift (`"MUST stay identical"` comments in three files) | `scrape-descriptions.js`, `send-daily-digest.js` | Board code imports shared modules instead of copying |
| L10 | Any profile exiting 1 makes the guard re-run **every** profile on the next 15-minute poll | `run-digest.sh`, `run-digest-guard.sh` | Board failures are *warnings* in `run-digest.sh`, never failures (§9.3), and a board already fetched in a slot is not fetched again on a re-run (§6.1) |

## 4. Architecture

```
                      profiles/fractional-boards/sources.json   filters.json
                                         │                          │
run-digest.sh ──► send-daily-digest.js --run-pipeline --profile fractional-boards --slot HH:MM
                   │
                   ├─ (sources.json present) ─► fetch-boards.js --profile fractional-boards --slot HH:MM
                   │       for each enabled source due in this slot and not yet fetched in it (sequential):
                   │         adapter.validate → adapter.list → normalise → ID-collision check
                   │         → job_filters (exclude/include) → cross-board dedup (alsoOn)
                   │         → first-run age cap (15 days)
                   │         → per new job: CSVs first, then job-data, then adapter.detail
                   │           (incremental, crash-safe)
                   │         → job-data-fractional-boards.json + csv/{Board}_master|_new|_excluded
                   │         → logs/board-runs/<date>_<slot>.json  (per-source status)
                   │     (exit 1 is logged; evaluation and the digest still run on data on disk)
                   │
                   ├─ evaluate-jobs.js --profile fractional-boards --days 7 --via-cli   (unchanged)
                   │
                   ├─ digest: new = URLs in today's and yesterday's csv/*_new.csv for this profile's
                   │          source prefixes → sent-history filter → sendWhen check
                   │          → deliverDigest (IMAP-verified)
                   │
                   └─ board_health: alert email if a source is failing / under its floor (≤1/day/source)

web.js  /fb/:jobId/fractional-boards/:signal · weekly review · /feedback triage · /boards (new stats page)
```

LinkedIn profiles (`default`, `ops-strategy`, `freiburg`) are unaffected: without `sources.json`
the pipeline step runs `scrape-linkedin.js` + `scrape-descriptions.js` exactly as today.

## 5. Configuration

### 5.1 Profile layout

```
profiles/fractional-boards/
  sources.json   which boards and how to fetch them
  filters.json   exclusions + evaluationCriteria (same format as other profiles)
  csv/           this profile's CSVs
  state/         per-source cursors, first-run markers, slot stamps and block flags
```

### 5.2 `sources.json`

```json
{
  "csvSubdir": "csv",
  "sendWhen": "any",
  "defaults": {
    "slots": ["06:00", "14:00"],
    "delayMs": [3000, 8000],
    "maxPages": 3,
    "firstRunMaxAgeDays": 15,
    "timeoutMs": 30000
  },
  "sources": [
    {
      "id": "board-a", "name": "Board A", "type": "rss", "enabled": true,
      "url": "https://example.com/feed",
      "map": { "company": "author" },
      "detail": { "engine": "fetch", "selector": ".job-description" },
      "expectMinRows7d": 2
    },
    {
      "id": "board-b", "name": "Board B", "type": "html-list", "engine": "fetch",
      "listUrls": ["https://example.com/jobs?q=sales"],
      "pagination": { "param": "page", "start": 1 },
      "selectors": { "card": ".job", "title": "h3", "company": ".company", "location": ".loc",
                     "link": "a.job-link", "posted": "time@datetime" },
      "noResultsSelector": ".empty-state",
      "idPattern": "/jobs/([a-z0-9-]+)",
      "facts": { "dayRate": ".rate", "daysPerWeek": ".commitment" }
    },
    {
      "_comment": "Abbreviated: a real entry also needs listUrls and selectors, as for board-b.",
      "id": "marketplace-c", "name": "Marketplace C", "type": "html-list", "engine": "browser",
      "loginUrl": "https://example.com/login",
      "loggedOutSelector": "a[href*='login']", "slots": ["06:00"]
    },
    {
      "id": "newsletter-d", "name": "Newsletter D", "type": "imap-alerts",
      "from": ["jobs@newsletter-d.example"], "subjectContains": "roles",
      "parser": "newsletter-d", "archiveAfterIngest": false
    }
  ]
}
```

Rules:
- `id` is lowercase `[a-z0-9-]`, unique, **never renamed** (it is part of every job ID). Its
  CSV prefix is `boardPrefix(id)` = `sanitizeForFilename(id.replace(/-/g, ' '))`, so `board-a` →
  `Board-A`. Plain `sanitizeForFilename` only splits on whitespace and would give `Board-a`.
  Verified against the function on 2026-09-28.
  - `validate()` rejects two ids that produce the same `boardPrefix`.
- URL parameters (§6.5):
  - By default only tracking parameters (`utm_*`, `ref`, `source`, `fbclid`, `gclid`, `mc_cid`,
    `mc_eid`) and the fragment are stripped. Other query parameters are kept, because many
    boards identify a posting by query (`/job?id=123`).
  - Optional `stripParams: ["sort", …]` adds per-board parameters to strip. Use it for
    parameters that vary between runs without identifying the posting.
- Unknown keys are an error in `validate()`, so typos fail at startup, not silently at 06:00.
  **Keys starting with `_` are ignored**, so `"_comment": "…"` can document an entry. JSON has
  no comments.
- `enabled` defaults to `true`. `enabled: false` keeps history and stats but stops fetching.
- Per-source keys override `defaults`.
- `firstRunMaxAgeDays` defaults to 15 (D17). A per-source override may lower it but not raise
  it; `validate()` rejects a value above 15.

### 5.3 `filters.json`

The same format and semantics as the other profiles: `excludeCompanies`,
`excludeTitleKeywords`, `includeTitleKeywords`, `excludeCountries`, `companyAliases`,
`evaluationCriteria`. There is no `workTypePolicy`, because remote-first is the norm here.
Exclusions run at fetch time and go to `_excluded.csv` with a Reason, as on LinkedIn.

**Initial content:** three placeholder criteria, each marked `"_placeholder": true`, so the
pipeline can be tested end-to-end before Gijs's real criteria exist:
1. `engagement_in_scope` (boolean, disqualify): the role is fractional, interim, freelance/project
   or advisory, and is not permanent full-time, commission-only or equity-only.
2. `sales_leadership` (boolean, disqualify): a senior B2B sales, commercial or revenue leadership
   remit.
3. `europe_workable` (boolean, disqualify): based in the EU or UK, or remote and workable in CET
   hours.

There is no `includeTitleKeywords` at first, because fractional titles are irregular. Section 8's
per-board stats decide whether one is needed.

Replacing the placeholders changes the criteria hash, and every job is re-evaluated automatically.
`fetch-boards.js` logs a warning each run while any placeholder criterion remains.

## 6. Components

### 6.1 `fetch-boards.js` (new)

```
node fetch-boards.js --profile fractional-boards [--slot HH:MM] [--source <id>] [--dry-run]
                     [--login <id>]
```

- `--slot` selects the sources whose `slots` include it. With no slot, all enabled sources run.
- `--source` runs one source, e.g. when adding or debugging a board.
- `--dry-run` fetches and parses, then prints the rows, facts and health verdict. It writes
  nothing: no job-data, CSV, state or run log. On a source's first run it also shows which items
  the 15-day cap would hold back.
- `--login <id>` opens a **headed** browser on that source's `loginUrl` using `.browser-data/`,
  and waits for Gijs to log in and close the window. It never types credentials.
- Sources run **sequentially** (a polite, predictable load). Each source runs inside its own
  try/catch, so one source can never abort the others.
- **Once per slot.** After a source finishes `ok` or `empty`, the run saves
  `state/<id>.last-ok = { date, slot }`. A later run in the same date and slot skips that source
  (`skipped`, note `already fetched this slot`). This matters when the guard re-runs every
  profile every 15 minutes because a LinkedIn profile failed (L10): boards must not be fetched
  again each time.
- **Write order, per new job.** This order means a crash can never leave a job marked as seen
  but missing from the day's new-jobs CSV:
  1. Append it to the master CSV and today's `_new.csv`.
  2. Write it to job-data (atomic).
  3. Run `detail`.
  4. Write job-data again (atomic).

  A crash after step 1 can leave a duplicate CSV row on the next run. That is harmless, because
  the digest filters through sent-history. Job-data writes use the same atomic tmp-file-and-rename
  pattern as the existing scripts.
- Exit codes:
  - `0`: the run completed, even if individual sources failed. Those failures are reported
    through the run log and health alerts.
  - `1`: `sources.json` or `filters.json` is invalid, or an unexpected crash occurred.
  - Exit `2` is **not** used, because `run-digest.sh` reserves it to mean "logged out of
    LinkedIn".

### 6.2 Adapter interface: `lib/sources/<type>.js`

```js
module.exports = {
  type: 'rss',
  validate(cfg) { return [/* error strings */]; },
  async list(cfg, ctx) {
    // → { status: 'ok' | 'empty' | <failure status>, items: RawListing[], note?: string }
    // RawListing = { nativeId?, url, title, company?, location?, postedAt?, teaser?, facts? }
  },
  async detail(cfg, item, ctx) {           // optional
    // → { description, facts? }  | throws
  },
};
```

`ctx` = `{ http, browser(), imap(), log, now, state, isKnown(jobId), firstRun }`:
- `ctx.http.get(url)` (in `lib/sources/http.js`):
  - Checks robots.txt, cached per host per run, and evaluates the rules for the `*` user agent,
    following RFC 9309:
    - `robots.txt` returns 4xx → everything is allowed.
    - `robots.txt` returns 5xx or a network error → nothing is fetched from that host this run.
      The source's status is `error`, not `robots_blocked`.
  - Waits a random delay from `delayMs` between requests to the same host.
  - Uses a normal browser User-Agent.
  - Detects bot challenges: HTTP 403, 429 or 503, or page text markers such as
    `cf-challenge`, `Just a moment`, `captcha` and `challenge-platform`. It returns a typed
    result instead of throwing.
- `ctx.browser()` starts the Playwright persistent context on `.browser-data/` lazily, only when a
  `browser`-engine source actually runs. It reuses `lib/browser.js`. If Chromium's profile lock
  (`SingletonLock`) is held, for example because `--login` is open or another profile is running,
  the source returns `error` with the note `browser profile in use`.
- `ctx.imap()` opens an `imapflow` session with the existing `GMAIL_USER` / `GMAIL_APP_PASSWORD`.

### 6.3 Adapter types

| Type | For | Mechanics |
|---|---|---|
| `rss` | Boards with RSS/Atom feeds | `fast-xml-parser`. `nativeId` comes from `guid`/`id`. `map` renames fields when a board repurposes them |
| `json-api` | Boards with a public JSON endpoint | `url` + `itemsPath` + a field `map` (dot paths). Pagination by param or cursor |
| `html-list` | Everything else | `engine: fetch` → `cheerio` on the HTTP body. `engine: browser` → Playwright page, then the same selector logic on `page.content()`. A `noResultsSelector` hit returns `empty`. `loggedOutSelector` returns `logged_out`. Selectors take the form `css` or `css@attribute` |
| `imap-alerts` | Newsletters and board alert emails | See §6.4 |

**Pagination** (`html-list` and `json-api`) stops at the first of:
- `maxPages`
- a page with 0 items (the end of the results, not a failure)
- a page whose items are all known IDs, so usually only page 1 is fetched per slot
- on a source's first run, a page whose items all have a `postedAt` older than
  `firstRunMaxAgeDays` (the list is assumed to be newest first)

`empty` and `parse_zero` (§9.1) are decided on **page 1 only**.

**Posting dates.** `postedAt` is taken from an absolute date or datetime where the board gives
one. Relative dates ("3 days ago", "vor 3 Tagen", "3 dagen geleden") are converted by one small
EN/DE/NL parser in `lib/sources/dates.js`. Anything else leaves `postedAt: null`. For
`imap-alerts`, `postedAt` is the email's `Date` header.

**Detail step (`detail`).**
- It runs only for **new** items with no full description yet: `scrapeStatus` is not `ok`,
  `seen_first_run` or `detail_failed`. It uses `detail.selector` on the item URL, with
  `engine: fetch` or `browser`.
- For sources with a detail step, a description under 300 characters counts as a teaser and is
  fetched again next run. It only becomes `scrapeStatus: 'ok'` once full, so the evaluator never
  judges a teaser.
- After 3 failed attempts, if the record has `alsoOn` entries, the detail step is tried once on
  each `alsoOn` URL using that source's own `detail` config. If none of them gives a full
  description, the item is marked `scrapeStatus: 'detail_failed'` and appears in the health
  output.
- An adapter can declare `detail: false` when its list data already contains full descriptions,
  as some JSON APIs and newsletters do.

**Facts.** Adapters extract `facts` (`engagement`, `daysPerWeek`, `dayRate`, `duration`,
`remote`, `startDate`) **only where the board states them as fields or labelled text**. Facts are
never guessed. They are rendered at the top of `description` as one line, for example
`Listing facts: 2 days/week · EUR 900/day · 6 months · remote`, so the unchanged evaluator sees
them. The digest card also shows them.

### 6.4 `imap-alerts` and Gmail labels (D16)

- **Selecting mail.** Messages are selected by Gmail search over IMAP (`X-GM-RAW`):
  `from:(<from…>) newer_than:7d` plus `subject:` when `subjectContains` is set. The search runs
  over All Mail, so there are **no Gmail filters to create or maintain**. If the pipeline is down
  for more than 7 days, mail from that gap is not ingested. This is accepted.
- **Tracking what's been read.** Processed Gmail message IDs (`X-GM-MSGID`) are stored in
  `state/imap-<id>.json`. A message is processed exactly once, even if it is later moved or
  read. IDs older than 30 days are pruned.
- **Labelling.** After a message is ingested successfully, the pipeline adds the Gmail label
  `jobs/fractional-boards/<id>` via IMAP (`X-GM-LABELS`), for traceability in Gmail. If
  `archiveAfterIngest: true`, it also removes `\Inbox`. That's a per-source choice; the default
  is false.
  - This is deterministic code, so the label is applied even when nobody is watching.
  - Where labels need creating up front (for example, if Gmail refuses to auto-create them over
    IMAP), Claude creates them once through the Gmail connector, **with Gijs's approval for each
    label**.
- **Parsing.**
  - Each sender gets a small deterministic parser in `lib/sources/parsers/<parser>.js`, using
    cheerio on the HTML part. A parser returns `{ items, noRolesMarker }`.
  - Each role becomes one RawListing. The URL is the role's own link.
  - **An item with no link** gets the URL `<issue web or archive URL>#<h>` and
    `nativeId = <h>`, where `h = hash(title|company)`. Setting `nativeId` directly matters:
    canonicalising the URL removes the fragment, so an ID taken from the URL would be the same for
    every link-less role in the issue and all but the first would be dropped.
  - When an email arrives from a configured sender but yields 0 items and no known "no roles
    this week" marker, the result is `parse_zero`, which usually means the template changed.
  - There is no LLM extraction by default. Using an LLM for a single irregular newsletter would be
    an explicit per-source exception, recorded in that source's `_comment` in `sources.json`.
- **Subscriptions.** Signing up for newsletters and alerts is done by Gijs. Claude does not
  subscribe or create accounts. They must be addressed to `GMAIL_USER`.

### 6.5 Shared modules (new)

| Module | Purpose |
|---|---|
| `lib/board_ids.js` | `boardPrefix(id)`; `jobId(sourceId, {nativeId, url})` (§7.1); `canonicalUrl(url, stripParams)`, which removes tracking parameters, any `stripParams` and the fragment and is used **only for hashing**; `displayUrl(url)`, which removes tracking parameters only and is what `record.url` and the CSVs store; `normCompany(company, sourceName)`, which maps anonymous companies to `null` (§7.4); `normKey` by calling `sent_history.normalizeKey` |
| `lib/job_filters.js` | Exclusion and whitelist matching with the same rules as CLAUDE.md "Exclusion Matching Rules". **`scrape-linkedin.js` is not changed in this project.** Moving it onto this module is a follow-up with a regression test on real master CSVs |
| `lib/board_health.js` | Reads run logs and computes the status table, the `expectMinRows7d` verdicts, consecutive-failure counts and alert eligibility |
| `lib/sources/http.js` | The polite HTTP client described in §6.2 |
| `lib/sources/dates.js` | Absolute and relative (EN/DE/NL) posting-date parsing (§6.3) |

New dependencies: `cheerio`, `fast-xml-parser`.

## 7. Data model

### 7.1 Job IDs

The format is `<sourceId>:<nativeId>`. `nativeId` is resolved in this order:
1. The board's own stable ID (feed `guid`, API `id`, or `idPattern` on the URL), or the
   `nativeId` a newsletter parser set for a link-less item (§6.4).
2. The first 12 hex characters of sha1(`canonicalUrl(url)`).

The `:` separator means an ID can never collide with LinkedIn's numeric IDs. IDs are already
URL-encoded in `/fb/:jobId/:profile/:signal` links (`web.js:660`).

**Collision check.** If two *different* display URLs in one `list()` result resolve to the same
job ID, the source's parameter settings are wrong: some query parameter identifies postings but
is being stripped. The second item is still dropped, but the run log records note `id_collision`
with both URLs. That is an alert (§9.2).

### 7.2 Record in `job-data-fractional-boards.json`

The top level is `{ version, jobs: { [jobId]: record } }`, the same as the other profiles. The
record uses the existing fields plus some additive ones:

```js
{
  company, title, location, url,                   // existing; url = displayUrl (§6.5)
  description, descriptionScrapedAt, scrapeStatus, // existing; evaluator requires scrapeStatus 'ok'
  evaluation,                                      // written by evaluate-jobs.js, unchanged
  source:   { id, name, type },                    // new
  alsoOn:   [{ sourceId, url, seenAt }],           // new — cross-board duplicates (§7.4)
  postedAt, fetchedAt, firstSeenAt,                // new
  facts:    { engagement, daysPerWeek, dayRate, duration, remote, startDate },  // new, all optional
  detailAttempts                                   // new
}
```

`scrapeStatus` takes the existing values plus two new ones:
- `seen_first_run`: held back by the first-run age cap (§7.5). It has no description. The detail
  step, the evaluator and the health counts all skip it.
- `detail_failed`: see §6.3.

`company` is stored as the board gave it. `normCompany()` is applied only when building dedup and
sent-history keys.

### 7.3 CSVs

These live in `profiles/fractional-boards/csv/`, selected by `csvSubdir` in `sources.json`.
There is one file set per source, with the prefix `boardPrefix(id)` (§5.2):

| File | Columns |
|---|---|
| `Board-A_master.csv` | Company, Job Title, Location, URL |
| `YYYY-MM-DD_Board-A_new.csv` | Company, Job Title, Location, URL |
| `YYYY-MM-DD_Board-A_excluded.csv` | Company, Job Title, Location, URL, Reason |

These are exactly the existing column contracts, so `parseURLsFromCSV` and the digest's "new"
logic work unchanged. The master CSV is capped at `masterCap` rows (default 2000), as on
LinkedIn. Anything that selects files per source (the digest, the `/boards` stats) matches
`_<prefix>_new.csv`, `_<prefix>_excluded.csv` and `<prefix>_master.csv` **exactly**, never with
`startsWith`. Otherwise source `board` would also count `board-a`'s files.

### 7.4 Dedup

**Anonymous companies.** Fractional boards and newsletters often hide the company or leave it
out. `normCompany()` maps these values to `null`: empty; `confidential`; anything starting with
`stealth`; `undisclosed`; `anonymous`; `n/a`; and the board's own name.

1. **Within a source, across runs.** By job ID. An item already in job-data is not new.
2. **Across sources, at fetch time.**
   - Only when `normCompany(company)` is not `null`. Otherwise unrelated roles with generic titles
     such as "Fractional CRO" would be merged.
   - If `normKey(company, title)` (with the profile's `companyAliases`) matches a record from a
     *different* source first seen within 30 days, the new listing is **not** stored as a new
     job.
   - Instead it is appended to that record's `alsoOn`, and to the source's `_excluded.csv` with
     reason `duplicate_of:<jobId>`, so the per-board stats can count it.
   - This avoids a second evaluation and lets the card say "also on …".
   - **Exception:** a match against a `seen_first_run` record counts as a duplicate only if the
     new listing has no `postedAt` or was posted more than `firstRunMaxAgeDays` ago. A listing
     posted recently elsewhere is stored as its own job.
3. **Across days.** `sent-history-fractional-boards.json` via the unchanged `lib/sent_history.js`
   (job ID plus the 30-day `company|title` repost key). When `normCompany(company)` is `null`,
   the company passed to sent-history is `anon:<jobId>`. That turns off repost matching for the
   job without changing `lib/sent_history.js`, so one generic "Fractional Head of Sales" does not
   hold back every later one for 30 days.
4. **Not across profiles (D14).**

### 7.5 First-run age cap (D17)

A source's first scan must not flood the first email with old roles. On a source's first
successful run (status `ok` or `empty`, with no `state/<id>.first-run` marker yet):
- Items **posted more than 15 days ago** (`firstRunMaxAgeDays`, max 15, §5.2) are written to the
  master CSV and to job-data with `scrapeStatus: 'seen_first_run'`, but **not** to `_new.csv`.
  So they never reach an email, are never detail-fetched and are never evaluated.
- Items **without `postedAt`** are treated the same way, because their age can't be shown to be
  within 15 days.
- Items posted within 15 days follow the normal path.
- Paging stops early at a page of only old items (§6.3), so the first scan stays within the load
  budget.
- The marker is written after the run completes. A first run that ends in any other status (for
  example `bot_challenge` or `error`) writes no marker, so the cap applies again on the next
  attempt.
- `imap-alerts` sources already only read mail from the last 7 days (§6.4). The cap still
  applies to them, using the email's `Date` header as `postedAt`.
- A newly added source gets a first run. Disabling and re-enabling a source keeps its marker, so
  it gets no second first run. Deleting the marker forces one.

## 8. Email, feedback and stats

### 8.1 Digest (changes to `send-daily-digest.js`)

- **Pipeline dispatch.** If the profile dir contains `sources.json`, run `fetch-boards.js`
  (passing `--slot`) instead of `scrape-linkedin.js` + `scrape-descriptions.js`. The LinkedIn
  scrape-stamp logic is not used; the per-source slot stamp (§6.1) replaces it.
- **Fetch failures.** If `fetch-boards.js` exits 1, the digest logs it and still evaluates and
  sends what is already on disk, with `fetch failed: <reason>` in the footer. If `sources.json`
  itself cannot be parsed, the prefixes are unknown, so the digest step exits non-zero, which
  counts as a board warning (§9.3).
- **Prefixes.** `getProfileSearchPrefixes` returns `boardPrefix(id)` for every source in
  `sources.json` (enabled or not, so late-evaluated jobs from a just-disabled board still count).
- **`resolveCsvDir`** (`lib/profile_paths.js`) also reads `csvSubdir` from `sources.json`.
- **"New" window.** For profiles with `sources.json`, "new" is the URLs in **today's and
  yesterday's** `_new.csv` files, not today's only. Sent-history stops repeats. This means a
  failed 14:00 send is picked up at 06:00 the next day. Board failures are only warnings (§9.3),
  so the guard never retries a board digest, and a job would otherwise be lost. LinkedIn profiles
  keep today's window.
- **`sendWhen`.**
  - `"any"`: skip sending when there are 0 new qualified, 0 disqualified and 0 pending jobs.
  - `"qualified"`: skip sending when there are 0 qualified.
  - A skipped send logs `DIGEST SKIPPED (sendWhen=…)` and records nothing in sent-history.
  - Profiles without `sendWhen` keep today's behaviour, including the "No new jobs today" email.
- **Card additions,** rendered only when the fields exist, so LinkedIn cards are unchanged:
  - a source line: `via Board A · also on Board B, Newsletter D`
  - a facts row
  - the posted date

  These values come from scraped HTML and newsletters, so they go through the **same HTML
  escaping** as the existing card fields.
- **Footer:** a board health summary, e.g. `Sources: 9 ok · 1 empty · 1 failing (Board C:
  logged_out since 2026-10-01)`.
- **Subject:** `3 qualified, 1 disqualified [fractional-boards] — YYYY-MM-DD`, using the existing
  profile label.
- **Delivery:** the existing `deliverDigest` path, unchanged, with IMAP verification and bisect.
- **Health step.** For profiles with `sources.json`, after the digest step (whether it sent or
  skipped), `send-daily-digest.js` calls `lib/board_health.js` and sends any due alert email
  (§9.2). A failure here is logged and never changes the exit code.

### 8.2 Feedback

- Register `fractional-boards` in `web.js` `PROFILES`, `lib/feedback_triage.js` `PROFILES` and
  `send-review-digest.js` `sealSession`. This is the procedure CLAUDE.md prescribes. The 👍/👎
  links, reasons, weekly review, `/feedback` triage and "Analyze" then work unchanged.
- "Analyze" proposes edits to `profiles/fractional-boards/filters.json`, as for other profiles.

### 8.3 Per-board stats: `/boards` page in `web.js` and a weekly-review line

These are computed on request from data already on disk: job-data, dated CSVs, `feedback.json`
and `logs/board-runs/`. There is no new store. For each source, over 7, 30 and 90 days:

| Column | Definition |
|---|---|
| Fetched | rows in `_new` + `_excluded` CSVs |
| Pre-filtered | `_excluded` rows by reason (filters vs `duplicate_of`) |
| Evaluated | jobs from this source with a verdict |
| Qualified | qualified jobs this source owns **or** appears on through `alsoOn`. Sources run in config order, so without this the board listed first would get all the credit for shared jobs |
| 👍 / 👎 | feedback signals on this source's jobs |
| **Unique qualified** | qualified jobs with no `alsoOn` entry, i.e. not on any other board. This is the column that decides whether to drop a board |
| Health | latest status, plus any failure streak |

The weekly review email gets one line per board: `Board A: 14 fetched · 3 qualified (2 unique) ·
👍2 👎1 · ok`. Dropping a board means setting `enabled: false`. There is no UI for it.

## 9. Health, failure handling and scheduling

### 9.1 Per-source status

Each run writes `logs/board-runs/<date>_<slot>.json` = `{ sourceId: { status, rows, newRows,
ms, note } }`. Pruned after 90 days.

| Status | Meaning | Response |
|---|---|---|
| `ok` | ≥1 row parsed | — |
| `empty` | the no-results marker was seen, or a feed/API legitimately returned 0 | normal; counts toward the floor check |
| `parse_zero` | HTTP 200 on page 1 with a non-trivial body, but 0 rows and no no-results marker | template or layout changed → alert |
| `bot_challenge` | challenge page or 403/429/503 | skip this source until the next calendar day (state flag); alert. **No retries (L4)** |
| `robots_blocked` | robots.txt disallows the path | skip permanently until the config changes; alert once |
| `logged_out` | `loggedOutSelector` matched | alert with `node fetch-boards.js --login <id>`; do not retry today |
| `error` | network error, timeout, robots.txt unreachable (§6.2), browser profile in use, or a crash in the adapter | retried at the next slot; alert after 2 consecutive |
| `skipped` | not due this slot, disabled, blocked for today, or already fetched this slot (§6.1) | not a failure |

A run-log `note` of `id_collision` (§7.1) comes with any status and is alert-worthy on its own.

### 9.2 Alerts

- **Floor check.** A source with `expectMinRows7d` that has fewer rows than that in its last
  7 days of run logs is flagged `below_floor`. The check applies only once the source has run
  logs covering 7 days, so a newly added source does not alert all week.
- **Alert email.** `lib/board_health.js` decides which sources are alert-worthy:
  - any `parse_zero`, `bot_challenge`, `robots_blocked`, `logged_out` or `below_floor`
  - an `id_collision` note
  - `error` two or more times in a row
  The alert is a short plain email (`[fractional-boards] source health — …`) sent via
  `lib/mailer.js` with IMAP verification.
  - It is sent at most once per source per day, tracked in `logs/board-health-alerts.json`.
  - A condition that persists is alerted daily for its first 3 days, then weekly, until it
    clears. For example, a board that stays `bot_challenge`.
  - It is sent whether or not the digest was sent, because on quiet days no digest goes out
    (D10). That is the reason this alert exists.
  - The wording is plain and neutral. Words like "spam" or "dropped" are avoided, because they
    are known to trip Gmail's content scoring (CLAUDE.md).
- **Placeholder criteria** are logged, not emailed.

### 9.3 `run-digest.sh` integration (L10)

- Add `fractional-boards` to `profiles` at **every slot**, **last** in the order. It runs after
  the LinkedIn profiles, because browser-engine sources share `.browser-data/`. Per-source
  `slots` handle once-a-day boards.
- Exit handling for this profile:
  - Exit `4` (evaluator outage) is handled exactly like the other profiles: `eval_outages++`.
  - Any **other** non-zero exit counts as `board_warnings++` with detail, **not** `failures++`.
  - Rationale: `failures` makes the guard re-run *every* profile on the next 15-minute poll. A
    broken board must never cause repeated LinkedIn re-runs and duplicate "No new jobs today"
    emails. A failed board send is recovered by the two-day "new" window instead (§8.1).
- The heartbeat message gains `WITH BOARD WARNINGS: …`, next to the geo warnings, so
  `~/.monitoring` sees it.
- Check the plan-only mode: `DIGEST_PLAN_ONLY=1 bash run-digest.sh 06:00` lists `fractional-boards`.

### 9.4 Load budget

Expected load, with ~15 sources × ≤3 list pages × ~5.5 s plus ~10 new detail pages × ~5.5 s:
about **5 minutes per slot** and at most a few requests per host per slot. In steady state
pagination usually stops after page 1 (§6.3), so it is less. A first run fetches more list pages
but no detail pages for old items (§7.5). If the per-slot time goes above 15 minutes, the
plan-level fix is lowering `maxPages`, not parallelism.

## 10. Testing

Plain `node test_*.js` scripts, matching the repo style. No test framework and no network.
Fixtures live in `test/fixtures/boards/`.

| Test | Covers |
|---|---|
| `test_board_ids.js` | ID resolution order; tracking-parameter stripping with identifying parameters kept; `stripParams`; `displayUrl` vs `canonicalUrl`; the `:` namespace; prefixes, including two ids that sanitise to the same prefix; two link-less newsletter items from one issue get **different** IDs; `normCompany` |
| `test_board_http.js` | robots allow and disallow, robots 4xx and 5xx, each challenge marker, timeout → `error`, the per-host delay (injected clock) |
| `test_board_dates.js` | absolute dates and EN/DE/NL relative dates; unknown formats → `null` |
| `test_board_adapters.js` | Each adapter against fixtures: an RSS feed, a JSON response, an HTML list, an HTML no-results page (→ `empty`), a changed-layout page (→ `parse_zero`), a page-2 with 0 cards (→ end of results, not `parse_zero`), a challenge page (→ `bot_challenge`), a logged-out page, and one `.eml` per newsletter parser; early pagination stop on known IDs and on old items in a first run |
| `test_fetch_boards.js` | The pipeline with mocked adapters: per-source isolation (one throws, the others complete); cross-board `alsoOn` dedup, and no dedup for an anonymous company; the first-run cap: items 16 days old and items with no date are held back as `seen_first_run` and not detail-fetched, items 15 days old or newer are new, and no marker after a failed first run; teaser → detail retry → `alsoOn` fallback → `detail_failed`; `id_collision`; slot selection and the once-per-slot skip; a crash between the CSV write and the job-data write still leaves the job in `_new.csv`; `--dry-run` writes nothing (directory snapshot before and after); `_comment` keys accepted and other unknown keys rejected; `firstRunMaxAgeDays` above 15 rejected; the exit codes |
| `test_board_health.js` | The status table, the floor check and its 7-day grace, consecutive-error counting, `id_collision` alerts, once-per-day suppression and the weekly back-off, and that the alert text contains no "spam" or "dropped" |
| `test_digest_sendwhen.js` | `sendWhen` any/qualified/absent (the absent case keeps the legacy email); prefixes from `sources.json` with exact matching; the two-day "new" window for board profiles and the one-day window for LinkedIn profiles; `anon:<jobId>` in sent-history; a LinkedIn-shaped card renders byte-identical to before; a board card with a `<script>` title comes out escaped; exit 1 from `fetch-boards.js` still sends from disk |
| `test_run_digest_slots.sh` (extend) | `fractional-boards` is listed at both slots; a non-4 exit counts as a board warning, not a failure |

Existing tests must still pass. That confirms the LinkedIn path is unchanged:
`test_profile_paths.js`, `test_digest_delivery.js`, `test_scrape_stamp.js`,
`test_run_digest_slots.sh`, `test_feedback_triage.js`, `test_review_render.js`.

**End-to-end check before enabling the schedule:**
1. `node fetch-boards.js --profile fractional-boards --source <first real board> --dry-run`.
   Check that nothing older than 15 days would be new.
2. `node send-daily-digest.js --run-pipeline --via-cli --profile fractional-boards --dry-run`
3. One real send, verified in the Gmail Inbox via IMAP (`delivery: inbox`).

## 11. Rollout

**Phase 1: framework.** This does not depend on the Cowork results.
- All components in §6–§9.
- The `rss`, `json-api`, `html-list` and `imap-alerts` adapters, proven against fixtures.
- One real public board end-to-end, chosen from the Cowork shortlist if it is ready, otherwise
  any public fractional board with an RSS feed or clean HTML.

**Phase 2: boards.** One small task per shortlisted board or newsletter:
1. config entry
2. `--dry-run`
3. fixture
4. test
5. enable

Newsletter subscriptions (Gijs) and the Gmail labels (Claude, §6.4) come here.

**Phase 3: tuning.**
- Gijs supplies the real criteria, which replace the placeholders.
- After 2–4 weeks, switch `sendWhen` to `"qualified"` if the rubric is stable.
- Use `/boards` stats to drop weak boards and decide on an `includeTitleKeywords` whitelist.

**Documentation:** a new "Fractional boards" section in `CLAUDE.md`. It covers config, adding a
board, `--dry-run`, `--login`, the statuses and alerts, and a pointer to this spec. The heading
"JobS Project" also becomes `~/code/Jobs`.

## 12. Out of scope

- Any change to the LinkedIn pipeline's behaviour or output, including migrating
  `scrape-linkedin.js` to `lib/job_filters.js` (follow-up).
- Suppressing roles across profiles (D14).
- Paid tools, APIs or scraping services (D8).
- Circumventing bot checks, CAPTCHAs or robots.txt (D7).
- Creating accounts, subscribing to newsletters, or entering credentials. Gijs does these.
- LLM-based extraction as a default path (L3, L7).
- Parallel fetching.

## 13. Assumptions to verify against the code

These parts of the design depend on how existing code behaves. They were not all checked when
the spec was written. Phase 0 of the plan checks them before any code changes. If one turns out
wrong, change this spec first.

| # | Assumption | Where | Used by |
|---|---|---|---|
| A1 | The "new" logic can take a date window (today + yesterday) | `send-daily-digest.js:~680` | §8.1 "New" window |
| A2 | `sent_history` takes a caller-supplied company string, so `anon:<jobId>` works without changes | `lib/sent_history.js` | §7.4.3 |
| A3 | Which date `evaluate-jobs.js --days 7` filters on. Jobs reach `ok` within about 1.5 days (3 detail attempts at 2 slots/day), which is inside the window for a CSV date or `descriptionScrapedAt`. Only `firstSeenAt` plus a long `bot_challenge` pause could push one out | `evaluate-jobs.js` | §6.3 |
| A4 | What `--run-pipeline` does today when the scrape step fails | `send-daily-digest.js` | §8.1 Fetch failures |
| A5 | The card HTML escape helper, reused for the new fields | `send-daily-digest.js` | §8.1 Card additions |
