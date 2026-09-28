# Fractional Boards Pipeline — Implementation Plan

**Date:** 2026-09-28
**Spec:** `docs/specs/2026-09-28-fractional-boards-pipeline-design.md`
**Review:** `docs/specs/2026-09-28-fractional-boards-pipeline-review.md` (findings R1–R13, M1–M10)
**Where:** `~/code/Jobs` (D1, D2), on a branch; one commit per task.

This plan assumes the review's **blocking** findings R1–R5 are accepted. Tasks that depend on a
review finding cite it. If a finding is rejected, drop the part of the task that cites it.

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

The spec was written from reading the code. Before building on it, check these points and record
any differences at the top of the spec:

| # | Check | Where |
|---|---|---|
| V1 | `sanitizeForFilename('board a')` → `Board-A` | wherever it's defined |
| V2 | "new" = URLs in today's `*_new.csv` matched by prefix; can it take a date window (R8)? | `send-daily-digest.js:~680` |
| V3 | `sent_history.normalizeKey` signature, and that the company string is caller-supplied (R6) | `lib/sent_history.js` |
| V4 | Which date `evaluate-jobs.js --days 7` filters on (M10); that it requires `scrapeStatus === 'ok'` | `evaluate-jobs.js` |
| V5 | What `--run-pipeline` does when the scrape step fails (R12) | `send-daily-digest.js` |
| V6 | The card HTML escape helper (R13) | `send-daily-digest.js` |
| V7 | The atomic tmp+rename writer: is it a shared function or copied inline? | `scrape-descriptions.js` |
| V8 | `run-digest.sh` exit-code handling and heartbeat format; the `DIGEST_PLAN_ONLY` path | `run-digest.sh`, `run-digest-guard.sh` |
| V9 | `lib/browser.js` API for starting the persistent context | `lib/browser.js` |
| V10 | The "Exclusion Matching Rules" section and how `scrape-linkedin.js` implements it | `CLAUDE.md`, `scrape-linkedin.js` |

**Done when** every row is confirmed, or the spec or plan has been changed to match the code.

## Phase 1: framework

### T1. Dependencies and profile skeleton
- `npm i cheerio fast-xml-parser`.
- Create `profiles/fractional-boards/`:
  - `sources.json` with `"sources": []`, `sendWhen: "any"`, `csvSubdir: "csv"`
  - `filters.json` with the three `_placeholder` criteria (§5.3)
  - `csv/` and `state/`, each with a `.gitkeep`
- **Done when** the regression set passes.

### T2. `lib/board_ids.js` and `test_board_ids.js`
- `boardPrefix(id)`
- `canonicalUrl(url, {keepParams|stripParams})`, which strips tracking parameters by default (R2)
- `jobId(sourceId, {nativeId, url})`: the resolution order from §7.1, plus the synthetic-fragment
  rule for link-less newsletter items (R1)
- `normKey(company, title, aliases)`, which delegates to `sent_history.normalizeKey`
- `normCompany()`, which maps anonymous companies to `null` (R6)
- Tests:
  - the resolution order
  - the `:` namespace
  - prefixes: `board-a` → `Board-A`, and two ids that sanitise to the same prefix are detected (M7)
  - tracking-parameter stripping, and that identifying parameters are kept
  - two link-less items from one issue get **different** IDs (R1)

### T3. `lib/job_filters.js` and `test_job_filters.js`
- Implement the CLAUDE.md exclusion rules: `excludeCompanies`, `excludeTitleKeywords`,
  `includeTitleKeywords`, `excludeCountries`, `companyAliases`. Each returns `{excluded, reason}`.
- Test against the rule examples in CLAUDE.md and against 20 or more rows from a real LinkedIn
  `_excluded.csv`, so the new module and `scrape-linkedin.js` agree. `scrape-linkedin.js` itself
  is **not** changed (§12).

### T4. `lib/sources/http.js` and `test_board_http.js`
- `get(url)` → `{ status: 'ok'|'bot_challenge'|'robots_blocked'|'error', httpStatus, body, note }`.
- robots.txt cached per host, evaluated for `*`, with the RFC 9309 behaviour for 4xx and 5xx (M1).
- A random per-host delay from `delayMs`, a timeout, and a browser User-Agent.
- Challenge detection: status 403, 429 or 503, or the text markers.
- The HTTP transport can be injected, so the test uses an in-memory fake with no network:
  - robots allow and disallow
  - robots 404 and 503
  - each challenge marker
  - a timeout becomes `error`
  - the delay is honoured, using an injected clock

### T5. Adapter contract and `rss`
- `lib/sources/index.js`:
  - the registry
  - `validateSources(json)`: unknown keys are errors, `_*` keys are allowed (R5), `enabled`
    defaults to true (M6), ids must be unique, and prefixes must not collide
- `lib/sources/rss.js`: RSS 2.0 and Atom, `guid`/`id` → `nativeId`, `map`, and 0 items → `empty`.
- Fixtures: `test/fixtures/boards/rss-feed.xml`, `atom-feed.xml` and `rss-empty.xml`.
- Starts `test_board_adapters.js`.

### T6. `html-list` (`engine: fetch`)
- The shared parsing function `parseListHtml(html, cfg)` is used by both engines. It supports:
  - the `sel@attr` selector syntax
  - `idPattern`
  - facts selectors
  - `noResultsSelector` → `empty`
  - `loggedOutSelector` → `logged_out`
  - `parse_zero` judged on page 1 only (M2)
- Pagination stops early on a page with only known IDs (M2).
- Fixtures: a list page, a no-results page, a changed layout, a challenge page and a logged-out
  page.

### T7. `json-api`
- `itemsPath`, dot-path `map`, and pagination by parameter or cursor with the same early stop.
- Fixtures: a page-1 and page-2 response.

### T8. `html-list` (`engine: browser`) and the `--login` helper
- Lazy `ctx.browser()` via `lib/browser.js` (V9), and `page.content()` → `parseListHtml`.
- A held `SingletonLock` → `error: browser profile in use` (M5).
- `--login <id>` opens a headed page at the source's `loginUrl` and waits for the window to close.
- **Test:** unit-test only the lock detection and the call into `parseListHtml`; the browser is
  mocked.

### T9. `imap-alerts` and the parser framework
- `X-GM-RAW` search with the `from:`, `newer_than:7d` and `subject:` terms.
- `state/imap-<id>.json` records processed `X-GM-MSGID`s and is pruned after 30 days (M3).
- After a successful ingest, add the label with `X-GM-LABELS`. If `archiveAfterIngest` is set,
  also remove `\Inbox`.
- `lib/sources/parsers/<name>.js` is a function from HTML to `{ items, noRolesMarker }`.
- 0 items with no marker → `parse_zero`.
- `postedAt` is the email's `Date` header (M4).
- A link-less item gets the synthetic URL and ID (R1).
- **Test:** a fake IMAP client, plus one `.eml` fixture with a sample parser.

### T10. `detail` step
- It runs only for records where `scrapeStatus` is neither `ok`, `seen_first_run` nor
  `detail_failed`.
- A description under 300 characters is a teaser. `detailAttempts` counts up, and on the third
  failure the adapter tries the `alsoOn` URLs (R10) before marking `detail_failed`.
- A facts line is prepended to `description` (§6.3).
- `detail: false` skips the step and takes the description from `list()`.

### T11. `fetch-boards.js`: orchestration and `test_fetch_boards.js`
Per source, in config order, each inside its own try/catch:
1. **Slot and state gates:**
   - due in this slot?
   - `bot_challenge` flag set for today?
   - `robots_blocked`?
   - slot stamp already `ok` or `empty` (R7)?
2. `list()`, then normalise, then check for ID collisions (R2).
3. `job_filters`, which writes excluded items to `_excluded.csv` with a reason.
4. Cross-board dedup by `normKey`, within 30 days and only when company is non-null (R6). A hit is
   added to `alsoOn` and written to `_excluded` as `duplicate_of:<jobId>`.
5. First-run cap. Old items go to master and job-data with `scrapeStatus: 'seen_first_run'`
   (R4).
6. For each new item, in this **write order** (R3):
   1. append to master CSV and today's `_new.csv`
   2. write to job-data (atomic)
   3. run `detail`
   4. write to job-data (atomic)
7. Write the first-run marker, the slot stamp and the run-log entry.

The mocked-adapter tests cover:
- isolation (one adapter throws)
- `alsoOn`
- the first-run cap and `seen_first_run`
- teaser → retry → `detail_failed`
- slot selection and the slot-stamp skip
- `--dry-run` writes nothing (checked by a directory snapshot before and after)
- a crash between the CSV write and the job-data write still leaves the job "new" (R3)
- exit codes: 0 with failing sources, and 1 for invalid config
- the placeholder warning is logged

### T12. `lib/board_health.js` and `test_board_health.js`
- Status table from `logs/board-runs/*.json`, pruned after 90 days.
- `below_floor` applies only once 7 days of logs exist (R9).
- Counts consecutive errors (an alert at 2 or more).
- At most one alert per source per day, tracked in `logs/board-health-alerts.json`. Optionally
  backs off to weekly (M9).
- `renderAlertEmail()` uses neutral wording; the test asserts that "spam" and "dropped" are absent.
- `footerSummary()` for the digest.

### T13. Digest changes in `send-daily-digest.js`, `lib/profile_paths.js`, and `test_digest_sendwhen.js`
- `resolveCsvDir` reads `csvSubdir` from `sources.json`.
- `getProfileSearchPrefixes` includes every source's `boardPrefix` (enabled or not).
- Pipeline dispatch: with `sources.json`, run `fetch-boards.js --slot`. On exit 1, carry on with
  the data on disk and note it in the footer (R12).
- The "new" window for board profiles follows the R8 decision.
- `sendWhen`: `any`, `qualified`, or absent (legacy behaviour). A skipped send logs
  `DIGEST SKIPPED (sendWhen=…)` and records nothing in sent-history.
- Card additions: the source and `alsoOn` line, the facts row and the posted date, all escaped
  (R13). None of them render when the fields are missing.
- Health step after the digest (T12). Its errors are logged and never change the exit code.
- Sent-history for records with a null company uses `anon:<jobId>` (R6).
- **Tests:**
  - the three `sendWhen` modes
  - prefixes
  - a LinkedIn-shaped card renders byte-identical to before (golden file)
  - a board card with a `<script>` title comes out escaped

### T14. `run-digest.sh` and extending `test_run_digest_slots.sh`
- Add `fractional-boards` last at both slots.
- Exit 4 → `eval_outages++`. Any other non-zero exit → `board_warnings++` with detail.
- If R8(b) was chosen instead of (a), exit 5 → retry this profile only.
- Heartbeat: `WITH BOARD WARNINGS: …`.
- **Test:** plan-only mode lists the profile at both slots, and a fake profile exiting 1 counts
  as a warning, not a failure.

### T15. Feedback registration and stats
- Register `fractional-boards` in:
  - `web.js` `PROFILES`
  - `lib/feedback_triage.js` `PROFILES`
  - `send-review-digest.js` `sealSession`
- A `/boards` page in `web.js` with the §8.3 table for 7, 30 and 90 days. Qualified credit is
  shared with `alsoOn` sources (R11). CSV files are matched by exact prefix (M7).
- One line per board in the weekly review email.
- **Test:** the stats function against a fixture directory (CSVs, job-data, feedback, run logs).

### T16. First real board, end to end
Choose a board: the first public RSS or clean-HTML board on the Cowork shortlist, or any public
fractional board if the shortlist isn't ready. Then:
1. Add its config entry.
2. `node fetch-boards.js --profile fractional-boards --source <id> --dry-run`, and check the rows
   and facts by hand.
3. Save one real response as a fixture and add an adapter test for it.
4. `node send-daily-digest.js --run-pipeline --via-cli --profile fractional-boards --dry-run`
5. Do one real send. **Done when** IMAP reports `delivery: inbox`.

### T17. Documentation
- A "Fractional boards" section in `CLAUDE.md` covering:
  - the config keys
  - adding a board (the Phase 2 checklist)
  - `--dry-run` and `--login`
  - the status table and what each alert means
  - a pointer to the spec
- Rename the "JobS Project" heading to `~/code/Jobs`.
- Fold the accepted review findings into the spec, and delete the review file.

## Phase 2: boards (one task per board or newsletter)

For each item on the Cowork shortlist:
1. config entry
2. `--dry-run`
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
