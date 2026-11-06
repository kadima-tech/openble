# Manual tape measurements

Design for the Bathroom Hub. This pull request is the spec only. Implementation follows review of this document.

Tape occasions are their own rows. Weight keeps its chart. Waist keeps its own chart. Body fat is one graph with three labeled lines: the scale estimate (Jebb), the tape estimate (Navy), and an unvalidated mean. Each person chooses which of those three is the main number. Stored scale fat and the Google Health upload do not change.

## What stays as it is

Sex is exactly `male` or `female`. `profiles.sex` is `CHECK (sex IN ('male', 'female'))`. The person form is two radios. The error is "Choose male or female; the body-fat equation needs it." Both formulas use that pair. There is no other stored value.

`measurements.fat_pct`, `lean_kg`, and `bmi` stay the Jebb results written at assignment and by `recompute_profile`. `Hub.upload` still sends that `fat_pct` as Google Health `body-fat`, and only that. Saving a tape, editing its time, or changing the selector does not write those columns, change `google_status`, or enqueue an upload. The "Sent" badge still means the stored scale estimate.

The weight card and `charts.line_chart` for weight are unchanged: kilograms, the same 7d / 30d / 90d / 1y window, one series. Waist stays on its own centimetre chart (`min_span` 2). Neck and hip are numbers on the tape card, not lines.

## Evidence

No published equation takes this scale's inputs together: one foot-to-foot impedance, weight, height, age, sex, waist, neck, and hip.

| Source | What it uses | Why it is not a combined formula here |
| --- | --- | --- |
| Jebb et al., Br J Nutr 2000, 83:115–122 | Foot-to-foot impedance, height, weight, age, sex. Residual SD 4.8% men, 3.3% women. | Already stored. No circumferences. |
| Hodgdon and Beckett, NHRC 84-11 (men) and 84-29 (women) | Height and circumferences. Men: abdomen and neck. Women: waist, hip, and neck. Inches inside the logs. | No impedance and no weight. This is the tape formula. |
| Stevens et al., Int J Obes 2017 (PMC5675766) | Hand-to-foot 50 kHz resistance and reactance, age, ethnicity, size. Waist plus a skinfold raised fit. | Needs reactance and ethnicity. No neck. This scale stores one impedance. |
| Luke et al., METS (PMC3766444) | Tetrapolar resistance, weight, height, sex. Waist was measured and is not in the published equation. | Different device. No neck. |
| Yeh et al. (PMC4489024) | Foot-to-foot impedance, weight, height, age, sex, in healthy Asian adults. | No waist or neck. Not a replacement for the stored scale estimate. |
| Cicone et al. (PMC8785250) | Skinfolds, waist, BMI, age, sex. | No impedance. Needs calipers. No neck. |

Papers that measure both a circumference equation and BIA compare each with a criterion. They do not validate the arithmetic mean of the two percentages. The mean is a display choice. The UI calls it an unvalidated mean and does not say it is more accurate. The Jebb ±4.8 / ±3.3 line stays on the scale figure only.

## Storage

New table, via the existing `CREATE TABLE IF NOT EXISTS` script in `db.py`. No tape columns on `measurements`.

```sql
CREATE TABLE IF NOT EXISTS tape_measures (
    id INTEGER PRIMARY KEY,
    profile_id INTEGER NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
    measured_at TEXT NOT NULL,
    utc_offset_s INTEGER NOT NULL,
    waist_cm REAL,
    neck_cm REAL,
    hip_cm REAL,
    received_at TEXT NOT NULL,
    CHECK (waist_cm IS NOT NULL OR neck_cm IS NOT NULL OR hip_cm IS NOT NULL),
    UNIQUE (profile_id, measured_at)
);
```

`measured_at` is UTC with `timespec="seconds"`, the same form as a weigh. A second save for the same person and the same instant shows "You already have a measurement at that time. Edit that one."

Ranges: waist 50–200 cm, neck 20–60 cm, hip 60–200 cm. At least one circumference. Comma decimals. One error list. POST, then redirect, flash "Saved on this hub."

Hip is optional for every person. The field is on the form for male and female profiles, and a value is stored for either. When the profile is female, the hint is "Needed by the Navy equation for women." Waist and neck still save without hip. Her tape estimate, and any mean that needs it, stay absent until hip is present. When the profile is male, the field has no requirement hint. The men's formula does not read `hip_cm`. A stored hip is a number on the tape card for either sex, not a chart line.

The display choice is a column added only when `PRAGMA table_info` does not already list it:

```sql
ALTER TABLE profiles ADD COLUMN fat_display TEXT NOT NULL DEFAULT 'scale';
```

The three values `scale`, `tape`, and `average` are checked in the person form, not with a `CHECK` on the `ALTER`. Existing and new people default to `scale`. Anything else is read as `scale`. The control is the same segmented radio group as sex, on the person edit page. Copy under it: "Chooses the main body-fat number. The chart still shows all three lines. Google Health still receives the scale estimate."

## Date and time

The tape form uses one `<input type="datetime-local" name="measured_at" required>` in a `.field` label, the same label-above-input pattern as date of birth. A new form defaults to the current local minute, from `datetime.now().astimezone()`, not to noon. Edit and backfill show the stored wall time and let the person change both the date and the time.

`datetime-local` has no zone. The hub reads it as a naive wall clock in the server's local zone, the same zone `scale_time` uses when it calls `astimezone()`.

1. Parse `YYYY-MM-DDTHH:MM` as a naive datetime. Reject a missing value, a date-only value, or any value that already carries a zone. The error is "Date and time are required."
2. Attach the current local `tzinfo` with `fold=0`.
3. Convert to UTC. Format the UTC instant back into that zone and compare the wall clock with the input. If they differ, the clock time falls in a spring-forward gap. Reject it with "That time does not exist in this timezone."
4. A repeated hour at the autumn change is the earlier offset (`fold=0`).
5. Store the UTC instant at whole seconds (`:00`) and `utc_offset_s` from that aware local time. Reject a result later than the current UTC minute. The error is "That time is in the future." A failed save redisplays the submitted value in the field.

The edit field does not use `when()` or the zone in force today. It rebuilds the wall clock from the stored instant plus the stored `utc_offset_s`, then formats `YYYY-MM-DDTHH:MM`. Re-opening the form after a later zone change still shows the time that was saved. Pairing compares the stored UTC instants, so that later zone change does not move a tape into or out of the 24-hour window.

`when()` remains the human label on lists and cards. It converts the UTC instant with the zone in force at display time, as it already does for weighs.

## The three figures

Scale estimate. Stored `fat_pct` where the weigh has usable impedance. Plotted at that weigh's `measured_at`.

Tape estimate. Navy, from that occasion's circumferences and the profile's current height and sex. Men need waist and neck, and ignore hip. Women need waist, neck, and hip. A non-positive log argument, or a result outside 3–70%, is no estimate and is not clamped. Inches are centimetres / 2.54, and only inside the formula.

Men:

```
%fat = 86.010 * log10(abdomen - neck) - 70.041 * log10(height) + 36.76
```

Women:

```
%fat = 163.205 * log10(waist + hip - neck) - 97.684 * log10(height) - 78.387
```

Reference results, to 0.1 point: 19.43 (70 in, abdomen 36, neck 16), 29.21 (64 in, waist 30, hip 38, neck 13), 20.83 (183 cm, waist 92, neck 38), 30.93 (170 cm, waist 80, neck 33, hip 100).

The tape line uses the occasion's own `measured_at`. It is not moved onto a weigh.

Unvalidated mean. The arithmetic mean of the two percentages already rounded to 0.1. Round half away from zero to one decimal with decimal tenths, not binary `round` (`round(24.15, 1)` is 24.1 here). 27.4 and 20.8 produce 24.1. 27.5 and 20.8 produce 24.2. The mean is not stored.

## Pairing

Each weigh chooses its own tape. One tape may be the nearest tape for more than one weigh. There is no rule that keeps the match one-to-one.

A tape is complete when a man has waist and neck, or a woman has waist, neck, and hip. Incomplete rows stay stored and do not enter pairing or the tape line.

For one weigh, look at that person's complete tapes whose UTC instant is at most 86400 seconds from the weigh's UTC instant, before or after. The bound is `<= 86400`. A tape 86401 seconds away does not match. Among those in range, pick the smallest absolute gap. If two are equally far, pick the earlier tape.

This is elapsed time, not a shared calendar date. A tape at 20:00 and a weigh at 07:00 the next morning are 11 hours apart and do pair. A tape more than 86400 seconds away does not, even when the two fall on neighbouring calendar dates. Backfilling a tape with an explicit past date and time pairs it with any weigh inside that window, including historical weighs. The tape point stays at the tape time. The mean point is at the weigh time. Nothing is carried forward beyond the nearest tape inside 86400 seconds. A pair still counts when the tape instant is outside the chart window and the weigh is inside it. The mean is drawn. The tape point is drawn only when its own instant is inside the window.

`bodycomp.nearest_tape(weigh_at, tapes) -> datetime | None` is a pure function. `tapes` are `(measured_at, complete)` pairs. It ignores incomplete rows and returns the chosen tape instant, or `None`.

## The body-fat graph

One graph on the body-fat card, same window as weight. Shared percent axis, `min_span` 1. X is position in the window, the same scale as `charts._scale`.

| Line | Points | Legend |
| --- | --- | --- |
| Scale | Weighs in the window with `fat_pct` | Scale estimate |
| Tape | Complete tape occasions in the window | Tape estimate |
| Mean | Paired weighs only, at the weigh instant | Average of scale and tape estimates. Unvalidated mean. |

`charts.line_chart` stays the single-series chart for weight and for waist. A new helper draws the three body-fat series and does not connect a point of one series to a point of another. Scale and tape are each one polyline. Two mean points are joined only when no weigh and no complete tape occasion sits strictly between them. One point is a dot and no line. An empty series is not drawn. The legend is text and always has three entries. An empty one reads "none in this range". Colour is not the only label. Scale uses the fat tone, tape the lean tone, and the mean a dashed stroke in `--text`. The accessible name lists each series, its point count, and that the third line is an unvalidated mean.

The mean line does not continue to the latest weigh unless that weigh itself has a tape inside 86400 seconds.

## Main number, lean mass, history

The selector changes the big number and the lean figure beside it. It does not hide lines.

- `scale`. The big number is the latest weigh's stored `fat_pct`, with the existing ± line for that sex. If that weigh has no scale fat, say so. Do not substitute a tape figure. Shown lean is that weigh's stored `lean_kg`.
- `tape`. The big number is the latest complete tape occasion. Caption: "Tape estimate. It is not the scale reading." No Jebb ±. Shown lean uses the nearest weigh within 86400 seconds of that occasion, earlier weigh on a tie, and this tape percentage. Caption: "From the tape estimate." With no weigh in that window, omit lean and say it needs a weigh within 24 hours.
- `average`. The big number is the latest weigh that has a pair. The caption is exactly "Average of scale and tape estimates." Under it: "Unvalidated mean of the two estimates. Not a more accurate measurement." If the latest weigh has no pair, the big number is absent. Do not fall back to scale or tape in that slot. Incomplete tapes do not block a farther complete tape inside the 86400 seconds. Say the hip is missing only when no complete tape is in range and the nearest female tape in range has waist and neck but no hip. Otherwise say there is no tape estimate within 24 hours. Shown lean, when a pair exists, is that weigh's weight times `(1 - mean/100)`. Caption: "From the unvalidated mean. Not the stored lean mass."

History keeps every stored scale percentage. A paired row also shows the tape percentage and the mean, using the legend words. The selected figure is the primary one. The raw scale and raw tape numbers stay visible, so a row never shows only a mean.

Routes follow the existing POST-and-redirect pages: `/measures/new` and `/measures/{id}` for the current profile cookie. Delete confirms: "Delete this tape measurement? It is only stored on this hub." Settings adds one sentence after the Jebb paragraph: a tape estimate uses the Navy equations, hip as well for women, and is not sent to Google Health.

## Tests

`apps/bathroom-hub/tests/test_bathroom_hub.py`, `unittest`, no network. Pure cases sit beside the existing Jebb tests. Form cases use the existing `TestClient`.

- Navy references above. A non-positive log and a result outside 3–70% return no estimate. A woman with no hip keeps the row and has no tape estimate and no mean. The same waist and neck with hip produce her estimate. A man's result is identical with and without a stored hip. The form shows hip for both sexes. Only the female form contains "Needed by the Navy equation for women." Sex outside `male` / `female` is still rejected. `fat_display` accepts only `scale`, `tape`, and `average`. An existing database gains the column and still reads as `scale`.
- `nearest_tape`. A tape 11 hours before a weigh matches, including across local midnight. A tape 86400 seconds away matches. A tape 86401 seconds away does not. Two tapes at the same distance: the earlier one wins. Two weighs three hours apart may both select one tape that sits between them. An incomplete tape inside the window is ignored. A tape for another person is ignored.
- Mean. 27.4 and 20.8 → 24.1. 27.5 and 20.8 → 24.2. Saving or backfilling a tape does not change `fat_pct`, `lean_kg`, or `google_status`. The default selector still shows the stored scale percentage and the ± line.
- Time. A new form's `datetime-local` value is the current local minute. A posted `2020-01-02T20:00` is stored as UTC with the offset `astimezone()` gives that wall time, seconds at `:00`. The edit form shows `2020-01-02T20:00`. A future minute re-renders the error. A date-only value re-renders "Date and time are required." The displayed edit value is rebuilt from `utc_offset_s`, not from the current zone.
- Graph. Two scale fats, two tape occasions, and one pair inside 24 hours produce two scale points, two tape points, and one mean point. The mean's x is the weigh instant. The tape's x is the tape instant. The unpaired points are not in the mean series. Two mean points with an unpaired weigh between them are two segments. A tape 11 hours before the window still creates a mean on a weigh inside the window, and that tape point is not drawn. The weight SVG stays one polyline. Waist points are not in the body-fat SVG.
- Selector `average` renders "Average of scale and tape estimates" and "Unvalidated mean", and does not say the result is more accurate. With no tape inside 24 hours, the big number is absent and that gap is named. Selector `tape` does not show the Jebb ± figures.
