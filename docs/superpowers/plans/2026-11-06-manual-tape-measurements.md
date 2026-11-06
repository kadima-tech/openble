# Manual tape measurements implementation plan

> **For agentic workers:** Execute this plan with test-driven development and task-by-task subagent-driven-development. Do not implement the tasks inline in one pass, and do not switch to a different runner. REQUIRED SUB-SKILL: superpowers:subagent-driven-development, one fresh worker per task, with superpowers:test-driven-development inside every task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Store manual waist, neck, and optional hip occasions, chart them apart from weight, and show scale, tape, and unvalidated-mean body-fat lines with a per-person main number.

**Architecture:** `tape_measures` is a new table. Navy, pairing, the mean, and the headline are pure functions in `bodycomp.py`. Wall-clock conversion sits next to `scale_time` in `hub.py`. `charts.fat_chart` draws the three body-fat lines; `charts.line_chart` stays the weight and waist charts. `web.py` adds the tape form, the profile selector, and the dashboard wiring. Stored `fat_pct` / `lean_kg` / `bmi` and `Hub.upload` stay the Jebb path.

**Tech Stack:** Python 3.11, sqlite3, aiohttp, stdlib `unittest` and `zoneinfo`. No new dependencies.

## Global Constraints

- Do not edit `docs/superpowers/specs/2026-11-06-manual-tape-measurements-design.md`.
- Python 3.11+. Ruff 0.17.1, line length 88, double quotes. No pytest.
- Sex stays exactly `male` or `female`. Do not add another value.
- `measurements.fat_pct`, `lean_kg`, and `bmi` stay the Jebb results. `Hub.upload` still sends that `fat_pct` and only that. Saving a tape or changing `fat_display` must not write those columns, change `google_status`, or enqueue an upload.
- The weight card keeps `charts.line_chart` in kilograms, ranges `("7", "7d", 7), ("30", "30d", 30), ("90", "90d", 90), ("365", "1y", 365)`, and the default `min_span` of 1.
- Waist has its own centimetre chart, `min_span=2`. Neck and hip are numbers, not lines.
- The mean is an unvalidated display value. Copy must not say it is more accurate. The ±4.8 / ±3.3 line stays on the scale figure only.
- Pairing is elapsed UTC time, `<= 86400` seconds, before or after. Equal gap: the earlier tape. One tape may match several weighs. Not a shared calendar date. Nothing is carried further than that nearest tape.
- Navy uses inches = centimetres / 2.54 inside the logs only. Do not clamp. A result `< 3` or `> 70`, or a non-positive log argument, is `None`.
- Hip is optional for every person and is stored for both sexes. The women's estimate needs it. The men's formula does not read it. The female hint is exactly `Needed by the Navy equation for women.`
- The tape field is `<input type="datetime-local" name="measured_at" required>`. A new form defaults to the current local minute, not noon. Store UTC `timespec="seconds"` and `utc_offset_s`.
- Duplicate `(profile_id, measured_at)` shows `You already have a measurement at that time. Edit that one.`
- Success flash is `Saved on this hub.` Delete confirm is `Delete this tape measurement? It is only stored on this hub.`
- `fat_display` defaults to `scale`. Any other stored value is read as `scale`. The selector is on the person edit page, not on welcome or add-person.
- From the repository root, `python -m unittest tests.test_bathroom_hub...` does not load this file. Root `tests/` is the Bluetooth suite. Run hub tests from `apps/bathroom-hub` after `make venv`. `.venv` is not present on a fresh checkout.
- Single-test command, from `apps/bathroom-hub`:

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.<Class>.<method> -v
```

---

## File map

- Modify `apps/bathroom-hub/src/bathroom_hub/db.py`: `tape_measures` table, `fat_display` migration, tape CRUD, `normalize_fat_display`.
- Modify `apps/bathroom-hub/src/bathroom_hub/bodycomp.py`: Navy, tenth rounding, mean, `nearest_tape`, `fat_series`, `fat_headline`.
- Modify `apps/bathroom-hub/src/bathroom_hub/hub.py`: `parse_wall_clock`, `wall_input`, `local_minute_value`.
- Modify `apps/bathroom-hub/src/bathroom_hub/charts.py`: `fat_chart` only. Leave `line_chart` and `sparkline` as they are.
- Modify `apps/bathroom-hub/src/bathroom_hub/static/app.css`: legend and `--text` dashed mean stroke.
- Modify `apps/bathroom-hub/src/bathroom_hub/web.py`: tape pages, edit-page selector, dashboard, history, settings sentence.
- Modify `apps/bathroom-hub/tests/test_bathroom_hub.py`: new tests. Do not change existing assertions.
- Modify `DESIGN.md`: one sentence that the body-fat chart has three labeled lines and the weight chart stays one line.

### Locked interfaces

```python
def normalize_fat_display(value: str | None) -> str: ...

def insert_tape(self, values: dict[str, Any]) -> int | None: ...
def update_tape(self, tape_id: int, values: dict[str, Any]) -> bool: ...
def delete_tape(self, tape_id: int) -> None: ...
def tape(self, tape_id: int) -> sqlite3.Row | None: ...
def profile_tapes(self, profile_id: int) -> list[sqlite3.Row]: ...
def set_fat_display(self, profile_id: int, value: str) -> None: ...

def navy_fat_pct(
    *,
    male: bool,
    height_cm: float,
    waist_cm: float | None,
    neck_cm: float | None,
    hip_cm: float | None,
) -> float | None: ...

def tenth(value: float) -> float: ...
def unvalidated_mean(scale_pct: float, tape_pct: float) -> float: ...
def nearest_tape(
    weigh_at: datetime, tapes: list[tuple[datetime, bool]]
) -> datetime | None: ...

def fat_series(
    *,
    male: bool,
    height_cm: float,
    weighs: list[tuple[datetime, float | None]],
    tapes: list[tuple[datetime, float | None, float | None, float | None]],
    start: datetime,
    end: datetime,
) -> FatSeries: ...

def fat_headline(
    *,
    mode: str,
    male: bool,
    height_cm: float,
    weighs: list[tuple[datetime, float, float | None, float | None]],
    tapes: list[tuple[datetime, float | None, float | None, float | None]],
) -> FatHeadline: ...

def parse_wall_clock(
    text: str, *, now: datetime, tz: datetime.tzinfo | None = None
) -> tuple[datetime, int] | str: ...
def wall_input(measured_at: str, utc_offset_s: int) -> str: ...
def local_minute_value(now: datetime) -> str: ...

def fat_chart(
    series: list[tuple[str, str, list[tuple[datetime, float]]]],
    *,
    start: datetime,
    end: datetime,
    breaks: list[datetime],
) -> str: ...
```

`weighs` for `fat_series` are `(measured_at, fat_pct)`. `tapes` are `(measured_at, waist_cm, neck_cm, hip_cm)`. `weighs` for `fat_headline` are `(measured_at, weight_kg, fat_pct, lean_kg)`. Instants are timezone-aware. `FatSeries` fields are `scale`, `tape`, `mean`, and `breaks`, each a list. `FatHeadline` fields are `number: float | None`, `caption: str`, `detail: str`, `lean_kg: float | None`, `lean_caption: str`, `show_jebb_error: bool`.

`parse_wall_clock` returns `(utc_instant, utc_offset_s)` or one of `Date and time are required.`, `That time does not exist in this timezone.`, `That time is in the future.` `tz=None` uses `datetime.now().astimezone().tzinfo`.

`fat_chart` series items are `(legend, tone, points)` with tones `fat`, `lean`, and `mean`, in that order. Breaks split only the `mean` tone.

---

### Task 1: Tape storage and fat_display migration

**Files:**
- Modify: `apps/bathroom-hub/src/bathroom_hub/db.py`
- Test: `apps/bathroom-hub/tests/test_bathroom_hub.py`

**Interfaces:**
- Consumes: existing `Database`, `ME` profile dict in the test module.
- Produces: `normalize_fat_display`, `insert_tape`, `update_tape`, `delete_tape`, `tape`, `profile_tapes`, `set_fat_display`.

- [ ] **Step 1: Write the failing test**

Add this class to `apps/bathroom-hub/tests/test_bathroom_hub.py`:

```python
class TapeStoreTests(unittest.TestCase):
    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.db = Database(Path(self.tmp.name) / "hub.db")

    def tearDown(self):
        self.db.conn.close()
        self.tmp.cleanup()

    def test_tape_round_trip_orders_newest_first(self):
        me = self.db.create_profile(ME)
        older = self.db.insert_tape(
            {
                "profile_id": me,
                "measured_at": "2020-01-01T18:00:00+00:00",
                "utc_offset_s": 0,
                "waist_cm": 90,
                "neck_cm": 38,
                "hip_cm": None,
                "received_at": "2020-01-01T18:00:01+00:00",
            }
        )
        newer = self.db.insert_tape(
            {
                "profile_id": me,
                "measured_at": "2020-01-02T18:00:00+00:00",
                "utc_offset_s": 0,
                "waist_cm": None,
                "neck_cm": 38,
                "hip_cm": 100,
                "received_at": "2020-01-02T18:00:01+00:00",
            }
        )
        self.assertIsNotNone(older)
        self.assertEqual(
            [row["id"] for row in self.db.profile_tapes(me)], [newer, older]
        )
        self.assertEqual(self.db.tape(older)["waist_cm"], 90)
        self.assertTrue(
            self.db.update_tape(
                older,
                {
                    "measured_at": "2020-01-01T19:00:00+00:00",
                    "utc_offset_s": 0,
                    "waist_cm": 91,
                    "neck_cm": 38,
                    "hip_cm": None,
                },
            )
        )
        self.assertEqual(self.db.tape(older)["waist_cm"], 91)

    def test_duplicate_instant_and_empty_tape_are_rejected(self):
        me = self.db.create_profile(ME)
        values = {
            "profile_id": me,
            "measured_at": "2020-01-02T18:00:00+00:00",
            "utc_offset_s": 0,
            "waist_cm": 90,
            "neck_cm": None,
            "hip_cm": None,
            "received_at": "2020-01-02T18:00:01+00:00",
        }
        self.assertIsNotNone(self.db.insert_tape(values))
        self.assertIsNone(self.db.insert_tape(values))
        self.assertFalse(
            self.db.update_tape(
                999,
                {
                    "measured_at": "2020-01-03T18:00:00+00:00",
                    "utc_offset_s": 0,
                    "waist_cm": 90,
                    "neck_cm": None,
                    "hip_cm": None,
                },
            )
        )
        empty = dict(values, measured_at="2020-01-04T18:00:00+00:00")
        empty["waist_cm"] = None
        self.assertIsNone(self.db.insert_tape(empty))

    def test_deleting_a_person_deletes_tapes_and_keeps_weighs(self):
        me = self.db.create_profile(ME)
        tape_id = self.db.insert_tape(
            {
                "profile_id": me,
                "measured_at": "2020-01-02T18:00:00+00:00",
                "utc_offset_s": 0,
                "waist_cm": 90,
                "neck_cm": 38,
                "hip_cm": None,
                "received_at": "2020-01-02T18:00:01+00:00",
            }
        )
        self.db.insert_measurement(
            {
                "measured_at": "2020-01-02T18:00:00+00:00",
                "utc_offset_s": 0,
                "weight_kg": 97.5,
                "raw_weight": "aa",
                "received_at": "2020-01-02T18:00:01+00:00",
                "profile_id": me,
                "fat_pct": 27.4,
                "lean_kg": 70.8,
                "bmi": 29.1,
            }
        )
        self.db.delete_profile(me)
        self.assertIsNone(self.db.tape(tape_id))
        self.assertEqual(len(self.db.measurements()), 1)
        self.assertIsNone(self.db.measurements()[0]["profile_id"])

    def test_existing_database_defaults_fat_display_to_scale(self):
        path = Path(self.tmp.name) / "old.db"
        conn = sqlite3.connect(path)
        conn.execute(
            """CREATE TABLE profiles (
                id INTEGER PRIMARY KEY,
                name TEXT NOT NULL,
                sex TEXT NOT NULL,
                birth_date TEXT NOT NULL,
                height_cm REAL NOT NULL,
                start_weight_kg REAL,
                created_at TEXT NOT NULL
            )"""
        )
        conn.execute(
            "INSERT INTO profiles (name, sex, birth_date, height_cm, created_at) "
            "VALUES ('Me', 'male', '1990-01-01', 183, '2020-01-01T00:00:00+00:00')"
        )
        conn.commit()
        conn.close()
        db = Database(path)
        self.addCleanup(db.conn.close)
        self.assertEqual(db.profile(1)["fat_display"], "scale")
        db.conn.execute("UPDATE profiles SET fat_display = 'nope' WHERE id = 1")
        db.conn.commit()
        self.assertEqual(
            db.normalize_fat_display(db.profile(1)["fat_display"]), "scale"
        )
        db.set_fat_display(1, "tape")
        self.assertEqual(db.profile(1)["fat_display"], "tape")
        with self.assertRaises(ValueError):
            db.set_fat_display(1, "nope")
        self.assertEqual(db.profile(1)["fat_display"], "tape")
```

Add `import sqlite3` to the test imports.

- [ ] **Step 2: Run the test to verify it fails**

From `apps/bathroom-hub`:

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.TapeStoreTests -v
```

Expected: FAIL. `Database` has no attribute `insert_tape`.

- [ ] **Step 3: Write the minimal implementation**

Append this table to `SCHEMA` in `db.py`, after the `events` table:

```python
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

In `Database.__init__`, after `executescript` and before `commit`:

```python
columns = {
    row[1] for row in self.conn.execute("PRAGMA table_info(profiles)")
}
if "fat_display" not in columns:
    self.conn.execute(
        "ALTER TABLE profiles "
        "ADD COLUMN fat_display TEXT NOT NULL DEFAULT 'scale'"
    )
```

Add the module function and methods:

```python
def normalize_fat_display(value: str | None) -> str:
    if value in ("scale", "tape", "average"):
        return value
    return "scale"


class Database:
    def normalize_fat_display(self, value: str | None) -> str:
        return normalize_fat_display(value)

    def set_fat_display(self, profile_id: int, value: str) -> None:
        if value not in ("scale", "tape", "average"):
            raise ValueError(value)
        self.conn.execute(
            "UPDATE profiles SET fat_display = ? WHERE id = ?",
            (value, profile_id),
        )
        self.conn.commit()

    def insert_tape(self, values: dict[str, Any]) -> int | None:
        columns = list(values)
        try:
            cur = self.conn.execute(
                f"INSERT INTO tape_measures ({', '.join(columns)}) "
                f"VALUES ({', '.join('?' for _ in columns)})",
                [values[column] for column in columns],
            )
        except sqlite3.IntegrityError:
            return None
        self.conn.commit()
        return int(cur.lastrowid)

    def update_tape(self, tape_id: int, values: dict[str, Any]) -> bool:
        if self.tape(tape_id) is None:
            return False
        sets = ", ".join(f"{column} = ?" for column in values)
        try:
            self.conn.execute(
                f"UPDATE tape_measures SET {sets} WHERE id = ?",
                [*values.values(), tape_id],
            )
        except sqlite3.IntegrityError:
            return False
        self.conn.commit()
        return True

    def delete_tape(self, tape_id: int) -> None:
        self.conn.execute("DELETE FROM tape_measures WHERE id = ?", (tape_id,))
        self.conn.commit()

    def tape(self, tape_id: int) -> sqlite3.Row | None:
        return self.conn.execute(
            "SELECT * FROM tape_measures WHERE id = ?", (tape_id,)
        ).fetchone()

    def profile_tapes(self, profile_id: int) -> list[sqlite3.Row]:
        return self.conn.execute(
            "SELECT * FROM tape_measures WHERE profile_id = ? "
            "ORDER BY measured_at DESC",
            (profile_id,),
        ).fetchall()
```

Do not add `fat_display` to `PROFILE_FIELDS`.

- [ ] **Step 4: Run the test to verify it passes**

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.TapeStoreTests -v
```

Expected: PASS, 4 tests.

- [ ] **Step 5: Commit**

```bash
git add apps/bathroom-hub/src/bathroom_hub/db.py apps/bathroom-hub/tests/test_bathroom_hub.py
git commit -m "Add tape storage and the fat display column."
```

---

### Task 2: Navy estimate, unvalidated mean, and nearest tape

**Files:**
- Modify: `apps/bathroom-hub/src/bathroom_hub/bodycomp.py`
- Test: `apps/bathroom-hub/tests/test_bathroom_hub.py`

**Interfaces:**
- Consumes: `MIN_FAT_PCT` and `MAX_FAT_PCT` already in `bodycomp.py`.
- Produces: `navy_fat_pct`, `tenth`, `unvalidated_mean`, `nearest_tape`.

- [ ] **Step 1: Write the failing test**

Add to `BodyCompositionTests`:

```python
def test_navy_references_and_limits(self):
    self.assertAlmostEqual(
        bodycomp.navy_fat_pct(
            male=True,
            height_cm=70 * 2.54,
            waist_cm=36 * 2.54,
            neck_cm=16 * 2.54,
            hip_cm=None,
        ),
        19.43,
        places=2,
    )
    self.assertAlmostEqual(
        bodycomp.navy_fat_pct(
            male=False,
            height_cm=64 * 2.54,
            waist_cm=30 * 2.54,
            neck_cm=13 * 2.54,
            hip_cm=38 * 2.54,
        ),
        29.21,
        places=2,
    )
    self.assertAlmostEqual(
        bodycomp.navy_fat_pct(
            male=True, height_cm=183, waist_cm=92, neck_cm=38, hip_cm=None
        ),
        20.83,
        places=2,
    )
    self.assertAlmostEqual(
        bodycomp.navy_fat_pct(
            male=False,
            height_cm=170,
            waist_cm=80,
            neck_cm=33,
            hip_cm=100,
        ),
        30.93,
        places=2,
    )
    bare = bodycomp.navy_fat_pct(
        male=True, height_cm=183, waist_cm=92, neck_cm=38, hip_cm=None
    )
    with_hip = bodycomp.navy_fat_pct(
        male=True, height_cm=183, waist_cm=92, neck_cm=38, hip_cm=100
    )
    self.assertEqual(bare, with_hip)
    self.assertIsNone(
        bodycomp.navy_fat_pct(
            male=False, height_cm=170, waist_cm=80, neck_cm=33, hip_cm=None
        )
    )
    self.assertIsNone(
        bodycomp.navy_fat_pct(
            male=True, height_cm=183, waist_cm=40, neck_cm=40, hip_cm=None
        )
    )
    self.assertIsNone(
        bodycomp.navy_fat_pct(
            male=True, height_cm=183, waist_cm=40, neck_cm=39, hip_cm=None
        )
    )
    self.assertIsNone(
        bodycomp.navy_fat_pct(
            male=True, height_cm=140, waist_cm=200, neck_cm=25, hip_cm=None
        )
    )

def test_unvalidated_mean_rounds_half_away_from_zero(self):
    self.assertEqual(bodycomp.unvalidated_mean(27.4, 20.8), 24.1)
    self.assertEqual(bodycomp.unvalidated_mean(27.5, 20.8), 24.2)

def test_nearest_tape_uses_a_24_hour_window(self):
    weigh = datetime(2026, 1, 2, 7, 0, tzinfo=UTC)
    evening = datetime(2026, 1, 1, 20, 0, tzinfo=UTC)
    self.assertEqual(
        bodycomp.nearest_tape(weigh, [(evening, True)]), evening
    )
    edge = weigh - timedelta(seconds=86400)
    outside = weigh - timedelta(seconds=86401)
    self.assertEqual(bodycomp.nearest_tape(weigh, [(edge, True)]), edge)
    self.assertIsNone(bodycomp.nearest_tape(weigh, [(outside, True)]))
    before = weigh - timedelta(hours=10)
    after = weigh + timedelta(hours=10)
    self.assertEqual(
        bodycomp.nearest_tape(weigh, [(after, True), (before, True)]), before
    )
    nearer = weigh - timedelta(hours=2)
    farther = weigh - timedelta(hours=5)
    self.assertEqual(
        bodycomp.nearest_tape(weigh, [(farther, True), (nearer, False)]),
        farther,
    )
    tape = weigh + timedelta(minutes=90)
    later = weigh + timedelta(hours=3)
    self.assertEqual(bodycomp.nearest_tape(weigh, [(tape, True)]), tape)
    self.assertEqual(bodycomp.nearest_tape(later, [(tape, True)]), tape)
```

Add `timedelta` to the datetime import in the test file.

- [ ] **Step 2: Run the test to verify it fails**

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.BodyCompositionTests.test_navy_references_and_limits tests.test_bathroom_hub.BodyCompositionTests.test_unvalidated_mean_rounds_half_away_from_zero tests.test_bathroom_hub.BodyCompositionTests.test_nearest_tape_uses_a_24_hour_window -v
```

Expected: FAIL. `bodycomp` has no attribute `navy_fat_pct`.

- [ ] **Step 3: Write the minimal implementation**

Add to `bodycomp.py`:

```python
TAPE_WINDOW_S = 86400


def tenth(value: float) -> float:
    quantized = Decimal(str(value)).quantize(
        Decimal("0.1"), rounding=ROUND_HALF_UP
    )
    return float(quantized)


def unvalidated_mean(scale_pct: float, tape_pct: float) -> float:
    total = Decimal(str(tenth(scale_pct))) + Decimal(str(tenth(tape_pct)))
    return float((total / Decimal(2)).quantize(Decimal("0.1"), rounding=ROUND_HALF_UP))


def navy_fat_pct(
    *,
    male: bool,
    height_cm: float,
    waist_cm: float | None,
    neck_cm: float | None,
    hip_cm: float | None,
) -> float | None:
    if (
        height_cm <= 0
        or waist_cm is None
        or neck_cm is None
        or waist_cm <= 0
        or neck_cm <= 0
    ):
        return None
    height_in = height_cm / 2.54
    waist_in = waist_cm / 2.54
    neck_in = neck_cm / 2.54
    if male:
        gap = waist_in - neck_in
        if gap <= 0:
            return None
        fat = (
            86.010 * math.log10(gap)
            - 70.041 * math.log10(height_in)
            + 36.76
        )
    else:
        if hip_cm is None or hip_cm <= 0:
            return None
        gap = waist_in + (hip_cm / 2.54) - neck_in
        if gap <= 0:
            return None
        fat = (
            163.205 * math.log10(gap)
            - 97.684 * math.log10(height_in)
            - 78.387
        )
    if fat < MIN_FAT_PCT or fat > MAX_FAT_PCT:
        return None
    return fat


def nearest_tape(
    weigh_at: datetime, tapes: list[tuple[datetime, bool]]
) -> datetime | None:
    best: datetime | None = None
    best_gap: float | None = None
    for measured_at, complete in tapes:
        if not complete:
            continue
        gap = abs((measured_at - weigh_at).total_seconds())
        if gap > TAPE_WINDOW_S:
            continue
        if (
            best is None
            or gap < best_gap
            or (gap == best_gap and measured_at < best)
        ):
            best = measured_at
            best_gap = gap
    return best
```

At the top of `bodycomp.py`, change `from datetime import date` to `from datetime import date, datetime`, and add `from decimal import ROUND_HALF_UP, Decimal` next to the other stdlib imports. Move those two import lines out of the snippet below if they were pasted with the functions. Keep `jebb2000_fat_pct` and `composition` unchanged, including their clamp.

- [ ] **Step 4: Run the test to verify it passes**

Run the same command as Step 2. Expected: PASS, 3 tests. Then run the existing Jebb tests:

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.BodyCompositionTests -v
```

Expected: PASS, including `test_jebb_matches_the_comparison_run`.

- [ ] **Step 5: Commit**

```bash
git add apps/bathroom-hub/src/bathroom_hub/bodycomp.py apps/bathroom-hub/tests/test_bathroom_hub.py
git commit -m "Add the Navy estimate, the unvalidated mean, and tape pairing."
```

---

### Task 3: Series points and the main number

**Files:**
- Modify: `apps/bathroom-hub/src/bathroom_hub/bodycomp.py`
- Test: `apps/bathroom-hub/tests/test_bathroom_hub.py`

**Interfaces:**
- Consumes: `navy_fat_pct`, `tenth`, `unvalidated_mean`, `nearest_tape`, `TAPE_WINDOW_S`.
- Produces: `FatSeries`, `fat_series`, `FatHeadline`, `fat_headline`.

- [ ] **Step 1: Write the failing test**

Add to `BodyCompositionTests`:

```python
def test_fat_series_keeps_each_line_on_its_own_instant(self):
    start = datetime(2026, 1, 10, tzinfo=UTC)
    end = datetime(2026, 1, 20, tzinfo=UTC)
    weigh = datetime(2026, 1, 12, 7, 0, tzinfo=UTC)
    tape_at = datetime(2026, 1, 11, 20, 0, tzinfo=UTC)
    outside = datetime(2026, 1, 21, 7, 0, tzinfo=UTC)
    series = bodycomp.fat_series(
        male=True,
        height_cm=183,
        weighs=[(weigh, 27.4), (outside, 27.4)],
        tapes=[(tape_at, 92, 38, None)],
        start=start,
        end=end,
    )
    self.assertEqual(series.scale, [(weigh, 27.4)])
    tape_pct = bodycomp.tenth(
        bodycomp.navy_fat_pct(
            male=True, height_cm=183, waist_cm=92, neck_cm=38, hip_cm=None
        )
    )
    self.assertEqual(series.tape, [(tape_at, tape_pct)])
    self.assertEqual(series.mean, [(weigh, 24.1)])
    early_weigh = start + timedelta(hours=1)
    early_tape = start - timedelta(hours=11)
    early = bodycomp.fat_series(
        male=True,
        height_cm=183,
        weighs=[(early_weigh, 27.4)],
        tapes=[(early_tape, 92, 38, None)],
        start=start,
        end=end,
    )
    self.assertEqual(early.tape, [])
    self.assertEqual(early.mean, [(early_weigh, 24.1)])
    self.assertIn(weigh, series.breaks)
    self.assertIn(tape_at, series.breaks)

def test_headline_follows_the_selected_figure(self):
    weigh = datetime(2026, 1, 12, 7, 0, tzinfo=UTC)
    tape_at = datetime(2026, 1, 11, 20, 0, tzinfo=UTC)
    weighs = [(weigh, 97.5, 27.4, 70.8)]
    tapes = [(tape_at, 92.0, 38.0, None)]
    scale = bodycomp.fat_headline(
        mode="scale", male=True, height_cm=183, weighs=weighs, tapes=tapes
    )
    self.assertEqual(scale.number, 27.4)
    self.assertTrue(scale.show_jebb_error)
    self.assertEqual(scale.lean_kg, 70.8)
    tape = bodycomp.fat_headline(
        mode="tape", male=True, height_cm=183, weighs=weighs, tapes=tapes
    )
    self.assertEqual(
        tape.number,
        bodycomp.tenth(
            bodycomp.navy_fat_pct(
                male=True, height_cm=183, waist_cm=92, neck_cm=38, hip_cm=None
            )
        ),
    )
    self.assertEqual(tape.caption, "Tape estimate. It is not the scale reading.")
    self.assertFalse(tape.show_jebb_error)
    self.assertEqual(tape.lean_caption, "From the tape estimate.")
    self.assertEqual(tape.lean_kg, bodycomp.tenth(97.5 * (1 - tape.number / 100)))
    average = bodycomp.fat_headline(
        mode="average", male=True, height_cm=183, weighs=weighs, tapes=tapes
    )
    self.assertEqual(average.number, 24.1)
    self.assertEqual(average.caption, "Average of scale and tape estimates.")
    self.assertEqual(
        average.detail,
        "Unvalidated mean of the two estimates. Not a more accurate measurement.",
    )
    self.assertNotIn("more accurate", average.caption)
    self.assertEqual(
        average.lean_caption, "From the unvalidated mean. Not the stored lean mass."
    )
    self.assertEqual(average.lean_kg, bodycomp.tenth(97.5 * (1 - 24.1 / 100)))

def test_average_does_not_fall_back_or_skip_a_farther_complete_tape(self):
    weigh = datetime(2026, 1, 12, 7, 0, tzinfo=UTC)
    missing = bodycomp.fat_headline(
        mode="average",
        male=True,
        height_cm=183,
        weighs=[(weigh, 97.5, 27.4, 70.8)],
        tapes=[],
    )
    self.assertIsNone(missing.number)
    self.assertEqual(missing.caption, "No tape estimate within 24 hours.")
    hip = bodycomp.fat_headline(
        mode="average",
        male=False,
        height_cm=170,
        weighs=[(weigh, 70.0, 30.0, 49.0)],
        tapes=[(weigh, 80.0, 33.0, None)],
    )
    self.assertIsNone(hip.number)
    self.assertIn("Hip is missing", hip.caption)
    older = weigh - timedelta(hours=10)
    paired = bodycomp.fat_headline(
        mode="average",
        male=False,
        height_cm=170,
        weighs=[(weigh, 70.0, 30.0, 49.0)],
        tapes=[(weigh, 80.0, 33.0, None), (older, 80.0, 33.0, 100.0)],
    )
    self.assertIsNotNone(paired.number)
    bare = bodycomp.fat_headline(
        mode="tape",
        male=True,
        height_cm=183,
        weighs=[],
        tapes=[(weigh, 92.0, 38.0, None)],
    )
    self.assertIsNone(bare.lean_kg)
    self.assertEqual(bare.lean_caption, "Lean mass needs a weigh within 24 hours.")
    early = weigh - timedelta(hours=1)
    late = weigh + timedelta(hours=1)
    chosen = bodycomp.fat_headline(
        mode="tape",
        male=True,
        height_cm=183,
        weighs=[(late, 90.0, 20.0, 72.0), (early, 97.5, 27.4, 70.8)],
        tapes=[(weigh, 92.0, 38.0, None)],
    )
    self.assertEqual(
        chosen.lean_kg, bodycomp.tenth(97.5 * (1 - chosen.number / 100))
    )
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.BodyCompositionTests.test_fat_series_keeps_each_line_on_its_own_instant tests.test_bathroom_hub.BodyCompositionTests.test_headline_follows_the_selected_figure tests.test_bathroom_hub.BodyCompositionTests.test_average_does_not_fall_back_or_skip_a_farther_complete_tape -v
```

Expected: FAIL. `bodycomp` has no attribute `fat_series`.

- [ ] **Step 3: Write the minimal implementation**

Add to `bodycomp.py`:

```python
@dataclass(frozen=True)
class FatSeries:
    scale: list[tuple[datetime, float]]
    tape: list[tuple[datetime, float]]
    mean: list[tuple[datetime, float]]
    breaks: list[datetime]


@dataclass(frozen=True)
class FatHeadline:
    number: float | None
    caption: str
    detail: str
    lean_kg: float | None
    lean_caption: str
    show_jebb_error: bool


def _complete(male: bool, waist, neck, hip) -> bool:
    if waist is None or neck is None:
        return False
    return True if male else hip is not None


def fat_series(*, male, height_cm, weighs, tapes, start, end) -> FatSeries:
    scale = [
        (at, fat)
        for at, fat in weighs
        if fat is not None and start <= at <= end
    ]
    tape_points = []
    complete_at = []
    by_instant = {}
    for at, waist, neck, hip in tapes:
        by_instant[at] = (waist, neck, hip)
        if not _complete(male, waist, neck, hip):
            continue
        fat = navy_fat_pct(
            male=male,
            height_cm=height_cm,
            waist_cm=waist,
            neck_cm=neck,
            hip_cm=hip,
        )
        if fat is None:
            continue
        complete_at.append(at)
        if start <= at <= end:
            tape_points.append((at, tenth(fat)))
    flags = [(at, True) for at in complete_at]
    mean = []
    for at, fat in weighs:
        if fat is None or not (start <= at <= end):
            continue
        chosen = nearest_tape(at, flags)
        if chosen is None:
            continue
        waist, neck, hip = by_instant[chosen]
        tape_fat = navy_fat_pct(
            male=male,
            height_cm=height_cm,
            waist_cm=waist,
            neck_cm=neck,
            hip_cm=hip,
        )
        mean.append((at, unvalidated_mean(fat, tape_fat)))
    breaks = [at for at, _fat in weighs] + complete_at
    return FatSeries(scale, tape_points, mean, breaks)


def fat_headline(*, mode, male, height_cm, weighs, tapes) -> FatHeadline:
    ordered = sorted(weighs, key=lambda item: item[0])
    latest = ordered[-1] if ordered else None
    complete = []
    for at, waist, neck, hip in tapes:
        fat = None
        if _complete(male, waist, neck, hip):
            fat = navy_fat_pct(
                male=male,
                height_cm=height_cm,
                waist_cm=waist,
                neck_cm=neck,
                hip_cm=hip,
            )
        if fat is not None:
            complete.append((at, tenth(fat)))
    if mode == "tape":
        if not complete:
            return FatHeadline(
                None,
                "Add a waist and neck measurement. It stays on this hub.",
                "",
                None,
                "",
                False,
            )
        at, number = max(complete, key=lambda item: item[0])
        nearest = _nearest_weigh(at, ordered)
        if nearest is None:
            lean, lean_caption = None, "Lean mass needs a weigh within 24 hours."
        else:
            lean = tenth(nearest[1] * (1 - number / 100))
            lean_caption = "From the tape estimate."
        return FatHeadline(
            number,
            "Tape estimate. It is not the scale reading.",
            "",
            lean,
            lean_caption,
            False,
        )
    if mode == "average":
        if latest is None or latest[2] is None:
            return FatHeadline(
                None, "No tape estimate within 24 hours.", "", None, "", False
            )
        chosen = nearest_tape(
            latest[0], [(at, True) for at, _fat in complete]
        )
        if chosen is None:
            return FatHeadline(
                None, _missing_tape_caption(male, latest[0], tapes), "", None, "", False
            )
        tape_fat = dict(complete)[chosen]
        number = unvalidated_mean(latest[2], tape_fat)
        return FatHeadline(
            number,
            "Average of scale and tape estimates.",
            "Unvalidated mean of the two estimates. Not a more accurate measurement.",
            tenth(latest[1] * (1 - number / 100)),
            "From the unvalidated mean. Not the stored lean mass.",
            False,
        )
    if latest is None or latest[2] is None:
        return FatHeadline(
            None,
            "Needs a barefoot weigh. The scale measures impedance through your feet.",
            "",
            None,
            "",
            False,
        )
    return FatHeadline(latest[2], "", "", latest[3], "", True)


def _nearest_weigh(tape_at, weighs):
    best = None
    best_gap = None
    for item in weighs:
        gap = abs((item[0] - tape_at).total_seconds())
        if gap > TAPE_WINDOW_S:
            continue
        if best is None or gap < best_gap or (gap == best_gap and item[0] < best[0]):
            best = item
            best_gap = gap
    return best


def _missing_tape_caption(male, weigh_at, tapes) -> str:
    in_range = []
    for at, waist, neck, hip in tapes:
        if abs((at - weigh_at).total_seconds()) <= TAPE_WINDOW_S:
            in_range.append((at, waist, neck, hip))
    if not male and in_range:
        nearest = min(
            in_range,
            key=lambda item: (
                abs((item[0] - weigh_at).total_seconds()),
                item[0],
            ),
        )
        _at, waist, neck, hip = nearest
        if waist is not None and neck is not None and hip is None:
            return "Hip is missing. It is needed by the Navy equation for women."
    return "No tape estimate within 24 hours."
```

- [ ] **Step 4: Run the test to verify it passes**

Run the Step 2 command. Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

```bash
git add apps/bathroom-hub/src/bathroom_hub/bodycomp.py apps/bathroom-hub/tests/test_bathroom_hub.py
git commit -m "Derive body-fat series and the selected headline."
```

---

### Task 4: Wall-clock parsing

**Files:**
- Modify: `apps/bathroom-hub/src/bathroom_hub/hub.py`
- Test: `apps/bathroom-hub/tests/test_bathroom_hub.py`

**Interfaces:**
- Consumes: nothing from later tasks.
- Produces: `parse_wall_clock`, `wall_input`, `local_minute_value` on the `hub` module.

- [ ] **Step 1: Write the failing test**

Add `from zoneinfo import ZoneInfo` to the test imports and this class:

```python
class WallClockTests(unittest.TestCase):
    def test_wall_input_uses_the_stored_offset(self):
        self.assertEqual(
            hub_module.wall_input("2020-01-02T18:00:00+00:00", 7200),
            "2020-01-02T20:00",
        )

    def test_parse_stores_utc_and_rejects_bad_values(self):
        now = datetime(2026, 6, 1, 12, 0, tzinfo=UTC)
        parsed = hub_module.parse_wall_clock("2020-01-02T20:00", now=now)
        self.assertIsInstance(parsed, tuple)
        instant, offset = parsed
        self.assertEqual(instant.second, 0)
        zone = datetime.now().astimezone().tzinfo
        aware = datetime(2020, 1, 2, 20, 0, tzinfo=zone)
        self.assertEqual(instant, aware.astimezone(UTC).replace(microsecond=0))
        self.assertEqual(offset, int(aware.utcoffset().total_seconds()))
        self.assertEqual(
            hub_module.parse_wall_clock("2020-01-02", now=now),
            "Date and time are required.",
        )
        self.assertEqual(
            hub_module.parse_wall_clock("2099-01-01T00:00", now=now),
            "That time is in the future.",
        )

    def test_gap_is_rejected_and_the_fold_uses_the_first_offset(self):
        ny = ZoneInfo("America/New_York")
        now = datetime(2026, 6, 1, 12, 0, tzinfo=UTC)
        self.assertEqual(
            hub_module.parse_wall_clock(
                "2026-03-08T02:30", now=now, tz=ny
            ),
            "That time does not exist in this timezone.",
        )
        parsed = hub_module.parse_wall_clock(
            "2026-11-01T01:30", now=now, tz=ny
        )
        instant, offset = parsed
        self.assertEqual(offset, -14400)
        self.assertEqual(instant, datetime(2026, 11, 1, 5, 30, tzinfo=UTC))

    def test_local_minute_value(self):
        now = datetime(2026, 6, 1, 12, 34, 59, tzinfo=UTC)
        self.assertEqual(
            hub_module.local_minute_value(now),
            now.astimezone().strftime("%Y-%m-%dT%H:%M"),
        )
```

Import the module as `from bathroom_hub import hub as hub_module` so it is not shadowed by `Hub` tests. If `hub` is already imported as a class module, use that name consistently. The test file currently imports `from bathroom_hub.hub import Hub, scale_time`. Add `import bathroom_hub.hub as hub_module`.

- [ ] **Step 2: Run the test to verify it fails**

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.WallClockTests -v
```

Expected: FAIL. `hub_module` has no attribute `wall_input`.

- [ ] **Step 3: Write the minimal implementation**

Add to `hub.py`:

```python
import re

_WALL = re.compile(r"^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}$")


def local_minute_value(now: datetime) -> str:
    return now.astimezone().strftime("%Y-%m-%dT%H:%M")


def wall_input(measured_at: str, utc_offset_s: int) -> str:
    instant = datetime.fromisoformat(measured_at).astimezone(UTC)
    wall = instant + timedelta(seconds=int(utc_offset_s))
    return wall.strftime("%Y-%m-%dT%H:%M")


def parse_wall_clock(
    text: str, *, now: datetime, tz: datetime.tzinfo | None = None
) -> tuple[datetime, int] | str:
    text = text.strip()
    if not _WALL.match(text):
        return "Date and time are required."
    if tz is None:
        tz = datetime.now().astimezone().tzinfo
    naive = datetime.fromisoformat(text)
    aware = naive.replace(tzinfo=tz, fold=0)
    instant = aware.astimezone(UTC).replace(second=0, microsecond=0)
    back = instant.astimezone(tz)
    if (back.year, back.month, back.day, back.hour, back.minute) != (
        naive.year,
        naive.month,
        naive.day,
        naive.hour,
        naive.minute,
    ):
        return "That time does not exist in this timezone."
    current_minute = now.astimezone(UTC).replace(second=0, microsecond=0)
    if instant > current_minute:
        return "That time is in the future."
    offset = int(aware.utcoffset().total_seconds()) if aware.utcoffset() else 0
    return instant, offset
```

A failed form redisplays the submitted string. That happens in Task 6; this function only returns the error string.

- [ ] **Step 4: Run the test to verify it passes**

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.WallClockTests -v
```

Expected: PASS, 4 tests. Also run `tests.test_bathroom_hub.HubTests.test_scale_time_is_local_wall_clock`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/bathroom-hub/src/bathroom_hub/hub.py apps/bathroom-hub/tests/test_bathroom_hub.py
git commit -m "Parse tape date and time in the hub's local zone."
```

---

### Task 5: Three-line body-fat chart

**Files:**
- Modify: `apps/bathroom-hub/src/bathroom_hub/charts.py`
- Modify: `apps/bathroom-hub/src/bathroom_hub/static/app.css`
- Test: `apps/bathroom-hub/tests/test_bathroom_hub.py`

**Interfaces:**
- Consumes: `_scale` in `charts.py`.
- Produces: `fat_chart`. Does not change `line_chart` or `sparkline`.

- [ ] **Step 1: Write the failing test**

Add to `ChartAndCatalogueTests`:

```python
def test_fat_chart_draws_three_labeled_lines(self):
    start = datetime(2026, 1, 1, tzinfo=UTC)
    end = datetime(2026, 1, 31, tzinfo=UTC)
    scale = [
        (datetime(2026, 1, 2, tzinfo=UTC), 27.4),
        (datetime(2026, 1, 3, tzinfo=UTC), 27.0),
        (datetime(2026, 1, 10, tzinfo=UTC), 26.0),
        (datetime(2026, 1, 11, tzinfo=UTC), 26.2),
        (datetime(2026, 1, 12, tzinfo=UTC), 21.0),
    ]
    tape = [
        (datetime(2026, 1, 2, 1, tzinfo=UTC), 20.8),
        (datetime(2026, 1, 11, 1, tzinfo=UTC), 20.0),
    ]
    mean = [
        (datetime(2026, 1, 2, tzinfo=UTC), 24.1),
        (datetime(2026, 1, 3, tzinfo=UTC), 23.9),
        (datetime(2026, 1, 11, tzinfo=UTC), 23.1),
        (datetime(2026, 1, 12, tzinfo=UTC), 20.5),
    ]
    unpaired = datetime(2026, 1, 7, tzinfo=UTC)
    html = charts.fat_chart(
        [
            ("Scale estimate", "fat", scale),
            ("Tape estimate", "lean", tape),
            (
                "Average of scale and tape estimates. Unvalidated mean.",
                "mean",
                mean,
            ),
        ],
        start=start,
        end=end,
        breaks=[unpaired],
    )
    self.assertIn("Scale estimate", html)
    self.assertIn("Tape estimate", html)
    self.assertIn("Average of scale and tape estimates. Unvalidated mean.", html)
    self.assertIn("unvalidated mean", html)
    self.assertEqual(html.count("stroke-dasharray"), 2)
    self.assertIn('class="tone-fat"', html)
    self.assertIn('class="tone-lean"', html)
    self.assertIn('class="tone-mean"', html)
    self.assertIn("2 points", html)
    self.assertIn("4 points", html)

def test_fat_chart_names_an_empty_series(self):
    start = datetime(2026, 1, 1, tzinfo=UTC)
    end = datetime(2026, 1, 31, tzinfo=UTC)
    html = charts.fat_chart(
        [
            ("Scale estimate", "fat", []),
            ("Tape estimate", "lean", [(datetime(2026, 1, 2, tzinfo=UTC), 20.8)]),
            (
                "Average of scale and tape estimates. Unvalidated mean.",
                "mean",
                [],
            ),
        ],
        start=start,
        end=end,
        breaks=[],
    )
    self.assertEqual(html.count("none in this range"), 2)
    self.assertNotIn("<polyline", html)
    self.assertNotIn("stroke-dasharray", html)

def test_weight_line_chart_stays_one_series(self):
    start = datetime(2026, 9, 14, tzinfo=UTC)
    end = datetime(2026, 10, 14, tzinfo=UTC)
    svg = charts.line_chart(
        [(start, 99.0), (end, 97.5)],
        start=start,
        end=end,
        tone="weight",
        label="Weight over 30 days",
    )
    self.assertEqual(svg.count("<polyline"), 1)
    self.assertNotIn("Tape estimate", svg)
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.ChartAndCatalogueTests.test_fat_chart_draws_three_labeled_lines tests.test_bathroom_hub.ChartAndCatalogueTests.test_fat_chart_names_an_empty_series -v
```

Expected: FAIL. `charts` has no attribute `fat_chart`.

- [ ] **Step 3: Write the minimal implementation**

Add this function to `charts.py`. Do not change `line_chart` or `sparkline`.

```python
def _runs(points, breaks):
    ordered = sorted(points, key=lambda item: item[0])
    if len(ordered) < 2:
        return []
    runs = [[ordered[0]]]
    for nxt in ordered[1:]:
        previous = runs[-1][-1][0]
        if any(previous < moment < nxt[0] for moment in breaks):
            runs.append([nxt])
        else:
            runs[-1].append(nxt)
    return [run for run in runs if len(run) >= 2]


def fat_chart(series, *, start, end, breaks) -> str:
    width, height, pad = 600.0, 200.0, 18.0
    points = [point for _legend, _tone, rows in series for point in rows]
    bits = ["Body fat."]
    items = []
    for legend, tone, rows in series:
        name = legend.split(".", 1)[0]
        count = len(rows)
        unit = "point" if count == 1 else "points"
        if tone == "mean":
            bits.append(f"{name}, {count} {unit}, unvalidated mean.")
        else:
            bits.append(f"{name}, {count} {unit}.")
        suffix = " none in this range" if not rows else ""
        items.append(f'<li class="tone-{tone}">{legend}{suffix}</li>')
    legend_html = f'<ul class="chart-legend">{"".join(items)}</ul>'
    if not points:
        return legend_html
    x, y, lo, hi = _scale(points, start, end, width, height, pad, 1.0)
    groups = []
    for _legend, tone, rows in series:
        split = breaks if tone == "mean" else []
        lines = []
        for run in _runs(rows, split):
            coords = " ".join(f"{x(at)},{y(value)}" for at, value in run)
            dash = ' stroke-dasharray="6 5"' if tone == "mean" else ""
            lines.append(
                f'<polyline points="{coords}" class="chart-line"{dash} '
                'vector-effect="non-scaling-stroke"/>'
            )
        dots = "".join(
            f'<circle cx="{x(at)}" cy="{y(value)}" r="7" class="chart-dot"/>'
            for at, value in rows
        )
        groups.append(f'<g class="tone-{tone}">{"".join(lines)}{dots}</g>')
    label = " ".join(bits)
    grid = "".join(
        f'<line x1="0" x2="{width:g}" y1="{level}" y2="{level}" '
        'class="chart-grid" vector-effect="non-scaling-stroke"/>'
        for level in (pad, height / 2, height - pad)
    )
    return (
        f'<div class="chart"><svg viewBox="0 0 {width:g} {height:g}" '
        f'role="img" aria-label="{label}">{grid}{"".join(groups)}</svg>'
        f'<div class="chart-y" aria-hidden="true"><span>{hi:.1f}</span>'
        f"<span>{lo:.1f}</span></div></div>{legend_html}"
    )
```

Add to `app.css` after the lean tone rule:

```css
.tone-mean .chart-line { stroke: var(--text); }
.tone-mean .chart-dot { fill: var(--text); }
.chart-legend { display: flex; flex-direction: column; gap: 2px; margin: var(--space-2) 0 0; padding: 0; list-style: none; font-size: 12px; color: var(--text-2); }
```

Do not add a new colour token. Do not change `.tone-weight`.

- [ ] **Step 4: Run the test to verify it passes**

Run the Step 2 command plus `test_line_chart_needs_two_points` and `test_weight_line_chart_stays_one_series`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/bathroom-hub/src/bathroom_hub/charts.py apps/bathroom-hub/src/bathroom_hub/static/app.css apps/bathroom-hub/tests/test_bathroom_hub.py
git commit -m "Draw scale, tape, and unvalidated-mean body-fat lines."
```

---

### Task 6: Create, edit, and delete a tape occasion

**Files:**
- Modify: `apps/bathroom-hub/src/bathroom_hub/web.py`
- Test: `apps/bathroom-hub/tests/test_bathroom_hub.py`

**Interfaces:**
- Consumes: `insert_tape`, `update_tape`, `delete_tape`, `tape`, `profile_tapes`, `parse_wall_clock`, `wall_input`, `local_minute_value`, `current_profile`, `redirect`, `page`.
- Produces: `GET`/`POST /measures/new`, `GET`/`POST /measures/{id}`, `POST /measures/{id}/delete`.

- [ ] **Step 1: Write the failing test**

Add `TapePageTests` as an `IsolatedAsyncioTestCase` with the same `asyncSetUp` / `asyncTearDown` as `WebTests` (`TemporaryDirectory`, `Database`, `Hub`, `TestClient` of `create_app(self.hub, NoBluetooth())`).

```python
async def test_new_tape_page_defaults_to_this_minute_and_hints_women(self):
    me = self.db.create_profile(ME)
    guest = self.db.create_profile(GUEST)
    male = await self.client.get("/measures/new", cookies={"profile": str(me)})
    male_html = await male.text()
    self.assertEqual(male.status, 200)
    self.assertIn('type="datetime-local"', male_html)
    self.assertIn('name="hip_cm"', male_html)
    self.assertNotIn("Needed by the Navy equation for women.", male_html)
    now = datetime.now().astimezone()
    values = {
        hub_module.local_minute_value(now),
        hub_module.local_minute_value(now - timedelta(minutes=1)),
    }
    self.assertTrue(any(f'value="{value}"' in male_html for value in values))
    female = await self.client.get(
        "/measures/new", cookies={"profile": str(guest)}
    )
    self.assertIn(
        "Needed by the Navy equation for women.", await female.text()
    )

async def test_save_edit_and_delete_keep_scale_fat(self):
    me = self.db.create_profile(ME)
    measurement_id = self.db.insert_measurement(
        {
            "measured_at": "2020-01-02T19:00:00+00:00",
            "utc_offset_s": 0,
            "weight_kg": 97.5,
            "raw_weight": "aa",
            "received_at": "2020-01-02T19:00:01+00:00",
            "profile_id": me,
            "fat_pct": 27.4,
            "lean_kg": 70.8,
            "bmi": 29.1,
            "google_status": "done",
        }
    )
    cookies = {"profile": str(me)}
    saved = await self.client.post(
        "/measures/new",
        data={
            "measured_at": "2020-01-02T20:00",
            "waist_cm": "92",
            "neck_cm": "38,0",
            "hip_cm": "",
        },
        cookies=cookies,
        allow_redirects=False,
    )
    self.assertEqual(saved.status, 302)
    self.assertIn("Saved%20on%20this%20hub.", saved.headers["Location"])
    row = self.db.profile_tapes(me)[0]
    zone = datetime.now().astimezone().tzinfo
    aware = datetime(2020, 1, 2, 20, 0, tzinfo=zone)
    self.assertEqual(
        row["measured_at"],
        aware.astimezone(UTC).replace(microsecond=0).isoformat(timespec="seconds"),
    )
    self.assertEqual(row["neck_cm"], 38.0)
    self.assertIsNone(row["hip_cm"])
    stored = self.db.measurement(measurement_id)
    self.assertEqual(stored["fat_pct"], 27.4)
    self.assertEqual(stored["lean_kg"], 70.8)
    self.assertEqual(stored["google_status"], "done")
    edit = await self.client.get(f"/measures/{row['id']}", cookies=cookies)
    self.assertIn('value="2020-01-02T20:00"', await edit.text())
    missing = await self.client.post(
        "/measures/new",
        data={"measured_at": "2020-01-02", "waist_cm": "", "neck_cm": "", "hip_cm": ""},
        cookies=cookies,
    )
    self.assertIn("Date and time are required.", await missing.text())
    self.assertIn("Enter at least one of waist, neck, or hip.", await missing.text())
    future = await self.client.post(
        "/measures/new",
        data={"measured_at": "2099-01-01T00:00", "waist_cm": "90", "neck_cm": "", "hip_cm": ""},
        cookies=cookies,
    )
    self.assertIn("That time is in the future.", await future.text())
    self.assertIn('value="2099-01-01T00:00"', await future.text())
    duplicate = await self.client.post(
        "/measures/new",
        data={"measured_at": "2020-01-02T20:00", "waist_cm": "93", "neck_cm": "", "hip_cm": ""},
        cookies=cookies,
    )
    self.assertIn(
        "You already have a measurement at that time. Edit that one.",
        await duplicate.text(),
    )
    deleted = await self.client.post(
        f"/measures/{row['id']}/delete",
        cookies=cookies,
        allow_redirects=False,
    )
    self.assertEqual(deleted.status, 302)
    self.assertIsNone(self.db.tape(row["id"]))
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.TapePageTests -v
```

Expected: FAIL. `GET /measures/new` is 404.

- [ ] **Step 3: Write the minimal implementation**

Add routes in `create_app`:

```python
app.router.add_route("*", "/measures/new", tape_new)
app.router.add_route("*", "/measures/{id:\\d+}", tape_edit)
app.router.add_post("/measures/{id:\\d+}/delete", tape_delete)
```

`tape_new` and `tape_edit` use `current_profile`. No profiles redirects to `/welcome`. A tape whose `profile_id` is not the current profile is 404.

Parse circumference fields the way `parse_profile` parses height: strip, treat blank as `None`, replace `,` with `.`, `float` or the error `Waist must be a number.` Range errors, only when the field is present:

- waist: `Waist must be between 50 and 200 cm.`
- neck: `Neck must be between 20 and 60 cm.`
- hip: `Hip must be between 60 and 200 cm.`

If all three are empty: `Enter at least one of waist, neck, or hip.` Call `parse_wall_clock(text, now=datetime.now(UTC))`. On a string error, redisplay the submitted `measured_at`. On success, store `measured_at` as `instant.isoformat(timespec="seconds")`, `utc_offset_s`, the three circumferences, and `received_at` from `now_iso()` on insert only. `insert_tape` / `update_tape` returning `None` / `False` because of the unique constraint shows `You already have a measurement at that time. Edit that one.` Success redirects to `/` with flash `Saved on this hub.`

The form is a `.form` with one `.field` per input, `.with-unit` and `cm` on the three circumferences, and `<ul class="errors">` when there are errors. Hip's hint is rendered only when `profile["sex"] == "female"`. The edit value uses `wall_input(row["measured_at"], row["utc_offset_s"])`. The new value uses `local_minute_value(datetime.now().astimezone())`. Delete is a POST form with `data-confirm="Delete this tape measurement? It is only stored on this hub."` and redirects to `/`.

- [ ] **Step 4: Run the test to verify it passes**

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.TapePageTests -v
```

Expected: PASS, 2 tests.

- [ ] **Step 5: Commit**

```bash
git add apps/bathroom-hub/src/bathroom_hub/web.py apps/bathroom-hub/tests/test_bathroom_hub.py
git commit -m "Add manual tape create, edit, and delete."
```

---

### Task 7: Per-person body-fat selector

**Files:**
- Modify: `apps/bathroom-hub/src/bathroom_hub/web.py`
- Test: `apps/bathroom-hub/tests/test_bathroom_hub.py`

**Interfaces:**
- Consumes: `set_fat_display`, `normalize_fat_display`, `profile_fields`, `edit_profile`.
- Produces: the selector on `/profiles/{id}/edit` only.

- [ ] **Step 1: Write the failing test**

Add to `TapePageTests` or `WebTests`:

```python
async def test_fat_display_is_only_on_the_edit_page(self):
    me = self.db.create_profile(ME)
    self.assertEqual(self.db.normalize_fat_display(self.db.profile(me)["fat_display"]), "scale")
    edit = await self.client.get(f"/profiles/{me}/edit")
    html = await edit.text()
    self.assertIn("Chooses the main body-fat number.", html)
    self.assertIn("Google Health still receives the scale estimate.", html)
    self.assertIn('name="fat_display"', html)
    new = await self.client.get("/profiles/new")
    self.assertNotIn('name="fat_display"', await new.text())
    saved = await self.client.post(
        f"/profiles/{me}/edit",
        data={
            "name": "Me",
            "sex": "male",
            "birth_date": "1990-01-01",
            "height_cm": "183",
            "start_weight_kg": "97",
            "fat_display": "average",
        },
        allow_redirects=False,
    )
    self.assertEqual(saved.status, 302)
    self.assertEqual(self.db.profile(me)["fat_display"], "average")
    rejected = await self.client.post(
        f"/profiles/{me}/edit",
        data={
            "name": "Me",
            "sex": "male",
            "birth_date": "1990-01-01",
            "height_cm": "183",
            "fat_display": "blend",
        },
    )
    self.assertIn("Choose scale, tape, or average.", await rejected.text())
    self.assertEqual(self.db.profile(me)["fat_display"], "average")
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.TapePageTests.test_fat_display_is_only_on_the_edit_page -v
```

Expected: FAIL. The edit page does not contain `name="fat_display"`.

- [ ] **Step 3: Write the minimal implementation**

In `edit_profile` only, under the existing profile form, add a segmented radio group named `fat_display` with values `scale`, `tape`, and `average`, labels `Scale estimate`, `Tape estimate`, and `Average of scale and tape estimates`. Under it, the exact sentence: `Chooses the main body-fat number. The chart still shows all three lines. Google Health still receives the scale estimate.`

On POST, read `fat_display` before saving the profile. If it is not one of the three values, append `Choose scale, tape, or average.` to the profile errors and do not call `update_profile` or `set_fat_display`. When the profile save succeeds, call `set_fat_display`. Do not add the field to `profile_fields`, `parse_profile`, or `PROFILE_FIELDS`.

- [ ] **Step 4: Run the test to verify it passes**

Run the Step 2 command. Expected: PASS. Also run `tests.test_bathroom_hub.WebTests.test_create_profile_and_reject_invalid`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/bathroom-hub/src/bathroom_hub/web.py apps/bathroom-hub/tests/test_bathroom_hub.py
git commit -m "Let each person choose the main body-fat number."
```

---

### Task 8: Dashboard, history, settings, and the design note

**Files:**
- Modify: `apps/bathroom-hub/src/bathroom_hub/web.py`
- Modify: `DESIGN.md`
- Test: `apps/bathroom-hub/tests/test_bathroom_hub.py`

**Interfaces:**
- Consumes: `fat_series`, `fat_headline`, `fat_chart`, `line_chart`, `profile_tapes`, `normalize_fat_display`, `tenth`, `nearest_tape`, `navy_fat_pct`.
- Produces: the tape card, the three-line fat card, history source text, the settings sentence, and one `DESIGN.md` sentence.

- [ ] **Step 1: Write the failing test**

Add to `TapePageTests`:

```python
async def test_dashboard_keeps_weight_and_shows_three_fat_lines(self):
    me = self.db.create_profile(ME)
    self.db.insert_measurement(
        {
            "measured_at": datetime.now(UTC).replace(microsecond=0).isoformat(timespec="seconds"),
            "utc_offset_s": 0,
            "weight_kg": 97.5,
            "impedance": 491.1,
            "raw_weight": "aa",
            "received_at": datetime.now(UTC).isoformat(timespec="seconds"),
            "profile_id": me,
            "fat_pct": 27.4,
            "lean_kg": 70.8,
            "bmi": 29.1,
        }
    )
    tape_at = datetime.now(UTC) - timedelta(hours=11)
    self.db.insert_tape(
        {
            "profile_id": me,
            "measured_at": tape_at.replace(microsecond=0).isoformat(timespec="seconds"),
            "utc_offset_s": 0,
            "waist_cm": 92,
            "neck_cm": 38,
            "hip_cm": None,
            "received_at": datetime.now(UTC).isoformat(timespec="seconds"),
        }
    )
    html = await (await self.client.get("/", cookies={"profile": str(me)})).text()
    self.assertIn("Weight over", html)
    self.assertIn('class="tone-weight"', html)
    self.assertIn("97.5", html)
    self.assertIn("Scale estimate", html)
    self.assertIn("Tape estimate", html)
    self.assertIn("Average of scale and tape estimates. Unvalidated mean.", html)
    self.assertIn("27.4", html)
    self.assertIn("±4.8", html)
    self.assertIn("92", html)
    weight_svg = html.split('class="tone-weight"')[1].split("</svg>")[0]
    self.assertNotIn("92", weight_svg)
    self.db.set_fat_display(me, "average")
    average = await (await self.client.get("/", cookies={"profile": str(me)})).text()
    self.assertIn("Average of scale and tape estimates.", average)
    self.assertIn(
        "Unvalidated mean of the two estimates. Not a more accurate measurement.",
        average,
    )
    self.assertEqual(average.count("more accurate"), 1)
    self.assertIn("From the unvalidated mean. Not the stored lean mass.", average)
    self.db.set_fat_display(me, "tape")
    tape_html = await (await self.client.get("/", cookies={"profile": str(me)})).text()
    self.assertIn("Tape estimate. It is not the scale reading.", tape_html)
    self.assertNotIn("±4.8", tape_html)

async def test_history_shows_raw_sources_and_settings_names_navy(self):
    me = self.db.create_profile(ME)
    weigh_at = datetime(2020, 1, 2, 19, 0, tzinfo=UTC)
    self.db.insert_measurement(
        {
            "measured_at": weigh_at.isoformat(timespec="seconds"),
            "utc_offset_s": 0,
            "weight_kg": 97.5,
            "raw_weight": "aa",
            "received_at": weigh_at.isoformat(timespec="seconds"),
            "profile_id": me,
            "fat_pct": 27.4,
            "lean_kg": 70.8,
            "bmi": 29.1,
        }
    )
    self.db.insert_tape(
        {
            "profile_id": me,
            "measured_at": (weigh_at - timedelta(hours=11)).isoformat(timespec="seconds"),
            "utc_offset_s": 0,
            "waist_cm": 92,
            "neck_cm": 38,
            "hip_cm": None,
            "received_at": weigh_at.isoformat(timespec="seconds"),
        }
    )
    self.db.set_fat_display(me, "average")
    history = await (
        await self.client.get("/history", cookies={"profile": str(me)})
    ).text()
    self.assertIn("24.1% average of scale and tape estimates", history)
    self.assertIn("27.4% scale estimate", history)
    self.assertIn("20.8% tape estimate", history)
    self.assertIn("70.8 kg lean", history)
    settings = await (await self.client.get("/settings")).text()
    self.assertIn("Jebb et al. (2000)", settings)
    self.assertIn(
        "A tape estimate uses the Navy equations, hip as well for women, and is not sent to Google Health.",
        settings,
    )
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
../../.venv/bin/python -m unittest tests.test_bathroom_hub.TapePageTests.test_dashboard_keeps_weight_and_shows_three_fat_lines tests.test_bathroom_hub.TapePageTests.test_history_shows_raw_sources_and_settings_names_navy -v
```

Expected: FAIL. The dashboard HTML does not contain `Average of scale and tape estimates. Unvalidated mean.`

- [ ] **Step 3: Write the minimal implementation**

Add this function to `web.py` and use its return value as the history detail string. `mode` is `normalize_fat_display` of that row's profile. `scale_pct` and `lean_kg` are the stored measurement columns. `tape_pct` is `tenth(navy_fat_pct(...))` for the tape chosen by `nearest_tape`, or `None`.

```python
def history_sources(mode, scale_pct, tape_pct, lean_kg) -> str:
    phrases = []
    mean = None
    if scale_pct is not None and tape_pct is not None:
        mean = bodycomp.unvalidated_mean(scale_pct, tape_pct)
    if mode == "average" and mean is not None:
        phrases.append(f"{mean:.1f}% average of scale and tape estimates")
    elif mode == "tape" and tape_pct is not None:
        phrases.append(f"{tape_pct:.1f}% tape estimate")
    elif scale_pct is not None:
        phrases.append(f"{scale_pct:.1f}% scale estimate")
    if scale_pct is not None and not phrases[0].endswith("scale estimate"):
        phrases.append(f"{scale_pct:.1f}% scale estimate")
    if tape_pct is not None and not any(
        item.endswith("tape estimate") for item in phrases
    ):
        phrases.append(f"{tape_pct:.1f}% tape estimate")
    if lean_kg is not None and scale_pct is not None:
        phrases.append(f"{lean_kg:.1f} kg lean")
    return " · ".join(phrases)
```

In `dashboard`, keep the existing weight-card block that calls `charts.line_chart` for `weight_points`. Add a tape card before the fat/lean pair:

- Load `profile_tapes` for the current profile only.
- Parse tape and weigh instants with `datetime.fromisoformat`.
- Call `fat_series` with the window `start`/`end` already computed for weight. Pass every weigh and tape, including those outside the window.
- Waist chart: `charts.line_chart` on waist points inside the window, `tone="lean"`, `min_span=2`, label `Waist over {days} days`. One waist point uses the existing empty-chart sentence pattern: `Add a few more waist measurements to see a trend.`
- Show the latest waist, neck, and hip numbers when present. Link `Add` to `/measures/new`. List each stored occasion, newest first, with `when(measured_at)` and a link to `/measures/{id}`.
- Body-fat card: `fat_headline` for `normalize_fat_display(profile["fat_display"])`. Render `number` as the mid-size figure when it is not `None`. Render `caption` and `detail`. Render the ± line only when `show_jebb_error` is true, using the existing `FAT_ERROR[profile["sex"]]`.
- Lean card: when `lean_kg` is not `None`, show it and `lean_caption`. In `scale` mode, keep the current `BMI {bmi:.1f}` line beside the stored lean mass. Do not replace stored `lean_kg` in the database.
- `fat_chart` gets the three series from `FatSeries`, legends `Scale estimate`, `Tape estimate`, and `Average of scale and tape estimates. Unvalidated mean.`, tones `fat`, `lean`, `mean`, and `breaks`.

History, for a row with a `profile_id`, loads that profile's tapes. A paired row uses `nearest_tape` and `navy_fat_pct`. The primary phrase depends on `fat_display`:

- `scale`: `27.4% scale estimate`
- `tape`: `20.8% tape estimate`
- `average`: `24.1% average of scale and tape estimates`

Then, when they exist and are not already the primary phrase, append `27.4% scale estimate` and `20.8% tape estimate`. Append `70.8 kg lean` from the stored `lean_kg` whenever `fat_pct` is not `None`. A row with no pair does not show a mean.

Settings: immediately after the existing Jebb paragraph, add the sentence `A tape estimate uses the Navy equations, hip as well for women, and is not sent to Google Health.`

`DESIGN.md` charts bullet: after the sentence about one line, add `The body-fat card draws three labeled lines on one percent axis: scale estimate, tape estimate, and an unvalidated mean. The weight chart stays one line. Waist has its own centimetre chart.`

Do not change `Hub.upload`, `composition_for`, or `recompute_profile`.

- [ ] **Step 4: Run the test to verify it passes**

Run the Step 2 command. Expected: PASS. Then:

```bash
../../.venv/bin/python -m unittest discover -s tests -p test_bathroom_hub.py -v
```

from `apps/bathroom-hub`. Expected: PASS, including `test_pages_render`, `test_line_chart_needs_two_points`, and `test_linked_profile_uploads_weight_and_body_fat`.

From the repository root, `make test`. Expected: the `apps/bathroom-hub/tests` section passes. Do not treat a failure in an unrelated suite as a reason to change this feature.

- [ ] **Step 5: Commit**

```bash
git add apps/bathroom-hub/src/bathroom_hub/web.py apps/bathroom-hub/tests/test_bathroom_hub.py DESIGN.md
git commit -m "Show tape, waist, and the selected body-fat figure."
```
