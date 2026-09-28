# Fractional Boards Pipeline — Implementation Plan

**Date:** 2026-09-28
**Spec:** `docs/specs/2026-09-28-fractional-boards-pipeline-design.md` (review findings folded in;
first scan limited to 15 days, D17)
**Where:** `~/code/Jobs` (D1, D2), on a branch; one commit per task.

Section references (§) are to the spec.

Each task follows the same loop:
1. Write the test with its fixtures.
2. Watch it fail.
3. Implement.
4. Run the new test and the **regression set**.
5. Commit.

**Regression set** (these must stay green after every task; they prove the LinkedIn path is
unchanged):

```
node test_profile_paths.js && node test_digest_delivery.js && node test_scrape_stamp.js \
  && bash test_run_digest_slots.sh && node test_feedback_triage.js && node test_review_render.js
```

## Phase 0: check the spec's assumptions against the code (no code changes)

The spec was written from reading the code. Before building on it, check these points. If one
differs, change the spec (§13) first, then this plan.

| # | Check | Where |
|---|---|---|
| V1 | `sanitizeForFilename('board a')` → `Board-A` | wherever it's defined |
| V2 | "New" = URLs in today's `*_new.csv` matched by prefix, and it can take a two-day window for board profiles (§13 A1) | `send-daily-digest.js:~680` |
| V3 | `sent_history.normalizeKey` signature, and that the company string is caller-supplied (§13 A2) | `lib/sent_history.js` |
| V4 | Which date `evaluate-jobs.js --days 7` filters on (§13 A3), and that it requires `scrapeStatus === 'ok'` and so skips `seen_first_run` | `evaluate-jobs.js` |
| V5 | What `--run-pipeline` does today when the scrape step fails (§13 A4) | `send-daily-digest.js` |
| V6 | The card HTML escape helper (§13 A5) | `send-daily-digest.js` |
| V7 | The atomic tmp+rename writer: is it a shared function or copied inline? | `scrape-descriptions.js` |
| V8 | `run-digest.sh` exit-code handling and heartbeat format; the `DIGEST_PLAN_ONLY` path | `run-digest.sh`, `run-digest-guard.sh` |
| V9 | `lib/browser.js` API for starting the persistent context | `lib/browser.js` |
| V10 | The "Exclusion Matching Rules" section and how `scrape-linkedin.js` implements it | `CLAUDE.md`, `scrape-linkedin.js` |

**Done when** every row is confirmed, or the spec and plan have been changed to match the code.

## Phase 1: framework

### T1. Dependencies and profile skeleton
- `npm i cheerio fast-xml-parser`.
- Create `profiles/fractional-boards/`:
  - `sources.json` with `"sources": []`, `sendWhen: "any"`, `csvSubdir: "csv"` and
    `defaults.firstRunMaxAgeDays: 15`
  - `filters.json` with the three `_placeholder` criteria (§5.3)
  - `csv/` and `state/`, each with a `.gitkeep`
- **Done when** the regression set passes.

### T2. `lib/board_ids.js` and `test_board_ids.js` (§6.5, §7.1)
- `boardPrefix(id)`
- `canonicalUrl(url, stripParams)` for hashing, and `displayUrl(url)` for storage. Both strip
  tracking parameters; only `canonicalUrl` also removes `stripParams` and the fragment.
- `jobId(sourceId, {nativeId, url})`, following the resolution order in §7.1
- `normKey(company, title, aliases)`, which delegates to `sent_history.normalizeKey`
- `normCompany(company, sourceName)`, which maps anonymous companies to `null` (§7.4)
- Tests are as listed for `test_board_ids.js` in §10.

### T3. `lib/job_filters.js` and `test_job_filters.js`
- Implement the CLAUDE.md exclusion rules: `excludeCompanies`, `excludeTitleKeywords`,
  `includeTitleKeywords`, `excludeCountries`, `companyAliases`. Each returns `{excluded, reason}`.
- Test against the rule examples in CLAUDE.md and against 20 or more rows from a real LinkedIn
  `_excluded.csv`, so the new module and `scrape-linkedin.js` agree. `scrape-linkedin.js` itself
  is **not** changed (§12).

### T4. `lib/sources/http.js`, `lib/sources/dates.js` and their tests (§6.2, §6.3)
- `get(url)` → `{ status: 'ok'|'bot_challenge'|'robots_blocked'|'error', httpStatus, body, note }`.
- robots.txt cached per host, evaluated for `*`, and following RFC 9309 for 4xx and 5xx.
- A random per-host delay from `delayMs`, a timeout, and a browser User-Agent.
- Challenge detection: status 403, 429 or 503, or the text markers.
- The HTTP transport and clock can be injected, so the test needs no network.
- `dates.js`: absolute dates, plus EN/DE/NL relative dates; anything else → `null`.

### T5. Adapter contract, config validation and `rss`
- `lib/sources/index.js`:
  - the registry
  - `validateSources(json)`:
    - unknown keys are errors, `_*` keys are ignored
    - `enabled` defaults to true
    - ids must be unique, and prefixes must not collide
    - `firstRunMaxAgeDays` above 15 is rejected (§5.2)
- `lib/sources/rss.js`: RSS 2.0 and Atom, `guid`/`id` → `nativeId`, `map`, and 0 items → `empty`.
- Fixtures: `test/fixtures/boards/rss-feed.xml`, `atom-feed.xml` and `rss-empty.xml`.
- Starts `test_board_adapters.js`.

### T6. `html-list` (`engine: fetch`)
- The shared parsing function `parseListHtml(html, cfg)` is used by both engines. It supports:
  - the `css@attribute` selector syntax
  - `idPattern`
  - facts selectors
  - `noResultsSelector` → `empty`
  - `loggedOutSelector` → `logged_out`
  - `parse_zero` judged on page 1 only
- Pagination (§6.3) stops at `maxPages`, at a page with 0 items, at a page of known IDs only, or,
  on a first run, at a page whose items are all older than `firstRunMaxAgeDays`. Known and first-run
  state come from `ctx.isKnown()` and `ctx.firstRun`.
- Fixtures: a list page, a no-results page, a changed layout, a challenge page, a logged-out page,
  and an empty page 2.

### T7. `json-api`
- `itemsPath`, dot-path `map`, and pagination by parameter or cursor with the same stop rules.
- Fixtures: a page-1 and page-2 response.

### T8. `html-list` (`engine: browser`) and the `--login` helper
- Lazy `ctx.browser()` via `lib/browser.js` (V9), and `page.content()` → `parseListHtml`.
- A held `SingletonLock` → `error`, note `browser profile in use`.
- `--login <id>` opens a headed page at the source's `loginUrl` and waits for the window to close.
- **Test:** unit-test only the lock detection and the call into `parseListHtml`; the browser is
  mocked.

### T9. `imap-alerts` and the parser framework (§6.4)
- `X-GM-RAW` search with the `from:`, `newer_than:7d` and `subject:` terms.
- `state/imap-<id>.json` records processed `X-GM-MSGID`s and is pruned after 30 days.
- After a successful ingest, add the label with `X-GM-LABELS`. If `archiveAfterIngest` is set,
  also remove `\Inbox`.
- `lib/sources/parsers/<name>.js` is a function from HTML to `{ items, noRolesMarker }`.
- 0 items with no marker → `parse_zero`.
- `postedAt` is the email's `Date` header.
- A link-less item gets the synthetic URL and `nativeId = hash(title|company)`.
- **Test:** a fake IMAP client, plus one `.eml` fixture with a sample parser.

### T10. `detail` step (§6.3)
- It runs only for records where `scrapeStatus` is neither `ok`, `seen_first_run` nor
  `detail_failed`.
- A description under 300 characters is a teaser. `detailAttempts` counts up, and after the
  third failure the step tries each `alsoOn` URL once before marking `detail_failed`.
- A facts line is prepended to `description`.
- `detail: false` skips the step and takes the description from `list()`.

### T11. `fetch-boards.js`: orchestration and `test_fetch_boards.js` (§6.1, §7)
Per source, in config order, each inside its own try/catch:
1. **Gates:**
   - due in this slot?
   - `bot_challenge` flag set for today?
   - `robots_blocked`?
   - `state/<id>.last-ok` already records `ok` or `empty` for this date and slot?
2. `list()`, then normalise, then the ID-collision check (§7.1).
3. `job_filters`, which writes excluded items to `_excluded.csv` with a reason.
4. Cross-board dedup (§7.4.2): only for a non-null `normCompany`, within 30 days, with the
   `seen_first_run` exception. A hit is added to `alsoOn` and written to `_excluded` as
   `duplicate_of:<jobId>`.
5. **First-run cap (§7.5, D17).** With no `state/<id>.first-run` marker, items posted more than
   15 days ago, or with no `postedAt`, go to master and job-data with
   `scrapeStatus: 'seen_first_run'`, and not to `_new.csv`.
6. For each new item, in this **write order** (§6.1):
   1. append to master CSV and today's `_new.csv`
   2. write to job-data (atomic)
   3. run `detail`
   4. write to job-data (atomic)
7. Write the first-run marker (only if the status is `ok` or `empty`), the slot stamp and the
   run-log entry.
8. Log the placeholder-criteria warning.

Tests are as listed for `test_fetch_boards.js` in §10. The first-run cap cases must use
`ctx.now` so the 15-day boundary is exact:
- 16 days old → held back
- 15 days old → new
- no date → held back
- a failed first run writes no marker

### T12. `lib/board_health.js` and `test_board_health.js` (§9)
- Status table from `logs/board-runs/*.json`, pruned after 90 days.
- `below_floor` applies only once 7 days of logs exist.
- `id_collision` notes are alert-worthy.
- Counts consecutive errors (an alert at 2 or more).
- At most one alert per source per day. A persistent condition alerts daily for 3 days, then
  weekly. Tracked in `logs/board-health-alerts.json`.
- `renderAlertEmail()` uses neutral wording; the test asserts that "spam" and "dropped" are absent.
- `footerSummary()` for the digest.

### T13. Digest changes in `send-daily-digest.js`, `lib/profile_paths.js`, and `test_digest_sendwhen.js` (§8.1)
- `resolveCsvDir` reads `csvSubdir` from `sources.json`.
- `getProfileSearchPrefixes` includes every source's `boardPrefix` (enabled or not), with exact
  file matching.
- Pipeline dispatch: with `sources.json`, run `fetch-boards.js --slot`. On exit 1, carry on with
  the data on disk and note it in the footer.
- Board profiles use a "new" window of today plus yesterday. LinkedIn profiles are unchanged.
- `sendWhen`: `any`, `qualified`, or absent (legacy behaviour). A skipped send logs
  `DIGEST SKIPPED (sendWhen=…)` and records nothing in sent-history.
- Card additions: the source and `alsoOn` line, the facts row and the posted date, all escaped.
  None of them render when the fields are missing.
- Sent-history for records with a null `normCompany` uses `anon:<jobId>`.
- Health step after the digest (T12). Its errors are logged and never change the exit code.
- Tests are as listed for `test_digest_sendwhen.js` in §10.

### T14. `run-digest.sh` and extending `test_run_digest_slots.sh` (§9.3)
- Add `fractional-boards` last at both slots.
- Exit 4 → `eval_outages++`. Any other non-zero exit → `board_warnings++` with detail.
- Heartbeat: `WITH BOARD WARNINGS: …`.
- **Test:** plan-only mode lists the profile at both slots, and a fake profile exiting 1 counts
  as a warning, not a failure.

### T15. Feedback registration and stats (§8.2, §8.3)
- Register `fractional-boards` in:
  - `web.js` `PROFILES`
  - `lib/feedback_triage.js` `PROFILES`
  - `send-review-digest.js` `sealSession`
- A `/boards` page in `web.js` with the §8.3 table for 7, 30 and 90 days. Qualified credit is
  shared with `alsoOn` sources. CSV files are matched by exact prefix.
- One line per board in the weekly review email.
- **Test:** the stats function against a fixture directory (CSVs, job-data, feedback, run logs).

### T16. First real board, end to end
Choose a board: the first public RSS or clean-HTML board on the Cowork shortlist, or any public
fractional board if the shortlist isn't ready. Then:
1. Add its config entry.
2. `node fetch-boards.js --profile fractional-boards --source <id> --dry-run`. Check the rows and
   facts by hand, and check that nothing posted more than 15 days ago, or undated, would be new.
3. Save one real response as a fixture and add an adapter test for it.
4. `node send-daily-digest.js --run-pipeline --via-cli --profile fractional-boards --dry-run`
5. Do one real send. **Done when** IMAP reports `delivery: inbox`.

### T17. Documentation
- A "Fractional boards" section in `CLAUDE.md` covering:
  - the config keys
  - adding a board (the Phase 2 checklist)
  - `--dry-run` and `--login`
  - the 15-day first-scan rule
  - the status table and what each alert means
  - a pointer to the spec
- Rename the "JobS Project" heading to `~/code/Jobs`.

## Phase 2: boards (one task per board or newsletter)

For each item on the Cowork shortlist:
1. config entry
2. `--dry-run`, including the 15-day check
3. fixture
4. adapter test
5. `enabled: true`
6. commit

Newsletters also need Gijs to subscribe with `GMAIL_USER`, a parser in `lib/sources/parsers/`
with an `.eml` fixture, and the Gmail label. Claude creates the label only if IMAP will not
auto-create it, and only with Gijs's approval for that label.

## Phase 3: tuning (Gijs)

- Replace the placeholder criteria. The criteria hash changes, so jobs are re-evaluated
  automatically.
- After 2–4 weeks, set `sendWhen: "qualified"` if the rubric is stable.
- Use `/boards` to drop weak boards (`enabled: false`) and decide whether `includeTitleKeywords`
  is needed.

## Order and dependencies

```
Phase 0 ─► T1 ─► T2 ─► T3
                 T2 ─► T4 ─► T5 ─► T6 ─► T7
                                  T6 ─► T8
                       T5 ─► T9
                 T5..T9 ─► T10 ─► T11 ─► T12 ─► T13 ─► T14 ─► T15 ─► T16 ─► T17
```

T3, T7, T8 and T9 are independent of each other once T5 is done. T11 needs at least one adapter
(T5) and can be built against mocks before T6–T9 land.
