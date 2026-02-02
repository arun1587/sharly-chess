# Timestamp to DateTime Conversion Guide

## Issue Reference
GitHub Issue: [#871 - Replace timestamps by date / datetime objects](https://github.com/Sharly-Chess/sharly-chess/issues/871)

## Overview
This guide explains how to refactor timestamp properties to use `datetime` objects in the application layer while keeping the database layer unchanged.

## Key Principle
- **Database Layer (`Stored*` classes)**: Continue using `float` timestamps
- **Application Layer (wrapper classes)**: Expose `datetime` or `date` objects
- **No database migration required**

---

## Existing Utilities

### Already Available in `src/utils/date_time.py`

```python
# Timestamp → DateTime
datetime.fromtimestamp(timestamp: float) -> datetime

# DateTime → Timestamp  
datetime_obj.timestamp() -> float

# Date → Timestamp
get_date_timestamp(date_: date) -> float

# Formatting (already exists - use these!)
format_date(date_: date | None) -> str
format_datetime(datetime_: datetime) -> str
format_time(datetime_: datetime) -> str
```

---

## Recommended Helper Functions

Add these to `src/utils/date_time.py` for consistency:

```python
from datetime import datetime, date
from typing import TypeAlias

# Type aliases for clarity
Timestamp: TypeAlias = float
OptionalTimestamp: TypeAlias = float | None

def timestamp_to_datetime(ts: float | None) -> datetime | None:
    """Convert a timestamp to datetime, handling None values.
    
    Args:
        ts: Unix timestamp (seconds since epoch) or None
        
    Returns:
        datetime object in local timezone, or None if input is None
    """
    return datetime.fromtimestamp(ts) if ts is not None else None

def datetime_to_timestamp(dt: datetime | None) -> float | None:
    """Convert a datetime to timestamp, handling None values.
    
    Args:
        dt: datetime object or None
        
    Returns:
        Unix timestamp (seconds since epoch), or None if input is None
    """
    return dt.timestamp() if dt is not None else None

def timestamp_to_date(ts: float | None) -> date | None:
    """Convert a timestamp to date, handling None values.
    
    Args:
        ts: Unix timestamp (seconds since epoch) or None
        
    Returns:
        date object, or None if input is None
    """
    return datetime.fromtimestamp(ts).date() if ts is not None else None

def date_to_timestamp(d: date | None) -> float | None:
    """Convert a date to timestamp (midnight local time), handling None values.
    
    Args:
        d: date object or None
        
    Returns:
        Unix timestamp at midnight, or None if input is None
    """
    return get_date_timestamp(d) if d is not None else None
```

---

---

## Pull Request Strategy

### Overview

This refactoring should be split into **4 focused PRs** for easier review, safer deployment, and better git history.

**Why 4 PRs instead of 5?**
- PR #1 combines utilities with Board to validate the nullable pattern
- Board tests the most common use case (`datetime | None`)
- Still keeps PRs small and reviewable
- Validates helpers are useful from day one

---

### PR #1: Foundation - Helpers + Board.last_result_update ⭐⭐⭐

**Size:** Small-Medium (~270 lines including tests)  
**Risk:** Low  
**Dependencies:** None  
**Merge Priority:** **HIGHEST** - Must merge first

**Changes:**
- Add helper functions to `src/utils/date_time.py`
- Add type aliases (`Timestamp`, `OptionalTimestamp`)
- Add comprehensive unit tests for helpers
- Refactor `Board.last_result_update` to return `datetime | None`
- Update board filtering/sorting logic
- Update tests for Board

**Files Changed:**
- `src/utils/date_time.py` (~50 lines)
- `tests/utils/test_date_time.py` (~150 lines)
- `src/data/board.py` (~20 lines)
- `tests/data/test_board.py` (~80 lines)

**PR Title:**  
`feat: Add datetime helpers and convert Board.last_result_update (#871)`

**Why Board?**
- ✅ Tests **nullable property pattern** (most common case)
- ✅ Validates `timestamp_to_datetime()` handles `None` correctly
- ✅ Medium complexity - not too simple, not too complex
- ✅ Lower risk than Event/Tournament
- ✅ Shows helpers are immediately useful

**PR Description Template:**
```markdown
## Description
Adds helper functions for converting between timestamps and datetime objects,
and applies the pattern to `Board.last_result_update` as a validation.

Part of #871 - Replace timestamps by date/datetime objects.

## Changes

### Helper Functions
- Added `timestamp_to_datetime()` helper with None handling
- Added `datetime_to_timestamp()` helper with None handling
- Added `timestamp_to_date()` helper with None handling
- Added `date_to_timestamp()` helper with None handling
- Added type aliases for clarity
- Added comprehensive unit tests (~150 lines)

### Board.last_result_update
- Changed return type from `float | None` to `datetime | None`
- Updated filtering/sorting logic to use datetime
- Updated tests to validate datetime behavior

## Why Board First?
Board.last_result_update is nullable, which is the most common pattern
we'll encounter. This PR validates that the helpers work correctly for
the nullable case.

## Testing
- [x] Unit tests for all helpers (100% coverage)
- [x] Tests cover None handling
- [x] Tests cover edge cases (epoch, far future, negative timestamps)
- [x] Board tests updated for datetime
- [x] Type checker passes
- [x] Manual testing: results display correctly

## Reviewer Notes
- Helper functions are pure additions (no breaking changes)
- Board changes are isolated and low-risk
- Database layer unchanged (still stores float timestamps)
- All conversions happen at property access time
```

**Search Commands Before Starting:**
```bash
# Find all usages of board.last_result_update
rg "board\.last_result_update" --type py

# Find sorting by last_result_update
rg "sort.*last_result_update" --type py

# Find filtering logic
rg "last_result_update.*>=" --type py
```

**Acceptance Criteria:**
- [ ] All helper functions implemented with docstrings
- [ ] 100% test coverage for helpers
- [ ] Type hints correct for all functions
- [ ] `Board.last_result_update` returns `datetime | None`
- [ ] Sorting/filtering logic updated to use datetime
- [ ] Tests handle None cases correctly
- [ ] Type checker passes
- [ ] No breaking changes
- [ ] CI passes
- [ ] Manual testing: results screen works

---

### PR #2: Event + Tournament Timestamps ⭐⭐⭐

**Size:** Medium (~400 lines)  
**Risk:** Low-Medium  
**Dependencies:** PR #1  
**Merge Priority:** High

**Changes:**
- `Event.last_update` → `datetime`
- `Tournament.last_update` → `datetime`
- `Tournament.last_player_update` → `datetime`
- `Tournament.last_pairing_update` → `datetime`
- Update all formatting methods
- Update comparisons/calculations
- **Update plugins that access these properties** ⚠️ **CRITICAL**
- Update tests

**Files Changed:**
- `src/data/event.py` (~20 lines)
- `src/data/tournament.py` (~30 lines)
- `tests/data/test_event.py` (~80 lines)
- `tests/data/test_tournament.py` (~120 lines)
- **`src/plugins/chess_results/chess_results_background_uploader.py`** (~10 lines) ⚠️
- **`src/plugins/ffe/ffe_background_uploader.py`** (~10 lines) ⚠️
- Files using event/tournament timestamps (~150 lines)

**PR Title:**  
`refactor: Convert Event and Tournament timestamps to datetime (#871)`

**Why Combine Event + Tournament?**
- Both are **non-nullable** properties (different from Board)
- Both are core entities (makes sense to review together)
- Shows pattern works for multiple properties
- Still manageable review size

**⚠️ Breaking Change Alert:**
This PR includes updates to `chess_results` and `ffe` plugins that access tournament timestamps.
Without these updates, the plugins will crash with `TypeError` when comparing floats to datetimes.

**Search Commands Before Starting:**
```bash
# Find all event.last_update usages
rg "event\.last_update" --type py

# Find all tournament timestamp usages
rg "tournament\.(last_update|last_player_update|last_pairing_update)" --type py

# Find timestamp comparisons
rg "time\.time\(\).*last_update" --type py

# Find formatting methods
rg "last_update_str" --type py

# Find plugin usages (CRITICAL)
rg "chess_results_upload_needed|ffe_upload_needed" --type py
```

**Acceptance Criteria:**
- [ ] All 4 properties return `datetime` (non-nullable)
- [ ] All `*_str` formatting methods updated
- [ ] All comparisons use `datetime.now()` and `timedelta`
- [ ] **chess_results plugin updated** ⚠️
  - [ ] `chess_results_upload_needed()` converts float to datetime
  - [ ] Import `datetime` added
  - [ ] Tests updated
- [ ] **ffe plugin updated** ⚠️
  - [ ] `ffe_upload_needed()` converts float to datetime
  - [ ] Import `datetime` added
  - [ ] Tests updated
- [ ] Tests updated and passing
- [ ] Type checker passes
- [ ] Manual testing: UI shows correct dates
- [ ] Manual testing: Plugin uploads still work

---

### PR #3: Screen/Family/ScreenSet.last_update ⭐⭐⭐

**Size:** Medium-Large (~350 lines)  
**Risk:** Medium  
**Dependencies:** PR #1, ideally PR #2  
**Merge Priority:** Medium

**Changes:**
- `Screen.last_update` → `datetime | None`
- `Family.last_update` → `datetime | None`
- `ScreenSet.last_update` → `datetime | None`
- Update result filtering logic in `screen.py` (lines ~642-652)
- Update formatting methods
- Update tests

**Files Changed:**
- `src/data/screen.py` (~40 lines)
- `src/data/family.py` (~20 lines)
- `src/data/screen_set.py` (~20 lines)
- `tests/data/test_screen.py` (~100 lines)
- `tests/data/test_family.py` (~50 lines)
- `tests/data/test_screen_set.py` (~50 lines)

**PR Title:**  
`refactor: Convert Screen/Family/ScreenSet.last_update to datetime (#871)`

**Critical Code to Update:**
```python
# src/data/screen.py around line 642
# OLD:
oldest = time.time() - self.results_max_age * 60
if board.last_result_update and board.last_result_update >= oldest:

# NEW:
oldest = datetime.now() - timedelta(minutes=self.results_max_age)
if board.last_result_update and board.last_result_update >= oldest:
```

**Search Commands Before Starting:**
```bash
# Find screen/family/screenset last_update usages
rg "(screen|family|screen_set)\.last_update" --type py

# Find time.time() comparisons
rg "time\.time\(\).*results_max_age" src/data/screen.py

# Find result filtering logic
rg "results_max_age" --type py
```

**Acceptance Criteria:**
- [ ] All three properties return `datetime | None`
- [ ] Result filtering uses `datetime` and `timedelta`
- [ ] Formatting methods updated
- [ ] Tests handle None cases
- [ ] Type checker passes
- [ ] Results screen still works correctly
- [ ] No performance regression

---

## Summary: 4-PR Strategy

| PR | Focus | Size | Risk | Dependencies |
|----|-------|------|------|--------------|
| **#1** | Utilities + Board | ~270 lines | Low | None |
| **#2** | Event + Tournament | ~400 lines | Low-Medium | PR #1 |
| **#3** | Screen/Family/ScreenSet | ~350 lines | Medium | PR #1, #2 |

**Total:** ~1,020 lines across 4 PRs

**Timeline:**
- **Week 1:** PR #1 (Utilities + Board) - Merge by Day 2-3
- **Week 1-2:** PR #2 (Event + Tournament) - Merge by Day 5-7
- **Week 2:** PR #3 (Screen/Family/ScreenSet) - Merge by Day 10-12

**Benefits of This Strategy:**
- ✅ PR #1 validates nullable pattern (most common)
- ✅ PR #2 validates non-nullable pattern
- ✅ Each PR is focused and reviewable
- ✅ Can merge incrementally
- ✅ Easy to revert if needed
- ✅ Clear git history

---

## Unit Tests

### Test File: `tests/utils/test_date_time.py`

Add comprehensive tests for the new helper functions:

```python
"""Tests for timestamp/datetime conversion helpers."""
import time
from datetime import datetime, date, timezone, timedelta
import pytest

from utils.date_time import (
    timestamp_to_datetime,
    datetime_to_timestamp,
    timestamp_to_date,
    date_to_timestamp,
)


class TestTimestampToDatetime:
    """Tests for timestamp_to_datetime helper."""
    
    def test_converts_valid_timestamp(self):
        """Should convert a valid timestamp to datetime."""
        # Known timestamp: 2024-02-07 14:00:00 UTC
        ts = 1707314400.0
        result = timestamp_to_datetime(ts)
        
        assert isinstance(result, datetime)
        assert result.year == 2024
        assert result.month == 2
        assert result.day == 7
    
    def test_handles_none(self):
        """Should return None when input is None."""
        result = timestamp_to_datetime(None)
        assert result is None
    
    def test_handles_zero(self):
        """Should handle epoch (timestamp 0)."""
        result = timestamp_to_datetime(0.0)
        assert isinstance(result, datetime)
        assert result.year == 1970
    
    def test_handles_current_time(self):
        """Should handle current timestamp."""
        now_ts = time.time()
        result = timestamp_to_datetime(now_ts)
        
        assert isinstance(result, datetime)
        # Should be very close to now (within 1 second)
        assert abs((datetime.now() - result).total_seconds()) < 1
    
    def test_handles_far_future(self):
        """Should handle far future timestamps."""
        # Year 2100
        future_ts = 4102444800.0
        result = timestamp_to_datetime(future_ts)
        
        assert isinstance(result, datetime)
        assert result.year == 2100


class TestDatetimeToTimestamp:
    """Tests for datetime_to_timestamp helper."""
    
    def test_converts_valid_datetime(self):
        """Should convert a valid datetime to timestamp."""
        dt = datetime(2024, 2, 7, 14, 0, 0)
        result = datetime_to_timestamp(dt)
        
        assert isinstance(result, float)
        assert result > 0
    
    def test_handles_none(self):
        """Should return None when input is None."""
        result = datetime_to_timestamp(None)
        assert result is None
    
    def test_roundtrip_conversion(self):
        """Should maintain value through roundtrip conversion."""
        original_ts = time.time()
        dt = timestamp_to_datetime(original_ts)
        result_ts = datetime_to_timestamp(dt)
        
        # Should be very close (within floating point precision)
        assert abs(original_ts - result_ts) < 0.001
    
    def test_handles_epoch(self):
        """Should handle epoch datetime."""
        epoch = datetime(1970, 1, 1, 0, 0, 0)
        result = datetime_to_timestamp(epoch)
        
        # Should be close to 0 (accounting for timezone)
        assert abs(result) < 86400  # Within 1 day


class TestTimestampToDate:
    """Tests for timestamp_to_date helper."""
    
    def test_converts_valid_timestamp(self):
        """Should convert a valid timestamp to date."""
        # 2024-02-07 14:00:00
        ts = 1707314400.0
        result = timestamp_to_date(ts)
        
        assert isinstance(result, date)
        assert result.year == 2024
        assert result.month == 2
        assert result.day == 7
    
    def test_handles_none(self):
        """Should return None when input is None."""
        result = timestamp_to_date(None)
        assert result is None
    
    def test_strips_time_component(self):
        """Should only return date, not time."""
        # Timestamp with specific time
        ts = 1707314400.0  # 14:00:00
        result = timestamp_to_date(ts)
        
        assert isinstance(result, date)
        assert not isinstance(result, datetime)


class TestDateToTimestamp:
    """Tests for date_to_timestamp helper."""
    
    def test_converts_valid_date(self):
        """Should convert a valid date to timestamp."""
        d = date(2024, 2, 7)
        result = date_to_timestamp(d)
        
        assert isinstance(result, float)
        assert result > 0
    
    def test_handles_none(self):
        """Should return None when input is None."""
        result = date_to_timestamp(None)
        assert result is None
    
    def test_uses_midnight(self):
        """Should use midnight (00:00:00) for the timestamp."""
        d = date(2024, 2, 7)
        result = date_to_timestamp(d)
        dt = timestamp_to_datetime(result)
        
        assert dt.hour == 0
        assert dt.minute == 0
        assert dt.second == 0
    
    def test_roundtrip_conversion(self):
        """Should maintain date through roundtrip conversion."""
        original_date = date(2024, 2, 7)
        ts = date_to_timestamp(original_date)
        result_date = timestamp_to_date(ts)
        
        assert original_date == result_date


class TestEdgeCases:
    """Tests for edge cases and error conditions."""
    
    def test_negative_timestamp(self):
        """Should handle timestamps before epoch."""
        # 1969-12-31
        ts = -86400.0
        result = timestamp_to_datetime(ts)
        
        assert isinstance(result, datetime)
        assert result.year == 1969
    
    def test_very_large_timestamp(self):
        """Should handle very large timestamps."""
        # Year 3000
        ts = 32503680000.0
        result = timestamp_to_datetime(ts)
        
        assert isinstance(result, datetime)
        assert result.year == 3000
    
    def test_fractional_seconds(self):
        """Should preserve fractional seconds in roundtrip."""
        original_ts = 1707314400.123456
        dt = timestamp_to_datetime(original_ts)
        result_ts = datetime_to_timestamp(dt)
        
        # Should preserve microseconds
        assert abs(original_ts - result_ts) < 0.000001


class TestIntegrationWithExistingUtils:
    """Tests for integration with existing date_time utilities."""
    
    def test_compatible_with_format_datetime(self):
        """Should work with existing format_datetime function."""
        from utils.date_time import format_datetime
        
        ts = time.time()
        dt = timestamp_to_datetime(ts)
        result = format_datetime(dt)
        
        assert isinstance(result, str)
        assert len(result) > 0
    
    def test_compatible_with_format_date(self):
        """Should work with existing format_date function."""
        from utils.date_time import format_date
        
        ts = time.time()
        d = timestamp_to_date(ts)
        result = format_date(d)
        
        assert isinstance(result, str)
        assert len(result) > 0
```

### Test File: `tests/data/test_event.py`

Update existing tests and add new ones:

```python
"""Tests for Event timestamp to datetime conversion."""
from datetime import datetime, timedelta
import time
import pytest

from data.event import Event
from database.sqlite.event.event_store import StoredEvent


class TestEventLastUpdate:
    """Tests for Event.last_update property."""
    
    def test_last_update_returns_datetime(self, sample_event):
        """Should return datetime object, not float."""
        result = sample_event.last_update
        
        assert isinstance(result, datetime)
        assert not isinstance(result, float)
    
    def test_last_update_reflects_database_timestamp(self, sample_event):
        """Should convert database timestamp correctly."""
        # Get the raw timestamp from database
        from database.sqlite.event.event_database import EventDatabase
        raw_timestamp = EventDatabase.database_modified_timestamp(
            sample_event.uniq_id
        )
        
        # Get the datetime from property
        result = sample_event.last_update
        
        # Should match (within 1 second)
        expected = datetime.fromtimestamp(raw_timestamp)
        assert abs((result - expected).total_seconds()) < 1
    
    def test_last_update_is_recent(self, sample_event):
        """Should be close to current time for new events."""
        result = sample_event.last_update
        now = datetime.now()
        
        # Should be within last minute
        assert result <= now
        assert (now - result) < timedelta(minutes=1)
    
    def test_last_update_str_formats_correctly(self, sample_event):
        """Should format datetime as string correctly."""
        result = sample_event.last_update_str
        
        assert isinstance(result, str)
        assert len(result) > 0
        # Should contain date components
        assert str(sample_event.last_update.year) in result


class TestEventComparisons:
    """Tests for comparing event timestamps."""
    
    def test_can_compare_with_datetime(self, sample_event):
        """Should be comparable with datetime objects."""
        now = datetime.now()
        
        # Should not raise TypeError
        assert sample_event.last_update <= now
    
    def test_can_calculate_age(self, sample_event):
        """Should be able to calculate event age."""
        age = datetime.now() - sample_event.last_update
        
        assert isinstance(age, timedelta)
        assert age.total_seconds() >= 0
    
    def test_cannot_compare_with_float(self, sample_event):
        """Should not be comparable with float timestamps."""
        timestamp = time.time()
        
        # This should raise TypeError (comparing datetime with float)
        with pytest.raises(TypeError):
            _ = sample_event.last_update > timestamp
```

### Test File: `tests/data/test_tournament.py`

```python
"""Tests for Tournament timestamp to datetime conversion."""
from datetime import datetime, timedelta
import time

from data.tournament import Tournament


class TestTournamentTimestamps:
    """Tests for Tournament timestamp properties."""
    
    def test_last_update_returns_datetime(self, sample_tournament):
        """Should return datetime for last_update."""
        result = sample_tournament.last_update
        assert isinstance(result, datetime)
    
    def test_last_player_update_returns_datetime(self, sample_tournament):
        """Should return datetime for last_player_update."""
        result = sample_tournament.last_player_update
        assert isinstance(result, datetime)
    
    def test_last_pairing_update_returns_datetime(self, sample_tournament):
        """Should return datetime for last_pairing_update."""
        result = sample_tournament.last_pairing_update
        assert isinstance(result, datetime)
    
    def test_all_timestamps_are_comparable(self, sample_tournament):
        """Should be able to compare all timestamp properties."""
        last_update = sample_tournament.last_update
        last_player = sample_tournament.last_player_update
        last_pairing = sample_tournament.last_pairing_update
        
        # Should not raise TypeError
        assert isinstance(last_update - last_player, timedelta)
        assert isinstance(last_update - last_pairing, timedelta)
    
    def test_timestamps_are_recent(self, sample_tournament):
        """Should all be recent for new tournaments."""
        now = datetime.now()
        
        assert sample_tournament.last_update <= now
        assert sample_tournament.last_player_update <= now
        assert sample_tournament.last_pairing_update <= now
```

### Test File: `tests/data/test_screen.py`

```python
"""Tests for Screen timestamp to datetime conversion."""
from datetime import datetime, timedelta
import time

from data.screen import Screen


class TestScreenLastUpdate:
    """Tests for Screen.last_update property."""
    
    def test_last_update_returns_datetime_or_none(self, sample_screen):
        """Should return datetime or None."""
        result = sample_screen.last_update
        
        assert result is None or isinstance(result, datetime)
    
    def test_last_update_none_when_no_timestamp(self):
        """Should return None when stored timestamp is None."""
        # Create screen with no last_update
        screen = create_screen_with_no_update()
        
        assert screen.last_update is None
    
    def test_last_update_datetime_when_timestamp_exists(self, sample_screen):
        """Should return datetime when timestamp exists."""
        # Ensure screen has a timestamp
        sample_screen.stored_screen.last_update = time.time()
        
        result = sample_screen.last_update
        assert isinstance(result, datetime)


class TestScreenResultsFiltering:
    """Tests for results filtering with datetime."""
    
    def test_results_filters_by_datetime(self, sample_screen):
        """Should filter results using datetime comparison."""
        # This tests the critical code in screen.py ~line 642
        results = list(sample_screen.results_lists)
        
        # Should not raise TypeError
        assert isinstance(results, list)
    
    def test_results_max_age_uses_timedelta(self, sample_screen):
        """Should use timedelta for max age calculation."""
        sample_screen.stored_screen.results_max_age = 60  # 60 minutes
        
        # Should calculate oldest as datetime
        # This is tested implicitly by results_lists not raising
        results = list(sample_screen.results_lists)
        assert isinstance(results, list)
```

---

## PR Workflow

### Before Starting Any PR

1. **Create feature branch:**
   ```bash
   git checkout -b feat/871-<pr-description>
   # Example: git checkout -b feat/871-add-datetime-helpers
   ```

2. **Review the guide:**
   - Read relevant sections
   - Understand the pattern
   - Check dependencies

3. **Run search commands:**
   - Find all affected code
   - Identify edge cases
   - Plan the changes

### During PR Development

1. **Make changes incrementally:**
   - Update property first
   - Update formatting next
   - Update comparisons
   - Update tests last

2. **Run tests frequently:**
   ```bash
   pytest tests/utils/test_date_time.py -v
   pytest tests/data/test_event.py -v
   # etc.
   ```

3. **Check types:**
   ```bash
   mypy src/data/event.py
   mypy src/data/tournament.py
   # etc.
   ```

4. **Manual testing:**
   - Start the application
   - Check UI displays
   - Verify date formatting

### PR Review Checklist

Use this for each PR:

- [ ] All acceptance criteria met
- [ ] Tests added/updated and passing
- [ ] Type checker passes
- [ ] No breaking changes
- [ ] Documentation updated if needed
- [ ] Manual testing completed
- [ ] Performance verified (no regression)
- [ ] Code follows existing patterns

### After PR Merge

1. **Update dependent branches:**
   ```bash
   git checkout feat/871-next-pr
   git rebase main
   ```

2. **Monitor production:**
   - Check error logs
   - Verify UI displays correctly
   - Watch for performance issues

3. **Update progress:**
   - Check off completed items in issue #871
   - Update this guide if needed

---

## Refactoring Patterns

### Example 1: Simple Property (Non-nullable)

**Before:**
```python
class Tournament:
    @property
    def last_update(self) -> float:
        return self.stored_tournament.last_update
```

**After:**
```python
class Tournament:
    @property
    def last_update(self) -> datetime:
        return datetime.fromtimestamp(self.stored_tournament.last_update)
```

### Example 2: Nullable Property

**Before:**
```python
class Screen:
    @property
    def last_update(self) -> float:
        return self.stored_screen.last_update or 0.0
```

**After:**
```python
class Screen:
    @property
    def last_update(self) -> datetime | None:
        if self.stored_screen.last_update:
            return datetime.fromtimestamp(self.stored_screen.last_update)
        return None
```

Or using the helper (recommended):
```python
class Screen:
    @property
    def last_update(self) -> datetime | None:
        return timestamp_to_datetime(self.stored_screen.last_update)
```

### Example 3: Date-only Property

**Before:**
```python
class Player:
    @property
    def birth_date(self) -> float:
        return self.stored_player.birth_date
```

**After:**
```python
class Player:
    @property
    def birth_date(self) -> date | None:
        return timestamp_to_date(self.stored_player.birth_date)
```

### Example 4: Writing Back to Database

When updating values, convert back to timestamp:

**Before:**
```python
def update_last_modified(self):
    self.stored_tournament.last_update = time.time()
```

**After:**
```python
def update_last_modified(self):
    # Still use time.time() which returns float timestamp
    self.stored_tournament.last_update = time.time()
    
    # Or if you have a datetime object:
    # self.stored_tournament.last_update = datetime.now().timestamp()
```

### Example 5: Setter Property

**Before:**
```python
class Event:
    @property
    def last_update(self) -> float:
        return self.stored_event.last_update
    
    @last_update.setter
    def last_update(self, value: float):
        self.stored_event.last_update = value
```

**After:**
```python
class Event:
    @property
    def last_update(self) -> datetime:
        return datetime.fromtimestamp(self.stored_event.last_update)
    
    @last_update.setter
    def last_update(self, value: datetime):
        self.stored_event.last_update = value.timestamp()
```

---

## Common Pitfalls ⚠️

### 1. Don't Mix Timestamps and Datetimes

```python
# ❌ BAD - mixing types
if event.last_update > time.time():  # datetime vs float!

# ✅ GOOD - consistent types
if event.last_update > datetime.now():
```

### 2. Be Careful with Timezone-Naive Datetimes

```python
# datetime.fromtimestamp() returns local time by default
# This is usually fine for display, but be aware:

# For timezone-aware datetimes (if needed):
from datetime import timezone
dt = datetime.fromtimestamp(ts, tz=timezone.utc)

# Note: Adding timezone support is out of scope for this refactoring
# but keep it in mind for future improvements
```

### 3. Comparison Operators Need Updated Types

```python
# ❌ BAD - comparing with old timestamp
if board.last_result_update and board.last_result_update >= oldest_timestamp:
    # oldest_timestamp is a float

# ✅ GOOD - comparing with datetime
if board.last_result_update and board.last_result_update >= oldest:
    # oldest = datetime.now() - timedelta(minutes=max_age)
```

### 4. Arithmetic Operations Change

```python
# ❌ BAD - adding seconds to timestamp
new_time = old_timestamp + 3600  # Add 1 hour

# ✅ GOOD - using timedelta
from datetime import timedelta
new_time = old_datetime + timedelta(hours=1)
```

```

---

## WebSocket & JSON Serialization 🔌

### Current State

The application uses WebSocket channels for real-time updates via Litestar's `ChannelsPlugin`:

```python
# src/web/controllers/index_controller.py:161-170
@websocket_stream('/ws')
async def ws_handler(self, channels: ChannelsPlugin) -> AsyncGenerator[dict, None]:
    async with channels.start_subscription(['ws']) as subscriber:
        async for raw_event in subscriber.iter_events():
            event = (
                json.loads(raw_event)
                if isinstance(raw_event, (bytes, str))
                else raw_event
            )
            yield event
```

### Potential Issue

**Currently:** Plugins publish simple dicts without datetime objects  
**Future Risk:** If datetime objects are added to WebSocket messages, JSON serialization will fail

**Example from plugins:**
```python
# src/plugins/chess_results/chess_results_background_uploader.py:138
channels_plugin.publish(
    {
        'event': 'upload-event',
        'data': '',  # Currently just strings
    },
    ['ws'],
)
```

### Solution: Custom JSON Encoder (If Needed)

**Only implement this if you start sending datetime objects via WebSocket:**

```python
# src/web/json_encoder.py (create new file)
import json
from datetime import datetime, date

from utils.date_time import format_datetime, format_date


class DateTimeAwareEncoder(json.JSONEncoder):
    """JSON encoder that handles datetime and date objects."""
    
    def default(self, obj):
        if isinstance(obj, datetime):
            return format_datetime(obj)  # Use existing formatter
        if isinstance(obj, date):
            return format_date(obj)
        return super().default(obj)


# Usage in plugins:
channels_plugin.publish(
    json.dumps({
        "event": "tournament_updated",
        "data": {
            "last_update": tournament.last_update  # Now a datetime
        }
    }, cls=DateTimeAwareEncoder),
    ['ws'],
)
```

**Status:** ⚠️ **Not needed immediately**, but good to know for future

---

## Plugin API Compatibility ⚠️ **BREAKING CHANGE**

### Impact Assessment

**This refactoring WILL break existing plugins** that access timestamp properties.

### Affected Plugins

#### **1. chess_results Plugin** 🔴 **REQUIRES UPDATE**

**File:** `src/plugins/chess_results/chess_results_background_uploader.py`  
**Lines:** 129-133

**Current Code (WILL BREAK):**
```python
def chess_results_upload_needed(
    cls, tournament: Tournament | StoredTournament
) -> bool:
    return cls.chess_results_last_upload(tournament) < max(
        tournament.last_update,           # ← Will become datetime!
        tournament.last_player_update,     # ← Will become datetime!
        tournament.last_pairing_update,    # ← Will become datetime!
    )
    # ❌ TypeError: '<' not supported between 'float' and 'datetime'
```

**Fixed Code:**
```python
from datetime import datetime

def chess_results_upload_needed(
    cls, tournament: Tournament | StoredTournament
) -> bool:
    last_upload_dt = datetime.fromtimestamp(
        cls.chess_results_last_upload(tournament) or 0.0
    )
    return last_upload_dt < max(
        tournament.last_update,
        tournament.last_player_update,
        tournament.last_pairing_update,
    )
```

---

#### **2. ffe Plugin** 🔴 **REQUIRES UPDATE**

**File:** `src/plugins/ffe/ffe_background_uploader.py`  
**Lines:** 134-138

**Current Code (WILL BREAK):**
```python
def ffe_upload_needed(cls, tournament: Tournament | StoredTournament) -> bool:
    return cls.ffe_last_upload(tournament) < max(
        tournament.last_update,           # ← Will become datetime!
        tournament.last_player_update,     # ← Will become datetime!
        tournament.last_pairing_update,    # ← Will become datetime!
    )
    # ❌ TypeError: '<' not supported between 'float' and 'datetime'
```

**Fixed Code:**
```python
from datetime import datetime

def ffe_upload_needed(cls, tournament: Tournament | StoredTournament) -> bool:
    last_upload_dt = datetime.fromtimestamp(
        cls.ffe_last_upload(tournament) or 0.0
    )
    return last_upload_dt < max(
        tournament.last_update,
        tournament.last_player_update,
        tournament.last_pairing_update,
    )
```

---

### Plugin Update Checklist

**For PR #2 (Event + Tournament), you MUST also update:**

- [ ] `src/plugins/chess_results/chess_results_background_uploader.py`
  - [ ] Update `chess_results_upload_needed()` method (lines 126-133)
  - [ ] Add `from datetime import datetime` import
  - [ ] Convert `last_upload` float to datetime before comparison
  
- [ ] `src/plugins/ffe/ffe_background_uploader.py`
  - [ ] Update `ffe_upload_needed()` method (lines 133-138)
  - [ ] Add `from datetime import datetime` import
  - [ ] Convert `last_upload` float to datetime before comparison

- [ ] Update tests for both plugins
  - [ ] `tests/plugins/chess_results/test_background_uploader.py`
  - [ ] `tests/plugins/ffe/test_background_uploader.py`

---

### Breaking Change Notice

**For External Plugin Developers (if applicable):**

If you maintain external plugins that access these properties, update your code:

**Old Plugin Code:**
```python
@hookimpl
def on_tournament_update(self, tournament: Tournament):
    if time.time() - tournament.last_update > 86400:  # ❌ BREAKS
        # Refresh data
        pass
```

**New Plugin Code:**
```python
from datetime import datetime, timedelta

@hookimpl
def on_tournament_update(self, tournament: Tournament):
    if datetime.now() - tournament.last_update > timedelta(days=1):  # ✅ WORKS
        # Refresh data
        pass
```

**Migration Path:**
1. If you need the float timestamp: `tournament.last_update.timestamp()`
2. If you need datetime: Use `tournament.last_update` directly
3. Update all comparisons to use `datetime.now()` and `timedelta`

---

## Properties to Refactor

### High Priority (Timestamp Properties)

1. **Event** (`src/data/event.py`):
   - `last_update` (line 293) → `datetime`

2. **Tournament** (`src/data/tournament.py`):
   - `last_update` (line 222) → `datetime`
   - `last_player_update` (line 226) → `datetime`
   - `last_pairing_update` (line 230) → `datetime`

3. **Screen** (`src/data/screen.py`):
   - `last_update` (line 719) → `datetime | None`

4. **Family** (`src/data/family.py`):
   - `last_update` → `datetime | None`

5. **ScreenSet** (`src/data/screen_set.py`):
   - `last_update` → `datetime | None`

6. **Board** (`src/data/board.py`):
   - `last_result_update` → `datetime | None`

### Already Correct ✅

- **TimerHour** (`src/data/timer.py`):
  - `triggered_at` (line 55) → Already returns `datetime` ✅

---

## Impact on Dependent Code

### Format Functions

These will need updates:

**Before:**
```python
def last_update_str(self) -> str:
    return format_timestamp_date_time(self.last_update)
```

**After:**
```python
def last_update_str(self) -> str:
    return format_datetime(self.last_update)
    # Note: format_datetime() already exists in utils/date_time.py
```

### Comparisons

**Before:**
```python
# In screen.py (lines 642-652 approximately)
oldest = time.time() - max_age * 60
if board.last_result_update and board.last_result_update >= oldest:
    # Process board
```

**After:**
```python
from datetime import datetime, timedelta

oldest = datetime.now() - timedelta(minutes=max_age)
if board.last_result_update and board.last_result_update >= oldest:
    # Process board
```

### Time Calculations

**Before:**
```python
# Calculate duration
duration_seconds = end_timestamp - start_timestamp
hours = duration_seconds / 3600
```

**After:**
```python
# Calculate duration
duration = end_datetime - start_datetime  # Returns timedelta
hours = duration.total_seconds() / 3600
# Or: hours = duration.seconds / 3600  (only for durations < 1 day)
```

---

## Performance Considerations

### When to Use Caching

If a datetime property is accessed frequently in a hot loop, consider caching it:

```python
from functools import cached_property

class Tournament:
    @cached_property  # Only converts once
    def last_update(self) -> datetime:
        return datetime.fromtimestamp(self.stored_tournament.last_update)
```

**⚠️ Important**: Only use `@cached_property` if the underlying timestamp won't change during the object's lifetime. For mutable objects or properties that reflect database state, use `@property` instead.

**When to use `@cached_property`:**
- ✅ Object is immutable after creation
- ✅ Property is expensive to compute and called frequently
- ✅ Value won't change during object lifetime

**When to use `@property`:**
- ✅ Value might change (most `last_update` fields)
- ✅ Object reflects live database state
- ✅ Simple conversions (low overhead)

---

## Recommended Refactoring Order

Follow this order to minimize breaking changes:

### Phase 1: Foundation (Day 1)
1. ✅ Add all helper functions to `src/utils/date_time.py`
2. ✅ Add type aliases if desired
3. ✅ Run tests to ensure no regressions

### Phase 2: Simple Properties (Day 2-3)
Update non-nullable properties with no dependencies:
4. ✅ `Event.last_update`
5. ✅ `Tournament.last_update`
6. ✅ `Tournament.last_player_update`
7. ✅ `Tournament.last_pairing_update`

### Phase 3: Nullable Properties (Day 4-5)
Update nullable properties:
8. ✅ `Screen.last_update`
9. ✅ `Family.last_update`
10. ✅ `ScreenSet.last_update`
11. ✅ `Board.last_result_update`

### Phase 4: Dependent Code (Day 6-7)
12. ✅ Update all `*_str` formatting methods to use `format_datetime()`
13. ✅ Update comparison logic (e.g., in `screen.py` lines 642-652)
14. ✅ Update any time arithmetic operations
15. ✅ Search codebase for `time.time()` comparisons with these properties

### Phase 5: Testing & Validation (Day 8)
16. ✅ Update unit tests
17. ✅ Update integration tests
18. ✅ Run type checker (`mypy`)
19. ✅ Manual testing of UI date/time displays
20. ✅ Test edge cases (None values, very old dates, future dates)

---

## Testing Strategy

### 1. Unit Tests

Update tests that mock or assert on timestamp values:

**Before:**
```python
def test_tournament_last_update():
    tournament.last_update = 1707321600.0
    assert tournament.last_update == 1707321600.0
```

**After:**
```python
def test_tournament_last_update():
    expected = datetime(2024, 2, 7, 14, 0, 0)
    tournament.last_update = expected
    assert tournament.last_update == expected
    assert isinstance(tournament.last_update, datetime)
```

### 2. Integration Tests

Verify database read/write still works:

```python
def test_tournament_persists_datetime():
    # Create tournament with datetime
    tournament.last_update = datetime.now()
    
    # Save to database
    save_tournament(tournament)
    
    # Reload from database
    loaded = load_tournament(tournament.id)
    
    # Should get datetime back
    assert isinstance(loaded.last_update, datetime)
    # Allow small precision differences
    assert abs((loaded.last_update - tournament.last_update).total_seconds()) < 1
```

### 3. Type Checking

Run mypy to catch type mismatches:

```bash
mypy src/data/event.py
mypy src/data/tournament.py
mypy src/data/screen.py
# etc.
```

### 4. Manual Testing Checklist

- [ ] UI displays dates correctly
- [ ] Date formatting matches previous behavior
- [ ] Comparison logic works (e.g., "updated in last N minutes")
- [ ] Sorting by date still works
- [ ] Filters based on date ranges work
- [ ] Date input/editing works (if applicable)

---

## Example: Complete Refactoring of Event.last_update

### Step 1: Update the property

```python
# src/data/event.py (line 293)

# Before:
@property
def last_update(self) -> float:
    return EventDatabase.database_modified_timestamp(self.uniq_id)

# After:
@property
def last_update(self) -> datetime:
    timestamp = EventDatabase.database_modified_timestamp(self.uniq_id)
    return datetime.fromtimestamp(timestamp)
```

### Step 2: Update the formatting method

```python
# src/data/event.py (find the last_update_str method)

# Before:
@cached_property
def last_update_str(self) -> str:
    return format_timestamp_date_time(self.last_update)

# After:
@cached_property
def last_update_str(self) -> str:
    return format_datetime(self.last_update)
```

### Step 3: Update any comparisons or calculations

```python
# Search for uses of event.last_update in the codebase

# Before:
if time.time() - event.last_update > 3600:  # 1 hour old
    notify_stale_event(event)

# After:
from datetime import timedelta
if datetime.now() - event.last_update > timedelta(hours=1):
    notify_stale_event(event)
```

### Step 4: Update tests

```python
# tests/test_event.py

# Before:
def test_event_last_update():
    event = create_test_event()
    assert event.last_update > 0
    assert isinstance(event.last_update, float)

# After:
def test_event_last_update():
    event = create_test_event()
    assert event.last_update < datetime.now()
    assert isinstance(event.last_update, datetime)
```

---

## Migration Checklist

### Setup
- [ ] Add helper functions to `utils/date_time.py`
- [ ] Add type aliases (optional)
- [ ] Review this guide with the team

### Property Refactoring
- [ ] Refactor `Event.last_update`
- [ ] Refactor `Tournament.last_update`
- [ ] Refactor `Tournament.last_player_update`
- [ ] Refactor `Tournament.last_pairing_update`
- [ ] Refactor `Screen.last_update`
- [ ] Refactor `Family.last_update`
- [ ] Refactor `ScreenSet.last_update`
- [ ] Refactor `Board.last_result_update`

### Dependent Code Updates
- [ ] Update all `*_str` formatting methods
- [ ] Update comparison logic (search for `time.time()` comparisons)
- [ ] Update arithmetic operations (search for `+ 3600` patterns)
- [ ] Search for `isinstance(x, float)` checks on these properties
- [ ] Check for any serialization code (JSON, etc.)

### Testing & Validation
- [ ] Update unit tests
- [ ] Update integration tests
- [ ] Run type checker (`mypy`)
- [ ] Manual UI testing
- [ ] Test with real database data
- [ ] Verify performance (no significant regression)

### Documentation
- [ ] Update docstrings for changed properties
- [ ] Update API documentation if applicable
- [ ] Add comments explaining conversion at boundaries

---

## Rollback Plan

If issues arise during deployment:

1. **Git revert** is straightforward since database schema is unchanged
2. No data migration means no data loss risk
3. Can revert individual properties independently
4. Type checker will catch most issues before deployment

---

## Notes

- The `Stored*` classes in `event_store.py` should **NOT** be changed
- Database schema remains unchanged (still stores `float` timestamps)
- This is purely an application-layer refactoring
- Existing data in the database works without migration
- Consider adding timezone support as a future enhancement
- `datetime.fromtimestamp()` uses local timezone by default

---

## Additional Resources

- Python datetime documentation: https://docs.python.org/3/library/datetime.html
- Timestamp conversion best practices: https://docs.python.org/3/library/time.html
- Type hints with datetime: https://mypy.readthedocs.io/en/stable/cheat_sheet_py3.html
