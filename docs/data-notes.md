# Data Notes

These are the rules I want to settle before writing database tables or dashboards. Keeping them here gives the app, pipeline, and Tableau workbook one shared definition of the data.

## Course setup

A course can have more than one tee set. Each tee set stores:

- Hole number
- Par
- Yardage

Par and yardage are copied into a round when it starts. Editing a course later should not rewrite an old scorecard.

Distances around the course are recorded in yards. First-putt distance is an estimated whole number of feet.

## Hole entry

Each completed hole records:

| Field | Notes |
| --- | --- |
| Score | Total strokes including penalties |
| Par | Copied from the selected tee set |
| Putts | Zero is allowed for a hole-out |
| Tee accuracy | Direction or final result of the tee shot |
| Approach accuracy | Direction or final result of the shot intended to reach the green |
| First-putt distance | Whole feet when putts are greater than zero; zero means an estimate below half a foot; blank when there were no putts |
| Penalties | Penalty strokes included in the score |
| Tee club | Selected from My Bag |
| Approach club | Optional and selected from My Bag |

On a par 3, the tee shot is also the approach. Its accuracy is entered once and used for both fields.

## Accuracy choices

Tee accuracy:

- Fairway
- Green
- Left, right, long, or short
- Left, right, long, or short out of bounds
- Left, right, long, or short bunker
- Not applicable

Approach accuracy uses the same directional, green, out-of-bounds, and bunker choices, without fairway. `Not applicable` is available when there was no separate approach. On par 3s, record and copy the actual tee outcome rather than choosing `Not applicable` merely because FIR does not apply.

## Calculated golf metrics

### Green in regulation

```text
strokes before putting = score - putts
GIR = strokes before putting <= par - 2
```

GIR is calculated rather than entered and labeled as inferred. The formula cannot establish actual regulation status in every case: chip-ins or leaving a green can change score minus putts without describing when the green was first reached. Putts means strokes played from the putting green.

### Fairway in regulation

- Par 4 or 5 ending in `Fairway` or `Green`: hit
- Par 4 or 5 with another recorded tee result: missed
- Par 3: not applicable
- Tee result `Not applicable`: unknown and excluded from the denominator

FIR is also calculated rather than entered. Counting a par-4/5 tee shot on the green as a hit is this project's reporting convention.

## Public data boundary

The Tableau dataset may contain course names, month and year, round sequence, hole details, clubs, and derived performance metrics.

It will not contain exact playing dates, tee times, account details, raw submission payloads, rejected records, or unfinished rounds. The private Supabase database remains the source of truth.

## Data-quality examples

The cleaning pipeline should catch or quarantine:

- A hole submitted twice during synchronization
- Putts greater than the total score
- Missing required clubs or outcomes
- A hole that does not belong to the selected tee set
- A 9-hole round containing the wrong set of hole numbers
- Invalid score, par, yardage, penalty, or distance values
- Older edits arriving after a newer version of the same hole
