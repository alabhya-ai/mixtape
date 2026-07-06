# Mixtape Bug Hunt — Submission

---

## AI Usage

I used Claude Code for codebase navigation and initial orientation — specifically to trace the call chain from routes through services and understand how models related to each other (e.g. how `ListeningEvent` connects the feed, streak, and notification flows). This saved time that would otherwise go to reading every file top to bottom.

For each bug, the AI flagged likely suspects during the read-through — the Sunday guard in `streak_service.py`, the `songs[:-1]` slice, the missing `RECENT_THRESHOLD` value, and the absent notification call in `rate_song()` were all surfaced before I ran any tests. I verified each by reading the relevant test cases and confirming the failure mode matched the diagnosis before making any change. In all five cases the AI's read was correct; no fixes were overridden or rolled back.

---

## Codebase Map

**Mixtape** is a Flask REST API for a social music-sharing platform. It uses SQLAlchemy with SQLite (`instance/mixtape.db`). Users are identified by passing a `user_id` in request bodies — there is no authentication layer.

### File Structure

```
app.py              — Flask app factory, DB init, blueprint registration
models.py           — All SQLAlchemy models
seed_data.py        — Script to populate the DB with test data
requirements.txt    — Flask, SQLAlchemy, pytest, python-dotenv

routes/             — Thin HTTP layer (blueprints)
  songs.py          — /songs/*
  playlists.py      — /playlists/*
  users.py          — /users/*
  feed.py           — /feed/*

services/           — Business logic
  search_service.py      — Song search by title/artist
  streak_service.py      — Listening streak tracking
  playlist_service.py    — Playlist CRUD
  notification_service.py — Ratings + playlist-add notifications
  feed_service.py        — Friends activity feed

tests/
  test_streaks.py
  test_search.py
  test_playlists.py
```

### Data Models

| Model | Key fields |
|---|---|
| `User` | username, email, `listening_streak`, `last_listened_at` |
| `Song` | title, artist, album, genre, `shared_by` (FK→User), tags |
| `Tag` | name (many-to-many with Song via `song_tags`) |
| `Playlist` | name, `created_by`, `is_collaborative`, songs (ordered by `position`) |
| `ListeningEvent` | user_id, song_id, `listened_at` |
| `Rating` | user_id, song_id, score (1–5), unique per user+song |
| `Notification` | user_id, type, body, `read` flag |
| `friendships` | symmetric many-to-many on User |

### API Endpoints

| Route | Description |
|---|---|
| `GET /songs/search?q=` | Search by title or artist |
| `GET /songs/<id>` | Get song detail |
| `POST /songs/<id>/rate` | Rate a song (1–5) |
| `POST /songs/<id>/listen` | Record a listen + update streak |
| `POST /playlists/` | Create a playlist |
| `GET /playlists/<id>` | Get playlist metadata |
| `GET /playlists/<id>/songs` | Get ordered songs in playlist |
| `POST /playlists/<id>/songs` | Add a song (notifies original sharer) |
| `GET /users/<id>` | Get user profile |
| `GET /users/<id>/streak` | Get current listening streak |
| `GET /users/<id>/notifications` | Get notifications (`?unread_only=true`) |
| `POST /users/notifications/<id>/read` | Mark notification read |
| `GET /feed/<id>/listening-now` | Friends active in last 10 minutes |
| `GET /feed/<id>/activity` | Recent friend listening history (limit 20) |

### How a listen flows through the system

```
POST /songs/<id>/listen
  → record_listening_event()       [streak_service]
      → ListeningEvent row saved
      → update_listening_streak()

GET /feed/<id>/listening-now
  → get_friends_listening_now()    [feed_service]
      → queries ListeningEvent for friends within threshold → returns feed
```

The feed is pull-based — nothing is pushed when a listen is recorded. The `ListeningEvent` table is the shared link between the two halves.

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

---

## Issue 3 — The same song keeps showing up twice in search

### How I reproduced it

With seed data loaded, search for a song that has multiple tags — e.g. `GET /songs/search?q=Crown+Heights`. The same song appears three times in the response (once per tag). A song with one tag appears once; a song with no tags appears once. The duplication scales directly with tag count.

### How I found the root cause

`search_songs()` in `services/search_service.py` is the only code path for song search. The query does an `outerjoin` on the `song_tags` association table to enable tag-based filtering:

```python
db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    .filter(...)
    .all()
```

A join against an association table multiplies rows — one result row per matching join row. A song with 3 tags produces 3 join rows and therefore 3 entries in `.all()`. The filter only checks title/artist, not tags, so the join serves no filtering purpose here — it's purely an artifact that causes the duplication.

### The root cause

The `outerjoin` on `song_tags` was left in the query without a corresponding `.distinct()`. SQLAlchemy returns one `Song` object per result row, not one per unique song. For a song with N tags, the join produces N rows, so the song appears N times in the returned list. Songs with no tags produce a single NULL-joined row and appear once, which is why tagless songs were unaffected.

### Fix and side-effect check

Added `.distinct()` before `.all()` to collapse duplicate rows back to one per song:

```python
.distinct()
.all()
```

The `outerjoin` itself can remain — it's harmless once deduplication is applied, and removing it would require restructuring the query. `Song.to_dict()` already loads tags via the `tags` relationship (defined as `lazy="subquery"` on the model), so tag data in the response is unaffected by this change. All five `test_search.py` tests pass.

---

## Issue 4 — I got notified when a friend added my song to a playlist but not when they rated it

### How I reproduced it

With seed data loaded, have one user rate a song shared by another user via `POST /songs/<song_id>/rate`. Then check the sharer's notifications via `GET /users/<sharer_id>/notifications`. No notification appears. By contrast, adding that same song to a playlist via `POST /playlists/<id>/songs` does produce a notification for the sharer.

### How I found the root cause

Both rating and playlist-add flow through `services/notification_service.py` — `rate_song()` and `add_to_playlist()` respectively. Reading `add_to_playlist()` shows it explicitly calls `create_notification()` for the song's sharer at the end. Reading `rate_song()` shows it saves the rating and commits, then returns — with no notification call at all.

### The root cause

`rate_song()` was never wired up to call `create_notification()`. The notification infrastructure existed and was working correctly; the call was simply absent from `rate_song()`. This is a missing feature path, not a logic error — the rating saves correctly, but the side-effect notification that should accompany it was never written.

### Fix and side-effect check

Added a `create_notification()` call inside `rate_song()` after the commit, mirroring the pattern in `add_to_playlist()`. The guard `song.shared_by != user_id` ensures users don't notify themselves when rating their own shared songs:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

The notification fires on both new ratings and re-ratings (score updates), since both are meaningful actions. `create_notification()` is self-contained and has no side effects beyond writing the `Notification` row. Ran the full test suite — all previously passing tests continue to pass.

---

## Issue 5 — The last song in a playlist never shows up

### How I reproduced it

With seed data loaded, add 5 songs to a playlist then call `GET /playlists/<id>/songs`. Only 4 songs are returned — the last one (highest position) is always missing. The count in the response also reflects the truncated list.

### How I found the root cause

`get_playlist_songs()` in `services/playlist_service.py` queries songs ordered by position, then returns the result. The return statement was:

```python
return [song.to_dict() for song in songs[:-1]]
```

The `[:-1]` slice immediately stood out — it's Python's "all items except the last" slice, applied to the ordered query results. There's no domain reason to exclude the final song; it's a straightforward off-by-one error in the return value.

### The root cause

`songs[:-1]` slices off the last element of the list. Since songs are ordered ascending by position, the last element is always the song with the highest position number — i.e., the most recently added song in the playlist. Every playlist call silently drops it. An empty playlist returns an empty list (no element to slice off), which is why `test_empty_playlist_returns_empty_list` passed despite the bug.

### Fix and side-effect check

Removed the slice, returning the full list:

```python
return [song.to_dict() for song in songs]
```

No other code in the service or routes modifies the returned list. The ordering logic (ascending by `position`) and the playlist existence check are both unchanged. All 13 tests across the full suite now pass.

---

## Git Log

```
9c76da1 fix: return all playlist songs instead of slicing off the last one
561259f fix: send notification to song sharer when their song is rated
533331e fix: deduplicate search results for songs with multiple tags
d6c9a71 fix: reduce listening-now threshold from 24 hours to 10 minutes
55abb74 fix: fixed Issue 1
2dfdeaa Add .gitignore file and update README with setup instructions
7b64551 initial commit
```
