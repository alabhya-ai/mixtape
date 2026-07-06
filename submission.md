# Mixtape Bug Hunt — Submission

---

## Issue 1 — My listening streak keeps resetting

### How I reproduced it

The test suite already contained a targeted case (`test_streak_increments_on_sunday`) that demonstrated the failure. To confirm manually: set a user's `last_listened_at` to a Saturday, then call `update_listening_streak` with a Sunday timestamp. The streak resets to 1 instead of incrementing to 2. Any Saturday → Sunday listen sequence triggers it; no other day pair is affected.

### How I found the root cause

I opened `services/streak_service.py` and read `update_listening_streak()` — the only function responsible for modifying the streak. The logic has three branches keyed on `days_since_last`. The middle branch, which handles consecutive-day increments, had an extra compound condition on line 73:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

The `today.weekday() != 6` clause had no corresponding business rule — nothing in the spec says Sunday should be treated differently. That mismatch between what the condition checked and what the feature requires was the moment of confidence: this wasn't a suspicious area, it was the cause.

### The root cause

Python's `datetime.weekday()` returns `6` for Sunday. The streak condition for incrementing was `days_since_last == 1 and today.weekday() != 6`. When a user listens on Saturday and then Sunday, `days_since_last` is correctly `1`, but `today.weekday()` is `6`, making `today.weekday() != 6` evaluate to `False`. The entire `elif` becomes `False`, so execution falls through to the `else` branch, which resets the streak to `1`. The bug is triggered specifically and only when the second listen lands on a Sunday — every other consecutive-day pair passes the guard and increments correctly.

### Fix and side-effect check

Removed the `and today.weekday() != 6` guard, leaving the branch as:

```python
elif days_since_last == 1:
    user.listening_streak += 1
```

This makes Sunday behave identically to every other day. The `days_since_last == 0` (same-day no-op) and `else` (gap reset) branches are unchanged and unaffected. Ran all five tests in `tests/test_streaks.py` — all pass, including the pre-existing Saturday → Sunday regression case.
