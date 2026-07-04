# Mixtape — Bug Hunt Submission

## Codebase Map

`app.py` is the Flask app factory. It builds the SQLAlchemy `db` object, sets config (SQLite by default), registers the four blueprints, and calls `db.create_all()` on startup. Nothing else happens here.

`models.py` defines seven models: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables. `friendships` is symmetric — no direction to it. `song_tags` is a plain join table. `playlist_entries` carries extra columns beyond the two foreign keys: `position`, `added_by`, `added_at`. Playlist order is explicit state, not insertion order, and those extra columns are `NOT NULL` with no defaults, which matters later. `Rating` has a unique constraint on `(user_id, song_id)` — a second rating updates the row instead of duplicating it.

`routes/` has four blueprints, one file each: `songs`, `playlists`, `users`, `feed`. Every route does the same three things: pull params off the request, call one service function, catch `ValueError` and turn it into a 4xx. No business logic sits in a route file.

`services/` is where the logic and all five bugs live.

- `streak_service.py` — consecutive-day listening streak tracking.
- `feed_service.py` — "friends listening now" (24-hour window) and a general activity feed (most-recent-N, no time filter).
- `search_service.py` — title/artist search via `ilike`, joined against tags.
- `notification_service.py` — creates and reads `Notification` rows. Also owns `add_to_playlist` and `rate_song`, which read as playlist and rating logic but live here because they're the trigger points for notifications.
- `playlist_service.py` — playlist creation and retrieval, including the ordered song list.

`seed_data.py` populates five users with friendships, 25 songs, three playlists, listening events spanning two weeks, and some pre-built notifications. Worth noting: it inserts into `playlist_entries` directly with raw SQL rather than calling `add_to_playlist()`, and it hardcodes one `Notification` row instead of calling `create_notification()`. So the seeded data doesn't actually exercise those service functions — it stages the *result* they're supposed to produce. That distinction turned out to matter when reproducing bug #4 (see below).

`tests/` has `test_streaks.py`, `test_search.py`, `test_playlists.py`. No test file for feed or notifications.

## Data Flow: Adding a Song to a Playlist

`POST /playlists/<playlist_id>/songs` in `routes/playlists.py` reads `song_id` and `added_by` from the body and calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.

That function loads the `Song`, the adding `User`, and the `Playlist`, raising `ValueError` if any is missing (→ 404/400 at the route). If the song isn't already on the playlist, it does `playlist.songs.append(song)` and commits. Then, only if the adder isn't the original sharer, it calls `create_notification()` with type `song_added_to_playlist`.

Two things stand out. The playlist mutation and the notification are separate commits with no shared transaction — if the second commit failed, the song would already be attached with no notification sent. And `playlist.songs.append(song)` only sets `playlist_id` and `song_id` on the underlying `playlist_entries` row. It never sets `position` or `added_by`, both `NOT NULL`. This throws `IntegrityError` on any playlist-add that goes through this function rather than raw SQL (see bonus bug below).

## Pattern Notes

Routes are thin, services are fat — every rule lives in `services/`. Service functions raise `ValueError` for anything that should become a 4xx; routes only check that required fields are present. `notification_service.py` isn't just "handles notifications" — it's the home for any action that causes a notification, even when that action conceptually belongs to another domain (rating, playlist membership).

## The Five Issues

1. Listening streak resets unexpectedly
2. Friends Listening Now shows people from yesterday
3. Search returns duplicate songs
4. Rating a song doesn't notify the sharer, but adding to a playlist does
5. Last song in a playlist never shows up

## Root Cause Analyses

### Bug #1 — Streak resets on Sunday

**Reproduced it:** ran the existing test suite. `tests/test_streaks.py::test_streak_increments_on_sunday` failed. The test sets up a listen on Saturday, June 15 2024 (streak → 1), then a listen the next day, Sunday June 16. Expected streak of 2, got 1.

**How I found the root cause:** the failing test pointed straight at `update_listening_streak()` in `streak_service.py`. Read the function top to bottom and found the increment branch had a second condition tacked onto it beyond the day-gap check.

**Root cause:** `update_listening_streak()` increments the streak with `elif days_since_last == 1 and today.weekday() != 6:`. The `weekday() != 6` clause forces a reset to 1 any time the current listen falls on a Sunday, even when the previous listen was exactly one day earlier. Nothing else in the function references weekday, and the docstring's streak rules don't mention any day-of-week exception, so this clause had no reason to exist.

**Fix:** removed `and today.weekday() != 6`, leaving `elif days_since_last == 1:`. Commit `b807f66`.

**Verification:** full `test_streaks.py` suite passes, 5/5, including `test_streak_resets_after_skipped_day`, confirming the reset-on-gap behavior on the other side of the boundary still works.

### Bug #5 — Last playlist song missing

**Reproduced it:** same pytest run, `tests/test_playlists.py`. `test_playlist_returns_all_songs` expected 5 songs back from a 5-song playlist, got 4. `test_playlist_returns_songs_in_order` expected `["Track 1", ... "Track 5"]`, got the same list minus "Track 5".

**How I found the root cause:** followed the two failing tests into `get_playlist_songs()` in `playlist_service.py`. The function builds the ordered list correctly through a loop over `playlist_entries`, then the return statement slices it. One line accounted for both symptoms — the wrong count and the missing final title.

**Root cause:** `get_playlist_songs()` returns `songs[:-1]` instead of `songs`, dropping the last song regardless of playlist length.

**Fix:** changed the return to `songs`. Commit `f97142a`. Before committing, grepped for other callers to check for hidden dependencies on the truncated list — found one real caller (`routes/playlists.py`) and one dead import in `notification_service.py` that's never invoked in the function it's imported into, so nothing else relied on the old behavior.

**Verification:** all three playlist tests pass, including `test_empty_playlist_returns_empty_list`, confirming the fix doesn't break the zero-song case.

### Bug #4 — Rating doesn't notify the sharer

**Reproduced it:** manually, since no test covers this path. Picked a song (`5e244b2f...`) shared by user `a3365bcd...`, and a different user (`6717df93...`) as the rater. Checked that user's notification count before and after calling `rate_song()` directly in a Python shell:

```
before: 1
after: 1
```

The one existing notification was the seeded `song_added_to_playlist` entry, unrelated to this call. Rating produced no new notification.

**How I found the root cause:** I planned to reproduce the working path (`add_to_playlist`) live through the HTTP route as a contrast, but that call throws a 500 (see the bug noted below). Instead I read `add_to_playlist()` and `rate_song()` side by side in `notification_service.py`. `add_to_playlist()` calls `create_notification()` right after committing the playlist change, gated on `song.shared_by != added_by_user_id`. `rate_song()` saves the `Rating` row, commits, and returns — no equivalent call anywhere in it.

**Root cause:** architectural, not a typo. `notification_service.py` has an established pattern — perform the action, then call `create_notification()` if the actor isn't the song's original sharer. `add_to_playlist()` follows it. `rate_song()` was written without it; the notification call was never added.

**Fix:** added a `create_notification()` call at the end of `rate_song()`, gated on `song.shared_by != user_id`, matching `add_to_playlist()`'s pattern. Commit `3bb2a92`.

One decision worth flagging: I re-rated the same song with a different score and confirmed the current fix sends a notification on every call, not just the first rating. I left it this way since the issue only asks that rating notify the sharer at all, and a changed score seems worth a second notification. If "notify once per song" is the intended behavior instead, the fix needs an added `not existing` check before the notification call, using the `existing` lookup already present earlier in the function.

**Verification:** first rating — before: 1, after: 2. Second rating, same song, different score — before: 2, after: 3. Full suite (13/13) still passes after all three fixes combined.

## Bug Found Outside the Assigned Five

While reproducing #4, calling `add_to_playlist()` through the HTTP route threw a 500:

```
sqlite3.IntegrityError: NOT NULL constraint failed: playlist_entries.position
[SQL: INSERT INTO playlist_entries (playlist_id, song_id, added_at) VALUES (?, ?, ?)]
```

`add_to_playlist()` does `playlist.songs.append(song)`, which inserts a row through the ORM relationship using only `playlist_id` and `song_id`. `position` and `added_by` are both `NOT NULL` with no default on the `playlist_entries` table, so any add-to-playlist call through this function fails on a fresh association. Seed data never hits this because it inserts into `playlist_entries` directly with `position` and `added_by` set explicitly — it never calls `add_to_playlist()` at all. Not fixing this since it's outside the five listed issues, but flagging it since it blocked part of my #4 reproduction and would affect real usage of the add-to-playlist endpoint.

## Plan

Fixing #1, #4, #5 as the required three — I confirmed each with a specific line-level root cause before touching any code. #2 (stale feed) and #3 (duplicate search results) as stretch, once the required three are committed.