The routes has 4 files: feed.py, playlists.py, songs.py, and users.py.  
- in feed.py, it takes in get_friends_listening_now() and get_activity_feed() from feed_service.py  it returns a json object of feed and count from listening now, it also returns their activity by feed and count 
- in playlists.py, it takes in create_playlist, get_playlist_songs, and get_
playlist 
    - when someone "posts" it creates a playlist with the data, name, and created by and returns that as a dictionary 
    - when someone "visits" the playlist id are of flask, they get the playlist
    - when someone "visits" the songs, they get a songs a count dictionary
    - when some "posts" songs, the song gets added, 
-in songs.py, when someone searches, they get their query back
    - when someone "vists" by song_id, they get a json object of that song ID
    - when some "posts" a rate, it gets added or an error
    - when someone "posts" a listen, the listening event gets triggered
-in users.py, it gets the user by id, 
    - the streak returns the streak
    - the nofications returns the notifications and marks them as read

## Bug Fixes

### Issue 1: My listening streak keeps resetting

**File:** `services/streak_service.py` — `update_listening_streak()`

**Problem:** The streak only incremented on consecutive days when today was not Sunday. The condition `days_since_last == 1 and today.weekday() != 6` caused listening on Saturday and then Sunday to reset the streak to 1 instead of incrementing it to 2.

**Fix:** Removed the `today.weekday() != 6` check so any consecutive calendar day increments the streak:

```python
elif days_since_last == 1:
    user.listening_streak += 1
```

**Verification:** `pytest tests/test_streaks.py::test_streak_increments_on_sunday`

### Issue 2: Friends Listening Now shows people from yesterday

**File:** `services/feed_service.py` — `get_friends_listening_now()`

**Problem:** `RECENT_THRESHOLD` was set to `timedelta(hours=24)`, so the "listening now" feed included any friend who listened in the past 24 hours — including activity from hours ago or late yesterday. Stale listeners appeared alongside truly recent ones.

**Fix:** Changed the threshold to `timedelta(minutes=30)` so only very recent listening events are included:

```python
RECENT_THRESHOLD = timedelta(minutes=30)
```

**Verification:** `GET /feed/<user_id>/listening-now` after running `python seed_data.py` — only friends who listened in the last ~30 minutes should appear.

### Issue 3: The same song keeps showing up twice in search

**File:** `services/search_service.py` — `search_songs()`

**Problem:** The query joins `song_tags` to include tag data, but a song with multiple tags produces one row per tag. Songs with 3 tags appeared 3 times in search results instead of once.

**Fix:** Added `.distinct()` to the query so each song is returned only once:

```python
.filter(...)
.distinct()
.all()
```

**Verification:** `pytest tests/test_search.py`
