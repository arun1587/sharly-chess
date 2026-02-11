# Plan: Convert Float Timestamps to Datetime (with DB Migration)

## Context

The codebase stores timestamps as FLOAT (unix epoch) in SQLite, then converts them to `datetime` in Python. The goal is to:

1. **Phase 1** — Convert all 6 DB-stored float timestamp fields at the Python level (StoredClass types become `datetime`, wrapper classes return directly, no DB changes)
2. **Phase 2** — Migrate DB columns from FLOAT to TEXT (SQLite-recommended ISO-8601 format), replace SQL triggers with Python code

`board.last_result_update` (Phase 1) is already done on branch `chore/871-add-datetime-helpers`.

---

## Phase 1: Python-Level Conversion (no DB changes)

Apply the same pattern as `board.last_result_update` to each field:
- StoredClass type: `float` → `datetime` (or `datetime | None`)
- Load via `load_optional_timestamp_from_database_field()` in `_row_to_stored_*` methods
- Write path: `time.time()` stays, but return/assign `datetime.fromtimestamp(date)` where the return value feeds a StoredClass field
- Wrapper class properties: return `stored_*.field` directly (no conversion)
- Display: replace `format_timestamp_date_time(float)` with `format_datetime(datetime)`

### Important: Comparison sites need updating

`screen_user_controller.py` compares `tournament.last_update > date` where `date` is a float from `get_if_modified_since()`. Once `last_update` becomes `datetime`, the comparison type must match. Two options:
- Convert the HTTP float to `datetime` (`datetime.fromtimestamp(date)`)
- Or keep `get_if_modified_since` returning float and convert in comparison

**Recommendation:** Convert `get_if_modified_since` to return `datetime | None` — this keeps comparisons clean and pushes the conversion to the boundary.

Similarly, `ffe_background_uploader.py` and `chess_results_background_uploader.py` compare `last_upload` (float from plugin data) against `max(tournament.last_update, ...)`. These need the same treatment — either convert `last_upload` to datetime or convert tournament fields to float for comparison. Since `last_upload` is also plugin-stored data, converting it to datetime is cleaner.

### Commit order

Each field/group can be a separate commit:

#### Commit 1: `family.last_update` (simplest — no triggers, no comparisons)

Files to modify:
- `src/database/sqlite/event/event_store.py` — `StoredFamily.last_update: float = 0.0` → `datetime`
- `src/database/sqlite/event/event_database.py`:
  - `_row_to_stored_family()` line 1337 — use `load_optional_timestamp_from_database_field(row['last_update'])`
  - `_write_stored_family()` line 1417 — keep `time.time()` for DB write, but where return value feeds stored field, convert
- `src/data/family.py`:
  - `last_update` property: return type `datetime | None`, return directly
  - `last_update_str`: replace `format_timestamp_date_time(self.last_update)` with `format_datetime(self.last_update)` (if not None)
- `src/web/controllers/user/screen_user_controller.py` line 108 — comparison `family.last_update or 0` needs updating to work with datetime

#### Commit 2: `screen.last_update`

Files to modify:
- `src/database/sqlite/event/event_store.py` — `StoredScreen.last_update: float = 0.0` → `datetime`
- `src/database/sqlite/event/event_database.py`:
  - `_row_to_stored_screen()` line 1500
  - `_write_stored_screen()` line 1611
  - `_set_stored_screen_last_update()` line 1530
- `src/data/screen.py`:
  - `last_update` property (line 718-724) — returns either `stored_screen.last_update` or `family.last_update`
  - `last_update_str` (line 761-763) — replace `format_timestamp_date_time`
- `src/web/controllers/user/screen_user_controller.py` line 64 — `screen.last_update > date`

#### Commit 3: `screen_set.last_update`

Files to modify:
- `src/database/sqlite/event/event_store.py` — `StoredScreenSet.last_update: float = 0.0` → `datetime`
- `src/database/sqlite/event/event_database.py`:
  - `_row_to_stored_screen_set()` line 1667
  - `_write_stored_screen_set()` line 1735
  - `clone_stored_screen_set()` line 1772
  - `delete_stored_screen_set()` line 1812
- `src/data/screen_set.py`:
  - `last_update` property (line 462-468)
  - `last_update_str` (line 471-472) — replace `format_timestamp_date_time`

#### Commit 4: `tournament.last_update`, `last_player_update`, `last_pairing_update`

Files to modify:
- `src/database/sqlite/event/event_store.py` — all three fields in `StoredTournament`
- `src/database/sqlite/event/event_database.py`:
  - `_row_to_stored_tournament()` lines 590-592
  - `_write_stored_tournament()` line 674
  - `set_tournament_pairing_settings()` line 739
  - `set_tournament_current_round()` line 753
- `src/data/tournament.py`:
  - `last_update` property (line 222) — return type `datetime`
  - `last_player_update` property (line 226) — return type `datetime`
  - `last_pairing_update` property (line 230) — return type `datetime`
- **Comparison sites** (these are the tricky part):
  - `src/web/controllers/user/screen_user_controller.py` — all comparisons with `date: float`
  - `src/web/controllers/base_controller.py` — `get_if_modified_since()` return type
  - `src/plugins/ffe/ffe_background_uploader.py` lines 134, 340-341 — `last_upload` comparisons
  - `src/plugins/chess_results/chess_results_background_uploader.py` lines 130-132

#### Commit 5: Clean up `format_timestamp_date_time`

Once all usages are converted, check if `format_timestamp_date_time` and `format_timestamp` in `src/utils/date_time.py` can be removed. They may still be used by:
- `src/data/event.py:299` — `Event.last_update_str` (file mtime, stays float)
- `src/data/loader.py:209` — `Archive.date_str` (file mtime, stays float)
- `src/data/input_output/data_source.py:353` — runtime float
- Plugin templates passing `format_timestamp_date_time` to Jinja context

These non-DB usages may keep `format_timestamp_date_time` alive, or they can also be converted to datetime at their source. Evaluate and clean up.

---

## Phase 2: DB Migration to TEXT + Replace Triggers with Python

**Separate PR — after Phase 1 is merged.**

### Step 1: New migration (e.g., m075)

Convert all FLOAT timestamp columns to TEXT:

```sql
-- For each table/column:
ALTER TABLE `tournament` ADD COLUMN `last_update_new` TEXT;
UPDATE `tournament` SET `last_update_new` =
    CASE WHEN `last_update` > 0
         THEN strftime('%Y-%m-%d %H:%M:%f', `last_update`, 'unixepoch')
         ELSE NULL END;
ALTER TABLE `tournament` DROP COLUMN `last_update`;
ALTER TABLE `tournament` RENAME COLUMN `last_update_new` TO `last_update`;
-- Repeat for last_player_update, last_pairing_update
-- Repeat for screen.last_update, screen_set.last_update, family.last_update
```

### Step 2: Drop timestamp-related triggers (9 triggers)

The database has two categories of triggers. Only the timestamp triggers are replaced:

**Timestamp triggers (TO REPLACE with Python — 9 triggers):**

| Table | Trigger | Action |
|-------|---------|--------|
| `tournament` | `mark_tournament_dirty_on_relevant_update` | Sets `dirty=1` when `last_*_update` changes |
| `pairing` | `set_tournament_last_pairing_update_on_pairing_insert` | Updates `last_pairing_update` |
| `pairing` | `set_tournament_last_pairing_update_on_pairing_update` | Updates `last_pairing_update` |
| `pairing` | `set_tournament_last_pairing_update_on_pairing_delete` | Updates `last_pairing_update` |
| `board` | `set_tournament_last_pairing_update_on_board_update` | Updates `last_pairing_update` |
| `player` | `set_tournament_last_player_update_on_player_update` | Updates `last_player_update` |
| `tournament_player` | `set_tournament_last_player_update_on_tournament_player_insert` | Updates `last_player_update` |
| `tournament_player` | `set_tournament_last_player_update_on_tournament_player_update` | Updates `last_player_update` |
| `tournament_player` | `set_tournament_last_player_update_on_tournament_player_delete` | Updates `last_player_update` |

**Cascade deletion triggers (KEEP as-is — 3 triggers):**

| Table | Trigger | Action |
|-------|---------|--------|
| `screen_set` | `delete_screen_trigger` | Deletes orphaned screens (m030) |
| `tournament_player` | `delete_player_on_tournament_player_delete` | Deletes orphaned players (m038) |
| `tournament_player` | `delete_pairing_on_tournament_player_delete` | Deletes related pairings (m038) |
| `pairing` | `delete_board_on_pairing_delete` | Deletes related boards (m038) |

The cascade deletion triggers are unrelated to timestamp handling and should remain in the database.

In the migration, drop only the 9 timestamp triggers:
```sql
DROP TRIGGER IF EXISTS `mark_tournament_dirty_on_relevant_update`;
DROP TRIGGER IF EXISTS `set_tournament_last_pairing_update_on_pairing_insert`;
DROP TRIGGER IF EXISTS `set_tournament_last_pairing_update_on_pairing_update`;
DROP TRIGGER IF EXISTS `set_tournament_last_pairing_update_on_pairing_delete`;
DROP TRIGGER IF EXISTS `set_tournament_last_pairing_update_on_board_update`;
DROP TRIGGER IF EXISTS `set_tournament_last_player_update_on_player_update`;
DROP TRIGGER IF EXISTS `set_tournament_last_player_update_on_tournament_player_insert`;
DROP TRIGGER IF EXISTS `set_tournament_last_player_update_on_tournament_player_update`;
DROP TRIGGER IF EXISTS `set_tournament_last_player_update_on_tournament_player_delete`;
```

### Step 3: Replace triggers with Python code in EventDatabase

Add private helper methods:

```python
def _touch_tournament_last_pairing_update(self, tournament_id: int):
    now = dump_datetime_to_database_field(datetime.now())
    self.execute(
        'UPDATE `tournament` SET `last_pairing_update` = ? WHERE `id` = ?',
        (now, tournament_id),
    )

def _touch_tournament_last_player_update(self, tournament_id: int):
    # similar

def _mark_tournament_dirty(self, tournament_id: int):
    self.execute(
        'UPDATE `tournament` SET `dirty` = 1 WHERE `id` = ?',
        (tournament_id,),
    )
```

Call these from existing database methods:
- `add_stored_pairing()` / `update_stored_pairing()` / `delete_stored_pairing()` → `_touch_tournament_last_pairing_update`
- `update_stored_board()` → `_touch_tournament_last_pairing_update`
- `update_stored_player()` → `_touch_tournament_last_player_update` (for all tournaments the player belongs to)
- `add_stored_tournament_player()` / `update_stored_tournament_player()` / `delete_stored_tournament_player()` → `_touch_tournament_last_player_update`
- All touch methods also call `_mark_tournament_dirty`

### Step 4: Update helpers

- Load path: replace `load_optional_timestamp_from_database_field(float)` with `load_optional_datetime_from_database_field(str)` using `datetime.fromisoformat()` or `strptime('%Y-%m-%d %H:%M:%f')`
- Write path: replace `time.time()` with `dump_datetime_to_database_field(datetime.now())`
- Choose a consistent TEXT format: `'%Y-%m-%d %H:%M:%S.%f'` (matches `strftime('%Y-%m-%d %H:%M:%f')` in SQLite) or simpler `'%Y-%m-%d %H:%M:%S'` if sub-second not needed

### Step 5: Backward migration

Convert TEXT back to FLOAT for rollback safety:
```sql
UPDATE `tournament` SET `last_update_new` =
    CAST(strftime('%s', `last_update`) AS REAL);
```
Recreate triggers (copy from m044).

---

## Critical files reference

| File | Role |
|------|------|
| `src/database/sqlite/event/event_store.py` | StoredClass definitions |
| `src/database/sqlite/event/event_database.py` | DB read/write methods |
| `src/database/sqlite/sqlite_database.py` | Conversion helpers |
| `src/data/tournament.py` | Tournament wrapper |
| `src/data/screen.py` | Screen wrapper |
| `src/data/screen_set.py` | ScreenSet wrapper |
| `src/data/family.py` | Family wrapper |
| `src/data/board.py` | Board wrapper (already done) |
| `src/utils/date_time.py` | `format_timestamp_date_time`, `format_datetime` |
| `src/web/controllers/user/screen_user_controller.py` | Timestamp comparisons for refresh |
| `src/web/controllers/base_controller.py` | `get_if_modified_since()` |
| `src/plugins/ffe/ffe_background_uploader.py` | Upload timestamp comparisons |
| `src/plugins/chess_results/chess_results_background_uploader.py` | Upload timestamp comparisons |
| `src/database/sqlite/event/migrations/m043_track_tournament_changes.py` | Dirty flag trigger |
| `src/database/sqlite/event/migrations/m044_use_sub_second_last_update_triggers.py` | Timestamp triggers |

## Verification

### Phase 1
- Run existing test suite
- Start the app, create a tournament, add players, create pairings, enter results
- Verify the results screen shows correct timestamps
- Verify screen refresh (polling) still works — screens auto-update when data changes
- Check FFE/Chess Results upload detection still triggers correctly

### Phase 2
- Run existing test suite
- Inspect DB with SQLite browser — timestamps should be readable TEXT
- Verify all the same scenarios as Phase 1
- Verify dirty flag mechanism still works (plugin hooks fire on tournament changes)
- Test backward migration: downgrade and verify floats are restored
