# Fairway Log Implementation Plan

**Goal:** Record personal golf rounds on an iPhone, import historical rounds, and turn both sources into tested data models and a useful public Tableau dashboard.

**Architecture:** The React PWA saves locally before synchronizing with Supabase. Immutable source events and private Excel uploads feed Python ETL, dbt models, statistical analysis, and a privacy-filtered Google Sheets export. GitHub Actions runs the pipeline; Tableau Public refreshes its extract separately.

**Stack:** TypeScript, React, Vite, Dexie, Supabase, Python, Pydantic, psycopg, openpyxl, dbt-postgres, scikit-learn, GitHub Actions, Cloudflare Pages, and Tableau Public.

**Design references:** [Architecture](architecture.md), [data notes](data-notes.md), [spreadsheet import design](superpowers/specs/2026-09-15-spreadsheet-import-design.md), and [roadmap](roadmap.md).

**Starting point:** The repository contains documentation only. This plan adds the implementation in the order agreed below. No application, hosted database, template, or workflow is assumed to exist already.

## Working rules

- Complete milestones in order. Each produces something independently testable; stop only at a real external dependency, such as a missing account login.
- Use a feature branch for implementation and small commits describing the change plainly: `Add course setup`, `Save rounds offline`, or `Import old rounds`.
- Keep the existing author identity and privacy email. Do not add attribution trailers or generated-by statements.
- Keep the short README style. Put technical detail and operating instructions in `docs/`.
- Write behavior tests before implementation where failure would lose, duplicate, expose, or misreport data. Do not generate tests just to increase a count.
- Use only free plans and default hosted domains. Do not buy a domain or enable a paid plan.
- Production contains personal data only. Fabricated fixtures belong exclusively to local/CI tests and must never connect to production destinations.
- The model's 360-hole threshold is a minimum eligibility gate, not a promise of useful predictions.
- No GPS, native iOS build, social features, multiple golfers, weather service, handicap calculation, Airflow, or Dagster in this version.

## Repository layout and development commands

Create these responsibility boundaries as their tasks begin, rather than filling the repository with empty directories:

| Location | Responsibility |
| --- | --- |
| `apps/capture/` | React app, local storage, sync, domain validation, and browser tests |
| `contracts/` | Versioned hole/round schemas, enum definitions, workbook headers, and public export schema |
| `supabase/` | Local configuration, migrations, RPC functions, grants, RLS, and database tests |
| `pipelines/src/fairway_log/` | Workbook generation/import, canonical ETL, pipeline entry point, statistics, and export |
| `pipelines/tests/` | Python unit/integration tests and small fixture builders |
| `warehouse/` | dbt sources, staging, facts, dimensions, marts, tests, and documentation |
| `tableau/` | Workbook definition, calculation notes, screenshots, and publishing instructions |
| `docs/` | Setup, operations, metric definitions, and measured results |
| `.github/workflows/` | CI and production workflow definitions |

Use Node 24 LTS, pnpm 11, Python 3.12, and a project-local uv environment. Resolve compatible stable library versions during bootstrap and commit lockfiles; do not use the unrelated global dbt installation. Use the PostgreSQL major version supplied by the selected Supabase project for local and CI testing. Docker or a free compatible runtime is required for local Supabase.

Expose these commands and document them in `docs/development.md`:

```sh
pnpm install --frozen-lockfile
pnpm --filter capture dev
pnpm --filter capture check
pnpm --filter capture test
pnpm --filter capture build
pnpm --filter capture test:e2e
uv sync --frozen
uv run pytest pipelines/tests
supabase start
supabase db reset --local
supabase test db
uv run dbt deps --project-dir warehouse --profiles-dir warehouse
uv run dbt build --project-dir warehouse --profiles-dir warehouse --target local
uv run fairway template --output /tmp/fairway-rounds-v1.xlsx
uv run fairway pipeline --target local
uv run fairway replay --target local
```

Database reset and fixture commands must reject hosted URLs. Ignore local data, secrets, logs, extracts, browser traces, model binaries, and temporary workbooks. Commit `.node-version`, `.python-version`, lockfiles, and environment examples; remove the existing ignore entry for `.python-version`.

## Shared decisions and contracts

### Golf and workbook rules

- One round contains 9 or 18 scheduled holes. A 9-hole round uses 1–9 or 10–18. An 18-hole round uses all 18 and orders them from the selected starting nine.
- Draft holes may omit performance fields; completing a round requires all scheduled holes to validate. Fields start blank, never as guessed scores or misses.
- Use the approved spreadsheet columns and ranges: par 3–6, yardage 50–800, score 1–20, penalties 0–10. Apply the same domain rules in the app, database, and importer. Penalties cannot exceed score; putts plus penalties cannot exceed score.
- First-putt distance uses whole feet. Accept zero as an estimate of less than half a foot when there was a putt; store null when there were no putts. This resolves the earlier strictly-positive spreadsheet rule without forcing inaccurate data.
- Tee outcomes are `FAIRWAY`, `GREEN`, the four directions, each direction with `_OB` or `_BUNKER`, and `NOT_APPLICABLE`. Approach outcomes exclude `FAIRWAY` and allow `NOT_APPLICABLE` for a hole without a separate approach. Blank required results remain validation errors.
- Record the par-3 tee outcome normally and copy it to approach accuracy; do not replace its direction with `NOT_APPLICABLE`. FIR applicability is a separate calculation. Tee `NOT_APPLICABLE` is reserved for genuinely unrecorded/exceptional outcomes and excluded from the FIR denominator rather than treated as a miss.
- Store clubs as stable IDs; record label snapshots with the round so renaming or retiring a club does not change history. New imported clubs start inactive in My Bag.
- Preserve the approved GIR formula, `score - putts <= par - 2`, as **inferred GIR**. It is a proxy: chip-ins, off-green putts, and leaving a green can make it differ from actual regulation status. Document the limitation and do not describe it as measured shot-level truth.
- Define putts as strokes played from the putting green. Penalties are already included in score. Do not subtract them again when calculating GIR.
- Define tee `GREEN` on a par 4/5 as a hit under this project's FIR convention. Exclude par 3 and `NOT_APPLICABLE` results from FIR percentages. Never average per-round percentages when a hole-weighted rate is intended.
- Preserve supplied exact dates privately; never infer missing historical observations or silently fill them with zero.

### Persistent entities and access

Use one owner provisioned through Supabase administration, email/password login, and disabled self-service sign-up. Store approved owner IDs in a private table and require membership as well as `auth.uid()` in every app policy.

| Schema | Tables and grain |
| --- | --- |
| `public` | Courses; tee sets; versioned hole definitions; clubs; rounds; round holes; import status summaries; duplicate-review summaries |
| `raw` | Immutable round revision envelopes, imported row payloads, and reference snapshots |
| `core` | One latest valid row per round and per `(round_id, hole_number)` after ETL |
| `quality` | Source processing ledger, quarantined records, import row errors, pipeline runs, and export attempts |
| `analytics` | dbt models, statistical output, and model evaluation tables |

Every owned operational table includes `owner_id`. Foreign keys must enforce same-owner references, not only entity existence. Courses and tees use normalized names for import matching while preserving display names. Hole definitions and historical round snapshots are immutable; editing a tee creates a new definition version.

Operational performance writes use checked RPCs, not direct browser table updates. Enable RLS on every exposed table, revoke anonymous access, and restrict function execution. Privileged RPCs use fixed search paths and explicit owner checks. Unexposed schemas are unavailable through the browser API.

Use distinct database roles for ETL, dbt, and publishing. The exporter reads only approved public-export views. Supabase service credentials, when necessary for private storage, exist only in trusted workflow steps. Never expose them through `VITE_` variables or logs.

### App synchronization interface

Synchronize a complete round snapshot because each round has at most 18 holes. This makes course snapshots, hole revisions, completion, and raw lineage atomic without a network request for every field.

```ts
type RoundStatus = 'IN_PROGRESS' | 'COMPLETED' | 'ABANDONED';
type SyncResult =
  | { status: 'APPLIED' | 'DUPLICATE'; serverRevision: number }
  | { status: 'CONFLICT'; serverRevision: number; serverSnapshot: RoundSnapshot };

// Supabase RPC: sync_round(p_request jsonb) returns jsonb
type SyncRequest = {
  schemaVersion: 1;
  mutationId: string;       // UUID, reused for an identical retry
  roundId: string;          // UUID, created locally once
  baseRevision: number;    // last acknowledged server revision, zero for a new round
  snapshot: RoundSnapshot; // metadata, source, ordered holes, definition/club snapshots
};
```

`RoundSnapshot` follows `contracts/round-v1.schema.json`: date, selected course/tee/version IDs, hole count, start hole, status, and per-hole fields defined above. Owner comes from the authenticated session, never the supplied payload. Spreadsheet-created rounds carry immutable source identity and are read-only in the hole editor; corrected uploads are their edit path.

Within one database transaction: lock the round, check owner/base revision, validate the snapshot, increment server revision, update operational state, and append a raw envelope. A repeated mutation ID with the same payload returns the recorded result; the same ID with different content is rejected. A mismatched base revision returns a conflict instead of discarding data.

Local draft generation counters are independent of server revisions. Acknowledge only the submitted generation; edits made during a request stay queued. Before the next write, apply the acknowledged server revision to the queued snapshot. Stop the queue on a conflict and offer `Use server copy` or `Keep this version`; the latter is a new explicit mutation against the refreshed server revision. Retain both copies locally until resolved.

### Pipeline interfaces

```text
fairway template --output PATH
fairway import --target local|production [--import-id UUID]
fairway etl --target local|production
fairway statistics --target local|production
fairway model --target local|production
fairway export --target local|production [--dry-run]
fairway pipeline --target local|production
fairway replay --target local
```

Commands return zero on success, nonzero on infrastructure/contract failure, and a sanitized summary with counts. Rejected business records yield `PARTIAL` data status and do not prevent independent valid rounds from progressing. Stage failures do stop publication. Production replay is not exposed in version 1; use the documented recovery procedure with a separate target.

Use a durable per-event processing ledger. A sequence ID alone is unsafe as an exclusive high-water mark because concurrent transactions can commit out of sequence. Select unprocessed committed events, apply each round transactionally, and mark those exact events processed in the same transaction. Sequence ranges remain audit metadata. Latest revision wins, and replay must include reference snapshots, corrections, and duplicate decisions.

## Milestone 1 — Database foundation

### Task 1. Bootstrap a reproducible project

**Produces:** app/workspace manifests, locked Python package, contracts directory, local Supabase setup, development instructions, and initial CI.

- [ ] Create runtime pins, package manifests, TypeScript checks, Ruff configuration, pytest configuration, and lockfiles.
- [ ] Establish `capture` as the app package and `fairway` as the Python console entry point.
- [ ] Add separate local/CI/production configuration with explicit destination checks. Fixture loaders require local database and local storage endpoints.
- [ ] Add CI that installs from lockfiles, runs checks and unit tests, builds the app, and exposes no production secrets to pull requests.
- [ ] Verify installation in a clean environment and document required Docker-compatible runtime setup.

**Acceptance:** another checkout can install and run the empty app and Python package using the documented commands. Production execution requires explicit configuration; no command defaults to a hosted database.

### Task 2. Establish domain rules, migrations, and access

**Consumes:** the shared decisions above. **Produces:** versioned JSON contracts, migrations, generated database types, SQL tests, and an entity diagram.

- [ ] Write shared contract cases covering completed/draft rounds, 9/18-hole order, enums, nulls, club references, and known GIR limitations.
- [ ] Implement TypeScript and Python validators against those same cases; keep calculations in SQL/dbt for reporting and mirror previews in the app with parity tests.
- [ ] Create owner-controlled reference tables, versioned tee definitions, operational rounds, raw envelopes, and the processing ledger.
- [ ] Implement `sync_round`, including locked revision checks, immutable source identity, same-owner foreign keys, and atomic completion validation.
- [ ] Add authenticated reference-management RPCs with server revisions; reference setup is online-only in version 1. Existing downloaded reference data remains available offline.
- [ ] Add SQL tests for anonymous requests, a second user, reference substitution, direct write denial, retry idempotency, stale revisions, and transaction rollback.

**Acceptance:** one authenticated round snapshot creates exactly one operational round and raw revision. Retrying changes neither count; a different concurrent edit produces a conflict. Partial failures leave neither half-saved operational state nor orphan raw events.

## Milestone 2 — Usable iPhone capture

### Task 3. Course, bag, and sign-in screens

**Produces:** owner login, course/tee setup, My Bag, reference caching, and a blank history screen.

- [ ] Build a simple React Router app with Home, Courses, My Bag, and History. Use native controls and a small CSS system; minimum 44px touch targets, visible labels, keyboard focus, and inline errors.
- [ ] Implement email/password login and session refresh. Initial account creation and password recovery remain documented administration actions; no public registration UI.
- [ ] Build course/tee setup and club add/rename/retire. A nine-hole definition may represent front or back nine; enabling 18-hole play requires all 18 definitions.
- [ ] Download reference data into Dexie after login and show whether offline setup is available. Never overwrite snapshots in existing rounds when reference data changes.
- [ ] Add a production PWA manifest and service worker using `vite-plugin-pwa`; cache only the app shell/static resources. Keep private API responses out of service-worker caches.

**Acceptance:** the owner can configure a course and bag, install the app through Safari, reopen it offline, and see the previously cached choices. A fresh offline browser explains that first-time setup requires a connection.

### Task 4. Hole entry, local durability, and synchronization

**Produces:** a complete playable round workflow and conflict-safe outbox.

- [ ] Add Dexie stores for owner-scoped references, round drafts, outbox requests, and conflict copies; version schema upgrades with migration tests.
- [ ] Save each field change locally and retain unfinished forms. Debounce online sync until 750ms after the last edit and flush on hole navigation/completion; do not depend on background execution.
- [ ] Show one hole at a time, a compact progress strip, a scorecard review, and save-and-exit. Use numeric keyboards for scores/distances. Offer accuracy as direction plus result controls backed by the fixed enum; no map or shot tracker.
- [ ] Implement local transaction boundaries, single-tab queue ownership, retry backoff for transient errors, and the mutation contract above. Authentication errors pause sync until sign-in succeeds; validation errors wait for correction.
- [ ] Resume on app open, connectivity restoration, or manual retry. Show `Saved on phone`, `Syncing`, `Synced`, and actionable errors. Request persistent browser storage when supported but make no guarantee against user-cleared storage.
- [ ] Queue completion after all expected holes validate. Permit later edits to completed app rounds through a new validated snapshot. Abandonment retains history but excludes the round from reporting.
- [ ] Preserve local drafts through expired sessions. Before sign-out, explain unsynced changes and allow cancel; scope local records so another login cannot view them.
- [ ] Offer service-worker updates only after saving drafts and avoid forced reloads during a round.

**Acceptance:** Playwright tests cover offline entry, reload, edits during in-flight sync, repeated submissions, conflict resolution, expired login, and reconnect. A physical iPhone completes an 18-hole round in airplane mode, relaunches, reconnects, and produces 18 unique latest holes. Record the actual device/browser version and outcome.

## Milestone 3 — Historical spreadsheet import

### Task 5. Generate the template and accept private uploads

**Consumes:** `contracts/workbook-v1.json` with the approved 17-column order. **Produces:** a downloadable blank `.xlsx` and authenticated import screens under History.

- [ ] Generate `instructions`, `holes`, and protected `lists` sheets with openpyxl. Set `instructions!B1` to `1`, freeze the header, size columns, format dates, and add dropdown/range validation through row 20,001. Ship no fabricated rows in the production template.
- [ ] Create private Storage buckets for source workbooks and error reports. Enforce authenticated owner/path policies, file-size limits, and immutable uploaded objects.
- [ ] Implement `begin_import` and `finish_import_upload` RPCs. The latter verifies object presence and marks a batch ready; only the worker can mark processing/results.
- [ ] Let the browser check extension/size and hash the file; recompute SHA-256 in Python before trusting it. A unique owner/checksum registration handles concurrent identical uploads.
- [ ] Add status/detail pages with accepted/rejected counts, retry for infrastructure failures, and short-lived authenticated downloads. Imports are queued for the next workflow; explain that upload success is not processing completion.

**Acceptance:** template headers and validations are tested; the file opens in Excel with zero data rows. Owner isolation, oversize rejection, abandoned upload handling, duplicate uploads, and source immutability pass integration tests. Both the 25 MB and 20,000-row limits apply independently.

### Task 6. Parse, validate, and import complete rounds

**Produces:** the Python workbook worker, private row error reports, source revisions, and imported rounds visible in app history.

- [ ] Open workbooks in read-only mode and inspect formulas without executing them. Reject formulas in data cells, unsupported versions, macros, missing/extra/reordered headers, malformed dates, and invalid file structure before loading rounds. Accept ISO date strings and genuine Excel date cells; reject locale-ambiguous strings.
- [ ] Validate workbook ZIP expansion (maximum 100 MB uncompressed) and stop on limits. Stream rows into a temporary local SQLite spool keyed by `round_key`; this supports interleaved rounds without retaining the workbook in memory. Delete temporary data after the run.
- [ ] Read and validate the entire file envelope before promoting any round. Empty workbooks fail with `NO_ROWS`; ordinary blank trailing rows are ignored.
- [ ] Normalize labels by whitespace trimming and case-folding, preserve original values privately, and process one complete round transaction at a time. Missing observations produce actionable errors; do not fabricate them.
- [ ] Match course and tee names within the owner. A new complete nine may create or extend a tee definition; conflicting existing holes reject that round. Historical differing yardages require a separately named historical tee definition, never overwriting earlier definitions.
- [ ] Create new historical clubs as inactive. Update operational history and append raw round events atomically; canonical ETL remains the sole writer to `core`.
- [ ] Use stable spreadsheet identity `(owner_id, round_key)`. A different valid workbook with the same key is a correction, assigned the newer upload sequence. An older queued batch must not overwrite a newer accepted correction. Invalid corrections leave the previous accepted round intact.
- [ ] Claim batches with an expiring lease and store per-round results keyed by batch/key. Retry infrastructure errors using the same batch; identical uploaded files return the original batch's status rather than creating a new import.
- [ ] Produce private CSV reports containing row, field, supplied value, error code, and reason; escape spreadsheet formula-leading text in the report.

**Acceptance:** generated test workbooks import 251 × 18 = 4,518 holes and the 20,000-row limit. Mixed files accept valid rounds and reject whole invalid rounds. Retrying, reversing batch processing order, and crashing after round commit create no duplicates or lost corrections.

### Task 7. Resolve possible app/import duplicates

**Produces:** a small review flow linked from import results.

- [ ] Match date, course, and tee against app rounds; do not assume that two rounds on the same date are duplicates.
- [ ] Preserve the imported candidate and mark it `NEEDS_REVIEW`; exclude it from analytics until resolved. Separate review counts from rejected and accepted counts.
- [ ] Offer `Keep both`, `Keep app round`, and `Keep imported round`. Keep both is explicit confirmation; the other actions exclude the alternate record from reporting without destroying its raw history.
- [ ] Append the resolution as an audited event so replay preserves the decision. Re-evaluate review status when the imported date/course/tee changes in a correction.

**Acceptance:** a candidate cannot inflate Tableau totals before review. All three decisions replay deterministically, remain owner-only, and keep both raw histories recoverable.

## Milestone 4 — Reliable ETL and warehouse

### Task 8. Canonical ETL, quarantine, and replay

**Produces:** `core.rounds`, `core.holes`, canonical references, processing ledger, run summaries, and a local replay command.

- [ ] Read immutable events from both sources through the same typed validation boundary. Revalidate source payloads even when the app validated them earlier.
- [ ] Claim unprocessed events and compare source/server revisions within a round transaction. Store lineage to the accepted event and source revision in canonical records.
- [ ] Apply whole-round corrections atomically, including removing obsolete canonical holes; preserve the prior version in raw history. An invalid newer event does not erase the last valid state.
- [ ] Track processed, superseded, rejected, and retryable events explicitly. Commit canonical changes and ledger entries together. Quarantine invalid payloads privately with stable rule codes.
- [ ] Record run start/end, code revision, counts, durations, and sanitized error summaries. Write final failure status even when a later stage fails.
- [ ] Implement replay into isolated shadow tables, applying events in revision order and comparing keyed content hashes/counts against incremental results. Never truncate production to demonstrate replay.

**Acceptance:** out-of-order commits, delayed old revisions, correction replays, rejection retries, partial crashes, and abandoned/duplicate exclusions produce the same canonical state as full replay. No sequence-only watermark can skip a late-committed event.

### Task 9. dbt dimensions, facts, and marts

**Produces:** documented SQL models, lineage, and analytical tests.

- [ ] Build staging models on `core`, then `dim_course`, `dim_tee_set_version`, `dim_hole_version`, `dim_club`, and private `dim_date`.
- [ ] Build `fct_hole_performance` at one row per round/hole and `fct_round` at one row per round. Preserve input provenance privately and admit only completed, reporting-eligible rounds to published marts.
- [ ] Derive inferred GIR, project FIR, score-to-par, three-putt indicator, club outcomes, and miss direction/result from atomic inputs.
- [ ] Build explicit marts for round history, hole results, club summaries, putting distance bands, and form. At personal scale use full table rebuilds in dbt; incremental engineering is demonstrated in ingestion/ETL where corrections and retries need it.
- [ ] Compute raw-score averages separately for 9/18-hole rounds; compare all rounds using strokes relative to par per hole. Rolling five-round summaries require five eligible rounds and weight rate denominators by holes/opportunities.
- [ ] Use putting bands 0–3, 4–6, 7–10, 11–20, 21–40, and 41+ feet. Report subsequent putt count/three-putt frequency rather than claiming first-putt make probability. Report penalty strokes and associations, not counterfactual strokes saved.
- [ ] Add dbt uniqueness, relationship, null, enum, completeness, denominator, and reconciliation tests. Verify model results against hand-calculated fixture expectations.
- [ ] Generate dbt docs/lineage using fixture-only CI data; do not expose private production catalogs or source samples through public artifacts.

**Acceptance:** dashboard-ready totals reconcile to accepted completed rounds. FIR excludes ineligible holes, club joins do not multiply rows, 9-hole averages do not distort 18-hole averages, and changes to an old round update all dependent marts.

## Milestone 5 — Orchestration and Tableau

### Task 10. Scheduled production pipeline and recovery

**Produces:** independent CI and production workflows plus `docs/operations.md`.

- [ ] Trigger production daily at `17 18 * * *` UTC (03:17 Japan time) and through `workflow_dispatch`. Use the trusted default branch and standard Linux runners.
- [ ] Serialize production runs with a workflow concurrency group and `cancel-in-progress: false`; acquire a database lock for the duration as protection against local/manual duplicate runners.
- [ ] Run workbook import → canonical ETL → `dbt build` → statistics → conditional model → export. Add the later stages only as their tasks are delivered.
- [ ] Put production secrets in a dedicated GitHub environment. Pull requests use disposable local Supabase and fixture data only. Pin external Actions to immutable commits and grant minimal token permissions.
- [ ] Set a 45-minute workflow timeout. Retry only transient network/rate errors three times with backoff; do not retry invalid records as infrastructure failures.
- [ ] Use GitHub's run history and failure notifications, with sanitized summaries. Keep raw files, private dbt outputs, and error reports out of public logs/artifacts.
- [ ] Document manual retry, restoring a stuck lease, Supabase project resume, daily schedule delays, and re-enabling schedules after prolonged repository inactivity. Show actual freshness in the app/dashboard rather than assuming every schedule ran.
- [ ] Provide an owner-run private backup/download command and restore rehearsal against local Supabase. Free-plan database backups must not be assumed; never commit backup files.

**Acceptance:** a failed dbt test prevents export, a retry safely resumes accepted imports, overlapping triggers do not duplicate writes, and a fresh checkout can run the same pipeline locally. Simulated outages result in visible failure status and preserve the last published data.

### Task 11. Public export and Tableau dashboard

**Produces:** Google Sheets delivery, a Tableau workbook, screenshots, and a public dashboard link.

- [ ] Create a versioned export allowlist under `contracts/`. Export `holes_public`, `rounds_public`, `statistics_public`, `model_public`, and `metadata` tabs. Each row carries the same export generation ID.
- [ ] Assign a public round sequence in private date order with a deterministic tie-breaker. Export only month as `YYYY-MM`, course/tee labels, hole/club snapshots, performance fields, sample sizes, and approved derived values. Export no internal UUIDs, exact dates, import keys, filenames, or account fields.
- [ ] Allow an exact pipeline refresh timestamp in metadata; this records publication time, never playing time. Enforce typed allowlisted columns and data lineage gates rather than relying on field-name removal.
- [ ] Read all marts in a consistent database snapshot and validate the entire payload before publishing. Upload the approved values to generation-specific hidden staging tabs in bounded chunks with literal `RAW` values. Tableau connects only to the five stable published tabs. Hidden tabs are not a privacy boundary; every staged value must already pass the public allowlist.
- [ ] Verify staging counts, then use one Google `spreadsheets.batchUpdate` call to resize/clear published ranges, copy staged values into all stable tabs, write metadata, and delete staging tabs atomically. Use value-only copy requests so large exports do not require one large payload. An earlier failure leaves the published tabs intact; after an uncertain final response inspect the generation ID before retrying. Clean abandoned staging tabs on the next run. Test both 4,518 and 20,000-row publication payloads and shortening exports.
- [ ] Author Tableau with Google Drive/Sheets connections for every logical table. Relate hole and round data by public sequence using logical relationships, not a physical join that multiplies round metrics. Use separate sources for unjoined statistics/model summaries.
- [ ] Build Overview, Tee & Approach, Putting & Scoring, and Methodology views. Include month/course/tee/par filters, explicit denominators, sample sizes, and refresh metadata. Add model results only when eligible.
- [ ] Use a readable light theme, simple charts, consistent colors, and mobile layout. Store calculated field definitions and metric notes with the workbook.
- [ ] Publish through Tableau Public's supported authoring UI, enable its daily extract refresh, and verify an actual later refresh. Google export success alone is not Tableau refresh success. Reject mixed-generation extracts in workbook calculations and show a refresh notice.
- [ ] Commit a source-only `.twb` and approved screenshots. Do not commit packaged workbooks, credential files, or private extracts. Inspect the workbook XML for local/private connection metadata before committing.

**Acceptance:** exported column/row counts reconcile to eligible marts; a shorter dataset leaves no stale rows; a failed API request leaves the old complete dataset; all views calculate the fixture expectations locally. Production publishing waits for actual personal rounds, supplied later by the user. Do not publish fixture dashboards as personal results.

## Milestone 6 — Statistics and interpretable modeling

### Task 12. Analysis with honest evaluation

**Produces:** reproducible analysis code, private model artifacts, public result tables, and a concise findings document after real data is available.

- [ ] Implement descriptive trends immediately. Use fixed-seed bootstrap resampling by round, 1,000 draws, for 95% intervals around mean score-to-par and overall rates; omit intervals below 10 rounds and show the sample size.
- [ ] Gate model eligibility on at least 360 eligible completed holes and 20 distinct rounds. A data correction below the gate clears the published model output instead of leaving stale results visible.
- [ ] Define the model as pre-hole score-to-par prediction using par, yardage, and tee-club category. Exclude tee/approach outcomes, approach club, putts, penalties, GIR, FIR, score, and identities because these are unavailable before the hole or leak outcome information. Analyze shot outcomes descriptively instead.
- [ ] Reserve the newest 20% of complete rounds as an untouched chronological holdout. Within earlier rounds use expanding-time validation on five round blocks (four validation folds), keeping all holes from a round together.
- [ ] Fit imputation, scaling, and one-hot encoding inside each training fold. Tune Elastic Net across alpha `[0.01, 0.1, 1.0, 10.0]` and l1 ratio `[0.1, 0.5, 0.9]` using MAE. Use deterministic seeds and handle unseen clubs.
- [ ] Compare to a training-only par-group mean baseline with overall-mean fallback. Report holdout MAE, R² when defined, per-par sample sizes, residuals, and the baseline comparison, including a negative result honestly.
- [ ] Store dataset fingerprint, code/dependency version, feature list, split boundaries, metrics, and training timestamp privately. Retrain when the eligible-data fingerprint or modeling code changes, not merely on every nightly schedule.
- [ ] Export only approved aggregate evaluation results. Never claim that the model establishes causation or generalizes to other golfers. Write actual findings only after evaluating personal data.

**Acceptance:** tests cover 359/360 holes, insufficient round count, corrections changing eligibility, shuffled input order, unseen clubs, no test-set preprocessing leakage, time-ordered split boundaries, baseline calculation, and failure of the model to beat baseline. A weak model remains a documented finding, not a reason to manipulate the evaluation.

## Final integration and handoff

- [ ] Provision the free Supabase project, owner account, Cloudflare Pages site, Google spreadsheet/service-account share, GitHub secrets, and Tableau account connection as needed. Obtain credentials through the providers' supported sign-in flow; never ask for secrets in a repository file.
- [ ] Deploy the app early at Milestone 2; apply reviewed migrations explicitly and run owner-access smoke checks. Back up existing records before any later migration that changes data.
- [ ] Keep every test fixture builder restricted to local/CI endpoints. Generate large workbook cases during tests instead of storing thousands of fabricated rows in Git.
- [ ] Run the full local acceptance sequence: offline round → sync → 251-round workbook → correction → duplicate review → ETL → dbt → export dry-run → analytical reconciliation.
- [ ] Rehearse recovery from a network failure, process crash, expired auth, and bad newer import revision. Record observed outcomes and runtimes, not estimated success claims.
- [ ] When the user's real workbook arrives, validate and import it, resolve errors without guessing values, reconcile source counts, publish approved marts, and verify Tableau's later refresh.
- [ ] Add a short portfolio evidence document linking tested pipeline runs, source/warehouse diagram, lineage, dashboard, measured import counts, and actual findings. Keep résumé claims limited to completed and measured capabilities.

The software can be built and tested before the historical workbook arrives. Personal-data reconciliation, public dashboard validation with real rounds, and meaningful model findings are separate final acceptance gates that remain pending until that data is available.

## Reference behavior checked for this plan

- [Supabase row-level security](https://supabase.com/docs/guides/database/postgres/row-level-security): browser access is governed by RLS and appropriate grants.
- [Google Sheets batch requests](https://developers.google.com/workspace/sheets/api/guides/batch): a batch groups subrequests and applies them atomically when valid.
- [Tableau Public FAQ](https://help.tableau.com/current/pro/desktop/en-us/public_faq.htm): Public uses extracts, with daily automatic refresh available for Google Sheets sources; it does not provide a live Supabase connection.
