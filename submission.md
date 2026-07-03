# Mixtape Bug Hunt — Submission

## AI Usage

I used Cursor (AI assistant) throughout this project for codebase navigation, concept clarification, and debugging support. Below are the main ways it helped and where I verified or overrode its output.

**Codebase navigation and concepts**
- I asked what the `@` symbol does in Python when I saw `@songs_bp.route` in the routes files. The AI explained decorators and how Flask registers URL handlers. I verified this by tracing a request from `app.py` → blueprint registration → route function.
- I asked what a "route" is in this project. The AI mapped URLs to handler functions and showed how `url_prefix` in `app.py` builds full paths like `/songs/search`. I confirmed this by reading `routes/songs.py` and `app.py` side by side.

**Environment and startup debugging**
- When `flask run` failed, the AI read my terminal output and identified two separate problems: `flask` was using system Python instead of `.venv`, and port 5000 was already in use (AirPlay Receiver + a leftover Python process). I verified with `which flask` and `which python3`, then started the app on port 5001.

**Bug investigation**
- For each service bug, I asked the AI to show me where the bug was. It pointed to specific lines (e.g. `today.weekday() != 6` in `streak_service.py`, `RECENT_THRESHOLD = timedelta(hours=24)` in `feed_service.py`, missing `.distinct()` in `search_service.py`, missing notification call in `rate_song()`). I did not apply fixes blindly — I read the service file, compared against the issue description and tests, then made or confirmed the change myself.
- For the streak bug, the AI explained the Saturday → Sunday failure case. I applied the fix and ran `pytest tests/test_streaks.py::test_streak_increments_on_sunday` to confirm.
- For the search bug, the AI suggested `.distinct()` on the join query. I ran `pytest tests/test_search.py` and all 5 tests passed before committing.

**Where I overrode or verified AI output**
- I fixed the streak bug myself after the AI showed the problematic line; I removed `today.weekday() != 6` rather than accepting any alternative explanation.
- I changed `RECENT_THRESHOLD` to 30 minutes based on the issue title ("listening now") and seed data comments, not just the AI's suggestion in isolation.
- I wrote my own route notes in this file before relying on AI summaries for the codebase map.
- I used conventional commit messages and pushed to `bugfix/mixtape` after reviewing the diff.

---

## Codebase Map

*Written before starting bug work — main files and how the app is organized.*

### Main files and roles

| File / folder | Role |
|---|---|
| `app.py` | Flask app factory, database setup, blueprint registration with URL prefixes |
| `models.py` | SQLAlchemy models: `User`, `Song`, `Playlist`, `ListeningEvent`, `Notification`, `Rating`, and association tables |
| `routes/` | HTTP endpoints — parse requests, call services, return JSON |
| `services/` | Business logic — where the bugs live |
| `seed_data.py` | Populates the database with test users, songs, events, and playlists |
| `tests/` | Pytest suites for streaks, search, and playlists |

### Routes layer

The routes folder has 4 files: `feed.py`, `playlists.py`, `songs.py`, and `users.py`.

- **`feed.py`** — calls `get_friends_listening_now()` and `get_activity_feed()` from `feed_service.py`; returns JSON with `feed` and `count` for listening-now and activity endpoints.
- **`playlists.py`** — calls `create_playlist`, `get_playlist`, and `get_playlist_songs` from `playlist_service.py`; handles creating playlists, viewing a playlist, listing songs, and adding songs.
- **`songs.py`** — calls `search_songs`, `get_song`, `rate_song`, and `record_listening_event`; handles search, song detail, ratings, and listen events.
- **`users.py`** — returns user profile, streak, and notifications; marks notifications as read.

### Data flow example: rating a song

```
POST /songs/<song_id>/rate
  → routes/songs.py  (rate handler)
    → services/notification_service.py  (rate_song)
      → models: Song, User, Rating, Notification
      → db.session commit
  → JSON response with rating dict
```

A similar pattern applies to all features: **route → service → model/DB → JSON response**. Bugs are in the service layer, so I traced each issue from the route file to the service it calls before editing anything.

---

## Root Cause Analysis

Each entry covers: **symptom**, **investigation**, **root cause**, **fix**, and **verification**.

### Issue 1: My listening streak keeps resetting

**Symptom:** Users report their listening streak resets even when they listen on consecutive days — especially across the Saturday → Sunday boundary.

**Investigation:** Traced `POST /songs/<song_id>/listen` → `routes/songs.py` → `streak_service.record_listening_event()` → `update_listening_streak()`. Read `tests/test_streaks.py::test_streak_increments_on_sunday` to see the expected behavior.

**Root cause:** In `update_listening_streak()`, the streak only incremented when `days_since_last == 1 and today.weekday() != 6`. Sunday has `weekday() == 6`, so consecutive Saturday → Sunday listening hit the `else` branch and reset the streak to 1.

**Fix:** Removed the `today.weekday() != 6` check so any consecutive calendar day increments the streak:

```python
elif days_since_last == 1:
    user.listening_streak += 1
```

**Verification:** `pytest tests/test_streaks.py::test_streak_increments_on_sunday` — passes.

---

### Issue 2: Friends Listening Now shows people from yesterday

**Symptom:** The "Friends Listening Now" feed shows friends who listened hours ago or late yesterday, not just people actively listening.

**Investigation:** Traced `GET /feed/<user_id>/listening-now` → `routes/feed.py` → `feed_service.get_friends_listening_now()`. Compared `RECENT_THRESHOLD` against seed data comments (recent events within 30 minutes vs. older events at 2+ hours).

**Root cause:** `RECENT_THRESHOLD` was set to `timedelta(hours=24)`, so any listening event in the past 24 hours qualified as "listening now." That window is far too wide for a real-time feed.

**Fix:** Changed the threshold to `timedelta(minutes=30)`:

```python
RECENT_THRESHOLD = timedelta(minutes=30)
```

**Verification:** After `python seed_data.py`, `GET /feed/<user_id>/listening-now` returns only friends who listened in the last ~30 minutes.

---

### Issue 3: The same song keeps showing up twice in search

**Symptom:** Search results include duplicate entries for the same song — a song with multiple tags can appear once per tag.

**Investigation:** Traced `GET /songs/search` → `routes/songs.py` → `search_service.search_songs()`. Read `tests/test_search.py::test_search_no_duplicates_multi_tag_song`, which expects 1 result but the bug returns 3 for a 3-tag song.

**Root cause:** The query outer-joins `song_tags`. Each tag produces a separate row for the same `Song`, so multi-tag songs are returned multiple times without deduplication.

**Fix:** Added `.distinct()` to the SQLAlchemy query:

```python
.filter(...)
.distinct()
.all()
```

**Verification:** `pytest tests/test_search.py` — all 5 tests pass.

---

### Issue 4: Not notified when a friend rated my song

**Symptom:** Users receive a notification when a friend adds their song to a playlist, but not when a friend rates their song.

**Investigation:** Traced `POST /songs/<song_id>/rate` → `routes/songs.py` → `notification_service.rate_song()`. Compared against `add_to_playlist()` in the same file, which correctly calls `create_notification()` for the song sharer. Seed data also includes a working `song_added_to_playlist` notification as a reference pattern.

**Root cause:** `rate_song()` saved or updated the `Rating` and committed, but never called `create_notification()`. The notification path existed for playlist adds but was missing entirely for ratings.

**Fix:** After committing the rating, notify the song sharer unless they rated their own song:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

**Verification:** `POST /songs/<song_id>/rate` with a different user's ID, then `GET /users/<sharer_id>/notifications` — a `song_rated` notification appears.
