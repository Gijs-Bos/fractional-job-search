# Fractional Boards Pipeline — Spec Review

**Date:** 2026-09-28
**Reviews:** `docs/specs/2026-09-28-fractional-boards-pipeline-design.md`
**Scope:** the written spec alone. The `~/code/Jobs` code was not available for this review, so
anything that depends on how existing functions behave is marked **(verify)**.

Each finding has a proposed fix. **Blocking** findings can lose jobs without anyone noticing, or
break the spec's own rules. They should be settled before the plan is built. **Should-fix**
findings are better fixed now than after the first real boards are live. **Minor** findings can
be handled while building.

Once each finding is accepted or rejected, fold the accepted ones into the spec and delete this
file.

## Blocking

### R1. Link-less newsletter items all get the same job ID
§6.4 gives a newsletter item with no link the URL `<issue URL>#<hash(title|company)>`. §6.5
`canonicalUrl` strips the fragment, and §7.1 falls back to sha1(canonical URL). So every link-less
role in one issue gets the same `nativeId`. The first one is stored and the rest are dropped by
"within a source, by job ID" dedup (§7.4.1), silently.
**Fix:** a parser that builds a synthetic URL also sets `nativeId = hash(title|company)`.
`jobId()` never canonicalises a URL whose fragment the adapter marked as identifying. Add this
case to `test_board_ids.js`.

### R2. Stripping the whole query string by default merges distinct jobs
§5.2 strips the whole query unless it is listed in `keepParams`. Many boards identify postings by
query (`/job?id=123`, `/vacancy.php?ref=…`). Without `keepParams`, every posting on such a board
hashes to the same ID, and after the first one the board goes silent while reporting `ok`. The
same stripping can also break the link in the email if `record.url` is the canonical form.
**Fix:**
- Flip the default: strip only known tracking parameters (`utm_*`, `ref`, `source`, `fbclid`,
  `gclid`, `mc_cid`, `mc_eid`) plus the fragment, and keep the rest. Keep `keepParams` for the
  opposite case, as `stripParams`.
- Use the canonical URL only for hashing. Store the original URL (minus tracking parameters) in
  `record.url` and the CSVs.
- Detect collisions: if two different raw URLs in one `list()` result give the same job ID, log
  it and set the run-log note to `id_collision`. That makes it an alert.

### R3. Crash between the job-data write and the CSV write loses the job
§6.1 saves job-data after each detail fetch. §4 writes the CSVs afterwards. If the process dies in
between, the item is in job-data and so is "not new" next run (§7.4.1), but it is not in today's
`_new.csv`, so the digest never sees it (L2).
**Fix:** state the write order. Append each new item to master and today's `_new.csv` **before**
it is first written to job-data. A duplicate CSV row after a crash is harmless, because the
digest filters through sent-history. Add a test that crashes between the two writes.

### R4. First-run "already seen" items need their own status
§7.5 puts old first-run items into job-data. Nothing says what `scrapeStatus` they get. If it is
empty or pending, the detail step (§6.3, "new items with no full description") will fetch every
one of them. That can be hundreds of requests on a board's first run, which breaks D7 and the
§9.4 budget. After 3 tries they would also show up as `detail_failed` in health.
**Fix:** store them with `scrapeStatus: 'seen_first_run'` and no description. The detail step,
the evaluator and the health counts all skip that status.

### R5. `sources.json` cannot hold the comment §6.4 requires
§6.4 says an LLM exception is "recorded in `sources.json` with a comment". JSON has no comments,
and §5.2 makes unknown keys an error.
**Fix:** `validate()` ignores any key that starts with `_`. This matches `_placeholder` in
`filters.json`. Use `"_comment": "…"`.

## Should-fix

### R6. Missing company breaks both kinds of dedup
Fractional boards and newsletters often hide the company ("Confidential", "Stealth SaaS scale-up",
"via Acme Search") or leave it out. With an empty or placeholder company:
- The §7.4.2 cross-board dedup merges unrelated "Fractional CRO" roles.
- `sent_history`'s 30-day `company|title` repost key suppresses every later role with the same
  generic title.

**Fix:** in `lib/board_ids.js`, normalise anonymous companies (empty, `confidential`, `stealth*`,
`undisclosed`, `anonymous`, `n/a`, and the board's own name) to `null`. When company is `null`:
- skip cross-board dedup
- give sent-history `anon:<jobId>` as the company, which turns off repost matching for that job
  without changing `lib/sent_history.js`

**(verify)** that `sent_history` accepts a caller-supplied company string.

### R7. Guard re-runs would fetch every board every 15 minutes
§8.1 drops the scrape stamp for boards. But when a **LinkedIn** profile fails, the guard re-runs
every profile every 15 minutes (L10), and that includes `fractional-boards`. Each board would then
be fetched up to about 30 times a slot. That is the load D7 rules out, and it is the kind of load
that gets you blocked (L4).
**Fix:** save a per-source slot stamp, `state/<id>.last-ok = { date, slot }`. `fetch-boards.js`
skips a source (`skipped`, note `already fetched this slot`) that already finished `ok` or
`empty` in the same date and slot. Evaluation and the digest still run on each re-run.

### R8. A failed 14:00 send is never retried
Because board failures are only warnings (§9.3), the guard does not retry a failed
`deliverDigest` for this profile. A failed 06:00 send rolls into 14:00, since today's `_new.csv`
still lists the jobs and sent-history does not. A failed 14:00 send does not roll anywhere: the
next day's "new" set is built from the next day's CSV.
**Fix:** either:
- (a) build "new" from the last 2 days of `_new.csv` for board profiles, with sent-history
  stopping repeats; or
- (b) give delivery failure its own exit code (e.g. `5`), which `run-digest.sh` treats as a
  retry of **this profile only**.

(a) is simpler and needs no guard changes. **(verify)** that `send-daily-digest.js:680` can take
a date window.

### R9. A new source is always `below_floor` in its first week
§9.2 checks `expectMinRows7d` against the last 7 days of run logs. A source added yesterday has 1
day of logs and will alert every day for a week.
**Fix:** apply the floor only once the source has run logs covering 7 days.

### R10. Duplicates should back up a failed detail fetch
When a record reaches `detail_failed` but has `alsoOn` entries, another board may have the full
description.
**Fix:** before marking `detail_failed`, try the detail step once against each `alsoOn` URL,
using that source's `detail` config.

### R11. Per-board stats give all the credit to whichever board is listed first
Sources run in config order (§6.1), so a shared job always belongs to the board listed first. The
later board gets no "Qualified" credit for it. Stats built this way would push toward dropping
boards because of where they sit in the file, not because of their quality.
**Fix:** in §8.3, count a qualified job for its owning source **and** every `alsoOn` source. Keep
"Unique qualified" as defined. It is the column that decides whether to drop a board.

### R12. Fetch failures should not stop the digest
§6.1 exits 1 on invalid config. §8.1 does not say whether `send-daily-digest.js` then carries on
with evaluation and the digest. For LinkedIn a scrape failure probably aborts. **(verify)**
**Fix:** if `fetch-boards.js` exits 1, log it, still evaluate and send what is already on disk, and
put "fetch failed: <reason>" in the footer. If `sources.json` itself cannot be parsed, the
prefixes are unknown, so exit non-zero. That counts as a board warning under §9.3.

### R13. Escape new card fields
Titles, companies and facts now come from scraped HTML and newsletters, not from LinkedIn. The new
card fields (source line, facts row, posted date) must go through the same HTML escaping as the
existing fields. **(verify)** the digest's escape helper and use it. Add a test with a `<script>`
title.

## Minor

- **M1. Robots edge cases.** Follow RFC 9309: a 4xx on `robots.txt` means allow; a 5xx or a
  network error means disallow for this run, reported as status `error`, not `robots_blocked`.
- **M2. Stop pagination early.** Stop `html-list` and `json-api` paging when a page has only known
  IDs. This usually cuts page requests to 1 per slot. Judge `parse_zero` and `empty` on page 1
  only. A page 2+ with 0 cards is just the end of the results.
- **M3. Prune IMAP state.** Remove message IDs older than 30 days from `state/imap-<id>.json`.
  Note that `newer_than:7d` means mail that arrives during a pipeline outage of more than 7 days
  is skipped.
- **M4. Dates.** Give `postedAt` for `imap-alerts` as the email's `Date` header. Relative dates
  ("3 days ago", "vor 3 Tagen", "3 dagen geleden") need one small parser for EN/DE/NL, or else
  `postedAt: null`. The spec should say which.
- **M5. Shared browser profile.** Running `--login` while a scheduled run is using `.browser-data/`
  hits Chromium's profile lock. Detect `SingletonLock` and report `error` with the note `browser
  profile in use`.
- **M6. `enabled` default.** State that `enabled` defaults to `true`.
- **M7. Prefix matching.** Per-board stats must match `_<prefix>_new.csv` exactly, not with
  `startsWith`. Otherwise source `board` also counts `board-a`'s files. `validate()` should also
  reject two ids that give the same `boardPrefix`.
- **M8. Example config.** Mark the `marketplace-c` entry in §5.2 as abbreviated. As written it has
  no `listUrls` or `selectors`, and `validate()` would reject it.
- **M9. Repeated alerts.** A board that stays `bot_challenge` alerts every day forever. Consider
  daily alerts for 3 days, then weekly.
- **M10. Evaluator window.** **(verify)** which date `evaluate-jobs.js --days 7` filters on. If it
  is the CSV date, a teaser that reaches `ok` after several slots is still inside the window.
  If it is `descriptionScrapedAt`, it also is. Only `firstSeenAt` combined with a long
  `bot_challenge` pause could push a job outside the window.
