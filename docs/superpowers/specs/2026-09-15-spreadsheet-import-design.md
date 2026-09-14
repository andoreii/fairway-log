# Spreadsheet Import Design

## Purpose

Fairway Log will accept historical golf rounds from a fixed Excel template in addition to rounds recorded in the iPhone app. The importer must comfortably process more than 250 rounds, preserve the source workbook, reject unreliable data clearly, and feed the same canonical tables as app-recorded rounds.

The first version accepts up to 20,000 hole rows and a maximum 25 MB workbook; both limits apply. It supports multiple uploads over time and never places personal source files in the public repository. The [implementation plan](../../implementation-plan.md) supplies the executable task sequence and shared edge-case decisions.

## User flow

1. Download the current `.xlsx` template from the private Fairway Log app.
2. Fill one row per hole on the `holes` sheet.
3. Upload the completed workbook from a private import page.
4. The app stores the original file and shows it as `Uploaded`.
5. The scheduled pipeline, or a manually triggered run, validates and imports the workbook.
6. The import page shows accepted rounds, accepted holes, rejected rounds, and a downloadable error report.

The import page does not parse or clean the workbook in the browser. It verifies the extension and size, calculates a checksum, uploads the original, and creates a pending import record.

## Workbook contract

The workbook contains three sheets:

- `instructions`: template version and short usage notes
- `holes`: one row per played hole
- `lists`: protected values used by Excel dropdowns

`instructions!B1` contains template version `1`. The importer rejects unsupported versions without attempting a partial import.

The `holes` sheet contains these columns in this order:

| Column | Required | Rule |
| --- | --- | --- |
| `round_key` | Yes | Stable text identifier for one round, repeated for all of its holes |
| `played_on` | Yes | Private date in `YYYY-MM-DD` format |
| `course_name` | Yes | Trimmed, non-empty text |
| `tee_name` | Yes | Trimmed, non-empty text |
| `holes_played` | Yes | `9` or `18` |
| `starting_hole` | Yes | `1` or `10`; must agree with the included hole numbers |
| `hole_number` | Yes | Integer from `1` through `18`, unique within the round |
| `par` | Yes | Integer from `3` through `6` |
| `yardage` | Yes | Integer from `50` through `800` yards |
| `score` | Yes | Integer from `1` through `20` |
| `putts` | Yes | Integer from `0` through `score` |
| `tee_accuracy` | Yes | Value from the tee-accuracy list |
| `approach_accuracy` | Yes | Value from the approach-accuracy list |
| `first_putt_distance_ft` | Conditional | Non-negative whole feet when `putts > 0`; zero estimates less than half a foot; blank when `putts = 0` |
| `penalties` | Yes | Integer from `0` through `10` |
| `tee_club` | Yes | Trimmed, non-empty text |
| `approach_club` | No | Trimmed text or blank |

Round-level columns must contain the same values on every row sharing a `round_key`. A 9-hole round must contain exactly holes 1–9 or 10–18. An 18-hole round must contain exactly holes 1–18.

On par 3s, `approach_accuracy` must equal `tee_accuracy`. The template instructions call out this rule, and the importer enforces it.

The accepted accuracy labels match the values documented in `docs/data-notes.md`. Input cleaning trims surrounding whitespace and compares labels case-insensitively. It does not perform fuzzy matching or silently guess misspellings.

## Upload storage and tracking

Original workbooks are stored in a private Supabase Storage bucket named `round-imports`. Object paths contain the authenticated user ID and an import UUID, never the original filename alone.

The import record stores:

- Import UUID and owner ID
- Original filename and private object path
- SHA-256 checksum and byte size
- Template version
- Status: `UPLOADED`, `PROCESSING`, `IMPORTED`, `PARTIAL`, `FAILED`, or `DUPLICATE`
- Counts for total, accepted, and rejected rounds and holes
- Separate counts for rounds awaiting duplicate review
- Created, processing-started, and processing-finished timestamps
- Error-report object path when applicable

An identical checksum for the same owner produces a `DUPLICATE` upload result pointing to the original batch and is not processed as a new batch. An infrastructure retry resumes the original batch. The worker verifies the browser-supplied checksum against the actual stored file.

## Processing model

The Python ETL job streams the `holes` sheet in read-only mode into a temporary local SQLite spool. It validates the full workbook envelope before promoting any rounds, normalizes cell types and labels, groups rows by `round_key`, and processes each group as one unit. This supports interleaved round rows without retaining the workbook in memory. Reject formulas in data cells, and do not evaluate workbook formulas or macros.

Round-level atomicity applies:

- A fully valid round is loaded with all of its holes.
- Any invalid hole rejects the entire round.
- A rejected round does not prevent unrelated valid rounds in the workbook from loading.
- Every rejected row retains its sheet row number, field, supplied value, stable error code, and human-readable reason.

Valid spreadsheet rounds enter the same operational history and raw event layers as app rounds, tagged with source type `SPREADSHEET`. Shared canonical ETL then writes `core`; the workbook parser does not bypass it. The original row payload and import ID remain available in the private raw layer for replay and audit. Imported rounds are read-only in the hole editor and corrected by workbook upload.

`round_key` is stable per owner and source type. Uploading corrected data with the same `round_key` creates a new audited import revision and atomically replaces the earlier spreadsheet-sourced version after the entire round passes validation. It never creates a second canonical round. Upload sequence determines correction precedence; processing an older queued upload cannot overwrite a newer accepted correction. An invalid correction preserves the earlier accepted version.

If an imported round appears to match an app-recorded round by date, course, and tee, the pipeline holds the candidate in `NEEDS_REVIEW` and excludes it from analytics. The owner chooses `Keep both`, `Keep app round`, or `Keep imported round`. The resolution is audited and replayable; exclusion never deletes either raw history.

## Courses and clubs

A new course and tee set may be created from an imported round only when the workbook supplies a complete, internally consistent 9-hole or 18-hole definition.

If the course and tee set already exist, imported par and yardage must match already-defined holes. A complete missing nine may extend the tee through a new immutable definition version. A mismatch rejects the affected round with a course-definition error; genuinely different historical definitions require a separate tee name. The importer never overwrites an existing definition.

New club labels are normalized and retained as historical clubs with `active_in_bag = false`. Existing labels map case-insensitively to their stable club IDs. The import does not change the current contents of My Bag.

## Error handling and recovery

- Invalid file type, excessive size, missing sheets, or unsupported template version fails the entire file before row processing.
- A transient storage or database error leaves the import retryable and does not advance its processing checkpoint.
- Pipeline retries are idempotent by import ID, round key, and import revision.
- A crashed run may safely restart without duplicating accepted rounds.
- Error reports are private CSV files available through short-lived authenticated download links.
- Original files and prior revisions are retained; imported history is corrected through a new upload rather than destructive source-file edits.

## Security and privacy

- Only the authenticated owner can upload, list, or download source workbooks and error reports.
- The browser receives a scoped upload path, not a service-role credential.
- The ETL job uses server-side credentials stored in deployment secrets.
- Exact dates, filenames, raw rows, import IDs, errors, and unfinished rounds never enter the public Tableau dataset.

## Testing and acceptance

Automated tests cover:

- A valid workbook containing more than 250 18-hole rounds
- Mixed valid and invalid rounds in one workbook
- Missing, renamed, and reordered sheets or columns
- Duplicate file uploads and safe pipeline retries
- Corrected re-uploads using an existing `round_key`
- Duplicate hole numbers and incomplete 9-hole or 18-hole groups
- Inconsistent round metadata within a group
- Unknown clubs, new courses, and conflicting course definitions
- Conditional first-putt distance and par-3 accuracy rules
- Case and whitespace normalization without fuzzy matching
- Possible duplicates across spreadsheet and app sources
- Owner isolation and unauthorized file access
- Error-report row numbers, values, and stable codes
- Full raw-layer replay producing the same canonical results

Acceptance requires importing at least 251 complete 18-hole rounds in one workbook, producing the expected 4,518 canonical hole rows, no duplicates, and a matching import summary.

## Out of scope

- `.xls`, CSV, Google Sheets, and arbitrary spreadsheet layouts
- Browser-side workbook cleaning
- Automatic fuzzy correction of courses, clubs, or accuracy labels
- Automatic merging of app and spreadsheet rounds
- Importing shot-level GPS data or multiple golfers
