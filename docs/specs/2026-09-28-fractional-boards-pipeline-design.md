# Fractional Boards Pipeline — Design

**Date:** 2026-09-28
**Status:** Design approved in conversation; awaiting review of this written spec
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
| L10 | Any profile exiting 1 makes the guard re-run **every** profile on the next 15-minute poll | `run-digest.sh`, `run-digest-guard.sh` | Board failures are *warnings* in `run-digest.sh`, never failures (§9.3) |

## 4. Architecture

```
                      profiles/fractional-boards/sources.json   filters.json
                                         │                          │
run-digest.sh ──► send-daily-digest.js --run-pipeline --profile fractional-boards --slot HH:MM
                   │
                   ├─ (sources.json present) ─► fetch-boards.js --profile fractional-boards --slot HH:MM
                   │       for each enabled source due in this slot (sequential):
                   │         adapter.validate → adapter.list → normalise → job_filters (exclude/include)
                   │         → cross-board dedup (alsoOn) → first-run age cap
                   │         → adapter.detail (new jobs only, incremental, crash-safe)
                   │         → job-data-fractional-boards.json + csv/{Board}_master|_new|_excluded
                   │         → logs/board-runs/<date>_<slot>.json  (per-source status)
                   │
                   ├─ evaluate-jobs.js --profile fractional-boards --days 7 --via-cli   (unchanged)
                   │
                   ├─ digest: new = URLs in today's csv/*_new.csv for this profile's source prefixes
                   │          → sent-history filter → sendWhen check → deliverDigest (IMAP-verified)
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
  state/         per-source cursors and first-run markers
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
    "firstRunMaxAgeDays": 14,
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
      "id": "marketplace-c", "name": "Marketplace C", "type": "html-list", "engine": "browser",
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
- Optional `keepParams: ["id", …]`: query parameters that identify a posting and must survive
  URL canonicalisation. By default the whole query string is stripped.
- Unknown keys are an error in `validate()`, so typos fail at startup, not silently at 06:00.
- `enabled: false` keeps history and stats but stops fetching.
- Per-source keys override `defaults`.

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
  nothing: no job-data, CSV, state or run log.
- `--login <id>` opens a **headed** browser on that source's login page using `.browser-data/`,
  and waits for Gijs to log in. It never types credentials.
- Sources run **sequentially** (a polite, predictable load). Each source runs inside its own
  try/catch, so one source can never abort the others.
- Writes are crash-safe: job-data is saved after each detail fetch, using the same atomic
  tmp-file-and-rename pattern as the existing scripts.
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

`ctx` = `{ http, browser(), imap(), log, now, state }`:
- `ctx.http.get(url)` (in `lib/sources/http.js`):
  - Checks robots.txt, cached per host per run, and evaluates the rules for the `*` user agent.
  - Waits a random delay from `delayMs` between requests to the same host.
  - Uses a normal browser User-Agent.
  - Detects bot challenges: HTTP 403, 429 or 503, or page text markers such as
    `cf-challenge`, `Just a moment`, `captcha` and `challenge-platform`. It returns a typed
    result instead of throwing.
- `ctx.browser()` starts the Playwright persistent context on `.browser-data/` lazily, only when a
  `browser`-engine source actually runs. It reuses `lib/browser.js`.
- `ctx.imap()` opens an `imapflow` session with the existing `GMAIL_USER` / `GMAIL_APP_PASSWORD`.

### 6.3 Adapter types

| Type | For | Mechanics |
|---|---|---|
| `rss` | Boards with RSS/Atom feeds | `fast-xml-parser`. `nativeId` comes from `guid`/`id`. `map` renames fields when a board repurposes them |
| `json-api` | Boards with a public JSON endpoint | `url` + `itemsPath` + a field `map` (dot paths). Pagination by param or cursor |
| `html-list` | Everything else | `engine: fetch` → `cheerio` on the HTTP body. `engine: browser` → Playwright page, then the same selector logic on `page.content()`. A `noResultsSelector` hit returns `empty`. `loggedOutSelector` returns `logged_out` |
| `imap-alerts` | Newsletters and board alert emails | See §6.4 |

**Detail step (`detail`).**
- It runs only for **new** items with no full description yet. It uses `detail.selector` on the
  item URL, with `engine: fetch` or `browser`.
- For sources with a detail step, a description under 300 characters counts as a teaser and is
  fetched again next run. It only becomes `scrapeStatus: 'ok'` once full, so the evaluator never
  judges a teaser.
- After 3 failed attempts the item is marked `scrapeStatus: 'detail_failed'` and appears in the
  health output.
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
  over All Mail, so there are **no Gmail filters to create or maintain**.
- **Tracking what's been read.** Processed Gmail message IDs (`X-GM-MSGID`) are stored in
  `state/imap-<id>.json`. A message is processed exactly once, even if it is later moved or
  read.
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
    cheerio on the HTML part.
  - Each role becomes one RawListing. The URL is the role's own link. If an item has no link, the
    URL is the issue's web or archive URL plus `#<hash(title|company)>`, which keeps URLs unique
    and clickable.
  - When an email arrives from a configured sender but yields 0 items and no known "no roles
    this week" marker, the result is `parse_zero`, which usually means the template changed.
  - There is no LLM extraction by default. Using an LLM for a single irregular newsletter would be
    an explicit per-source exception, recorded in `sources.json` with a comment.
- **Subscriptions.** Signing up for newsletters and alerts is done by Gijs. Claude does not
  subscribe or create accounts. They must be addressed to `GMAIL_USER`.

### 6.5 Shared modules (new)

| Module | Purpose |
|---|---|
| `lib/board_ids.js` | `boardPrefix(id)`, `jobId(sourceId, nativeId \| url)`, `canonicalUrl(url)` (strips the query string except params listed in `keepParams`, strips `utm_*` and the fragment), `normKey` by calling `sent_history.normalizeKey` |
| `lib/job_filters.js` | Exclusion and whitelist matching with the same rules as CLAUDE.md "Exclusion Matching Rules". **`scrape-linkedin.js` is not changed in this project.** Moving it onto this module is a follow-up with a regression test on real master CSVs |
| `lib/board_health.js` | Reads run logs and computes the status table, the `expectMinRows7d` verdicts, consecutive-failure counts and alert eligibility |
| `lib/sources/http.js` | The polite HTTP client described in §6.2 |

New dependencies: `cheerio`, `fast-xml-parser`.

## 7. Data model

### 7.1 Job IDs

The format is `<sourceId>:<nativeId>`. `nativeId` is resolved in this order:
1. The board's own stable ID (feed `guid`, API `id`, or `idPattern` on the URL).
2. The first 12 hex characters of sha1(canonical URL).

The `:` separator means an ID can never collide with LinkedIn's numeric IDs. IDs are already
URL-encoded in `/fb/:jobId/:profile/:signal` links (`web.js:660`).

### 7.2 Record in `job-data-fractional-boards.json`

The top level is `{ version, jobs: { [jobId]: record } }`, the same as the other profiles. The
record uses the existing fields plus some additive ones:

```js
{
  company, title, location, url,                   // existing
  description, descriptionScrapedAt, scrapeStatus, // existing; evaluator requires scrapeStatus 'ok'
  evaluation,                                      // written by evaluate-jobs.js, unchanged
  source:   { id, name, type },                    // new
  alsoOn:   [{ sourceId, url, seenAt }],           // new — cross-board duplicates (§7.4)
  postedAt, fetchedAt, firstSeenAt,                // new
  facts:    { engagement, daysPerWeek, dayRate, duration, remote, startDate },  // new, all optional
  detailAttempts                                   // new
}
```

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
LinkedIn.

### 7.4 Dedup

1. **Within a source, across runs.** By job ID. An item already in job-data is not new.
2. **Across sources, at fetch time.**
   - If `normKey(company, title)` (with the profile's `companyAliases`) matches a record from a
     *different* source first seen within 30 days, the new listing is **not** stored as a new
     job.
   - Instead it is appended to that record's `alsoOn`, and to the source's `_excluded.csv` with
     reason `duplicate_of:<jobId>`, so the per-board stats can count it.
   - This avoids a second evaluation and lets the card say "also on …".
3. **Across days.** `sent-history-fractional-boards.json` via the unchanged `lib/sent_history.js`
   (job ID plus the 30-day `company|title` repost key).
4. **Not across profiles (D14).**

### 7.5 First-run age cap

On a source's first successful run (no `state/<id>.first-run` marker yet), items posted more than
`firstRunMaxAgeDays` ago are written to the master CSV and job-data as already seen, but **not**
to `_new.csv`, so they never reach an email. Items without `postedAt` on a first run are treated
as old. The marker is written after the run completes.

## 8. Email, feedback and stats

### 8.1 Digest (changes to `send-daily-digest.js`)

- **Pipeline dispatch.** If the profile dir contains `sources.json`, run `fetch-boards.js`
  (passing `--slot`) instead of `scrape-linkedin.js` + `scrape-descriptions.js`. The scrape-stamp
  logic is not used, because board fetches are cheap and incremental.
- **Prefixes.** `getProfileSearchPrefixes` returns `boardPrefix(id)` for every source in
  `sources.json` (enabled or not, so late-evaluated jobs from a just-disabled board still count).
- **`resolveCsvDir`** (`lib/profile_paths.js`) also reads `csvSubdir` from `sources.json`.
- **`sendWhen`.**
  - `"any"`: skip sending when there are 0 new qualified, 0 disqualified and 0 pending jobs.
  - `"qualified"`: skip sending when there are 0 qualified.
  - A skipped send logs `DIGEST SKIPPED (sendWhen=…)` and records nothing in sent-history.
  - Profiles without `sendWhen` keep today's behaviour, including the "No new jobs today" email.
- **Card additions,** rendered only when the fields exist, so LinkedIn cards are unchanged:
  - a source line: `via Board A · also on Board B, Newsletter D`
  - a facts row
  - the posted date
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
| Qualified | verdict qualified |
| 👍 / 👎 | feedback signals on this source's jobs |
| **Unique qualified** | qualified jobs with no `alsoOn` entry, i.e. not on any other board |
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
| `parse_zero` | HTTP 200 with a non-trivial body, but 0 rows and no no-results marker | template or layout changed → alert |
| `bot_challenge` | challenge page or 403/429/503 | skip this source until the next calendar day (state flag); alert. **No retries (L4)** |
| `robots_blocked` | robots.txt disallows the path | skip permanently until the config changes; alert once |
| `logged_out` | `loggedOutSelector` matched | alert with `node fetch-boards.js --login <id>`; do not retry today |
| `error` | network error, timeout or crash in the adapter | retried at the next slot; alert after 2 consecutive |
| `skipped` | not due this slot, disabled, or blocked for today | not a failure |

### 9.2 Alerts

- **Floor check.** A source with `expectMinRows7d` that has fewer rows than that in its last
  7 days of run logs is flagged `below_floor`.
- **Alert email.** `lib/board_health.js` decides which sources are alert-worthy:
  - any `parse_zero`, `bot_challenge`, `robots_blocked`, `logged_out` or `below_floor`
  - `error` two or more times in a row
  The alert is a short plain email (`[fractional-boards] source health — …`) sent via
  `lib/mailer.js` with IMAP verification.
  - It is sent at most once per source per day, tracked in `logs/board-health-alerts.json`.
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
    emails.
- The heartbeat message gains `WITH BOARD WARNINGS: …`, next to the geo warnings, so
  `~/.monitoring` sees it.
- Check the plan-only mode: `DIGEST_PLAN_ONLY=1 bash run-digest.sh 06:00` lists `fractional-boards`.

### 9.4 Load budget

Expected load, with ~15 sources × ≤3 list pages × ~5.5 s plus ~10 new detail pages × ~5.5 s:
about **5 minutes per slot** and at most a few requests per host per slot. If the per-slot time
goes above 15 minutes, the plan-level fix is lowering `maxPages`, not parallelism.

## 10. Testing

Plain `node test_*.js` scripts, matching the repo style. No test framework and no network.
Fixtures live in `test/fixtures/boards/`.

| Test | Covers |
|---|---|
| `test_board_ids.js` | ID resolution order, canonical URL stripping, the `:` namespace, and sanitizing prefixes |
| `test_board_adapters.js` | Each adapter against fixtures: an RSS feed, a JSON response, an HTML list, an HTML no-results page (→ `empty`), a changed-layout page (→ `parse_zero`), a challenge page (→ `bot_challenge`), a logged-out page, and one `.eml` per newsletter parser |
| `test_fetch_boards.js` | The pipeline with mocked adapters: per-source isolation (one throws, the others complete), cross-board `alsoOn` dedup, the first-run age cap, teaser → detail retry → `detail_failed`, slot selection, `--dry-run` writes nothing, and the exit codes |
| `test_board_health.js` | The status table, the floor check, consecutive-error counting, and once-per-day alert suppression |
| `test_digest_sendwhen.js` | `sendWhen` any/qualified/absent (the absent case keeps the legacy email), prefixes from `sources.json`, and card rendering with and without source/facts fields |
| `test_run_digest_slots.sh` (extend) | `fractional-boards` is listed at both slots; a non-4 exit counts as a board warning, not a failure |

Existing tests must still pass. That confirms the LinkedIn path is unchanged:
`test_profile_paths.js`, `test_digest_delivery.js`, `test_scrape_stamp.js`,
`test_run_digest_slots.sh`, `test_feedback_triage.js`, `test_review_render.js`.

**End-to-end check before enabling the schedule:**
1. `node fetch-boards.js --profile fractional-boards --source <first real board> --dry-run`
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
