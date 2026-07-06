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

---

## Issue 2 — Friends Listening Now shows people from yesterday

### How I reproduced it

With seed data loaded, record a `ListeningEvent` for a friend with a timestamp from 23 hours ago, then call `GET /feed/<user_id>/listening-now`. The friend appears in the feed despite not having listened recently. Any event within the past 24 hours would show up, meaning someone who listened yesterday evening still appears as "listening now" the following morning.

### How I found the root cause

The feed endpoint calls `get_friends_listening_now()` in `services/feed_service.py`. That function filters events against a `cutoff` computed as `datetime.now(timezone.utc) - RECENT_THRESHOLD`. The threshold was defined at the top of the file:

```python
RECENT_THRESHOLD = timedelta(hours=24)
```

A 24-hour window for "listening now" is far too wide — it's the same window used by the activity feed. The constant name `RECENT_THRESHOLD` and the feature name "Listening Now" both imply a short, present-tense window. 24 hours is not that.

### The root cause

`RECENT_THRESHOLD` was set to `timedelta(hours=24)`, meaning any friend who listened within the past 24 hours qualified as "listening now." A user who listened at 10pm last night would still appear in the feed at 9pm the following day. The filtering logic itself is correct — it was only the threshold value that was wrong.

### Fix and side-effect check

Changed the threshold to 10 minutes:

```python
RECENT_THRESHOLD = timedelta(minutes=10)
```

This constant is used only in `get_friends_listening_now()` — `get_activity_feed()` has no recency filter and is unaffected. Ran the full test suite after the change; all previously passing tests continue to pass.
