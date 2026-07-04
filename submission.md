# Mixtape — Bug Hunt Submission

## AI Usage

I used AI for navigation and verification, not for finding bugs before I'd read the code myself.

Most of the AI use was environment troubleshooting rather than codebase work: a venv mismatch pulling in an unrelated project's Python, PowerShell vs. Git Bash differences for curl and env vars, and one case where I left literal placeholder text like `<playlist_id_from_above>` in a URL instead of a real UUID. The error message made that one obvious once pointed out.

For orientation, I had the AI read `models.py`, `app.py`, and the route/service files and summarize what each was doing, then asked it to trace the add-to-playlist data flow specifically. I checked this against the actual files afterward. It matched, and it caught something I'd have missed on a first pass: the playlist mutation and its notification are two separate commits with no shared transaction.

For each bug, I ran the failing test first, then had the AI help me read the suspicious function before deciding what to change. For bug #1 that meant working through what `weekday() != 6` was actually doing inside `update_listening_streak`. For bug #5 it meant isolating the `songs[:-1]` line in `get_playlist_songs`. For bug #4, the hypothesis that `rate_song` never calls `create_notification` came from reading `rate_song` and `add_to_playlist` side by side, not from guessing off the bug title.

The AI got one thing wrong along the way. I found a sixth bug by accident — `add_to_playlist` throws `IntegrityError: NOT NULL constraint failed: playlist_entries.position` when called through the HTTP route. The AI predicted that failure correctly from reading the code before I hit it live. But it also guessed that a dead import of `get_playlist_songs` inside `add_to_playlist` might be doing position math. I checked the function body myself and that import isn't used anywhere in it. Worth remembering: AI reads code fast but doesn't always confirm a line actually executes before reasoning about it.

The AI proposed a fix for bug #4 that notifies the sharer on every re-rate, not just the first one. I tested that myself by rating the same song twice and watching the notification count. I kept the behavior since the issue doesn't specify which is correct, and I wrote up the trade-off in that bug's entry instead of picking one silently.

## Codebase Map

`app.py` is the Flask app factory. It builds the SQLAlchemy `db` object, sets config (SQLite by default), registers the four blueprints, and calls `db.create_all()` on startup. That's all it does.

`models.py` defines seven models: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables. `friendships` is symmetric, no direction to it. `song_tags` is a plain join table. `playlist_entries` carries extra columns beyond the two foreign keys — `position`, `added_by`, `added_at` — so playlist order is explicit state, not insertion order. Those extra columns are `NOT NULL` with no defaults, which turns out to matter later. `Rating` has a unique constraint on `(user_id, song_id)`, so a second rating updates the row instead of duplicating it.

`routes/` has four blueprints, one file each: `songs`, `playlists`, `users`, `feed`. Every route does the same three things — pull params off the request, call one service function, catch `ValueError` and turn it into a 4xx. No business logic sits in a route file.

`services/` is where the logic lives, and where all five bugs live too.

`streak_service.py` tracks consecutive-day listening streaks. `feed_service.py` handles "friends listening now" (24-hour window) and a general activity feed with no time filter, just most-recent-N. `search_service.py` does title/artist search via `ilike`, joined against tags. `notification_service.py` creates and reads `Notification` rows, and also owns `add_to_playlist` and `rate_song` — these read like playlist and rating logic, but they live here because they're the trigger points for notifications. `playlist_service.py` handles playlist creation and retrieval, including the ordered song list.

`seed_data.py` populates five users with friendships, 25 songs, three playlists, two weeks of listening events, and some pre-built notifications. It inserts into `playlist_entries` directly with raw SQL instead of calling `add_to_playlist()`, and it hardcodes one `Notification` row instead of calling `create_notification()`. So the seeded data stages the *result* those service functions are supposed to produce without ever actually exercising them. That distinction mattered when I went to reproduce bug #4.

`tests/` has `test_streaks.py`, `test_search.py`, `test_playlists.py`. Nothing covers feed or notifications.

## Data Flow: Adding a Song to a Playlist

`POST /playlists/<playlist_id>/songs` in `routes/playlists.py` reads `song_id` and `added_by` from the body and calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.

That function loads the `Song`, the adding `User`, and the `Playlist`, raising `ValueError` if any is missing. If the song isn't already on the playlist, it does `playlist.songs.append(song)` and commits. Then, only if the adder isn't the original sharer, it calls `create_notification()` with type `song_added_to_playlist`.

Two things stand out here. The playlist mutation and the notification are separate commits with no shared transaction, so if the second commit failed, the song would already be attached with no notification sent. And `playlist.songs.append(song)` only sets `playlist_id` and `song_id` on the underlying row — it never sets `position` or `added_by`, both `NOT NULL`. That throws `IntegrityError` on any add-to-playlist call that goes through this function instead of raw SQL. More on that below.

## Pattern Notes

Routes are thin, services are fat. Every rule lives in `services/`. Service functions raise `ValueError` for anything that should become a 4xx; routes just check that required fields are present. `notification_service.py` isn't only "handles notifications" — it's the home for any action that causes one, even when that action belongs conceptually to another domain, like rating a song or adding it to a playlist.

## The Five Issues

1. Listening streak resets unexpectedly
2. Friends Listening Now shows people from yesterday
3. Search returns duplicate songs
4. Rating a song doesn't notify the sharer, but adding it to a playlist does
5. Last song in a playlist never shows up

## Root Cause Analyses

### Bug #1 — Streak resets on Sunday

I reproduced this by running the existing test suite. `tests/test_streaks.py::test_streak_increments_on_sunday` failed. The test logs a listen on Saturday, June 15 2024 (streak goes to 1), then a listen the next day, Sunday June 16. It expects a streak of 2 and gets 1.

The failing test pointed straight at `update_listening_streak()` in `streak_service.py`. Reading the function top to bottom, the increment branch had a second condition tacked onto the day-gap check that had no obvious reason to be there.

The root cause: `update_listening_streak()` increments with `elif days_since_last == 1 and today.weekday() != 6:`. That `weekday() != 6` clause forces a reset to 1 any time the current listen lands on a Sunday, even when the previous listen was exactly one day before. Nothing else in the function touches weekday, and the streak rules in the docstring don't mention any day-of-week exception. The clause shouldn't have been there.

I removed `and today.weekday() != 6`, leaving `elif days_since_last == 1:`. Commit `b807f66`. All five tests in `test_streaks.py` pass now, including `test_streak_resets_after_skipped_day`, which confirms the reset-on-gap behavior on the other side of the boundary still works.

### Bug #5 — Last playlist song missing

I reproduced this from the same test run. `test_playlist_returns_all_songs` expected 5 songs back from a 5-song playlist and got 4. `test_playlist_returns_songs_in_order` expected `["Track 1", ... "Track 5"]` and got the same list minus "Track 5".

Both failing tests led into `get_playlist_songs()` in `playlist_service.py`. The function builds the correctly ordered list through a loop over `playlist_entries`, then the return statement slices it. One line accounted for both symptoms at once — the wrong count and the missing final title.

The root cause: `get_playlist_songs()` returns `songs[:-1]` instead of `songs`, dropping the last song regardless of how long the playlist is.

I changed the return to `songs`. Commit `f97142a`. Before committing I grepped for other callers to check for hidden dependencies on the truncated list. There's one real caller (`routes/playlists.py`) and one dead import inside `notification_service.py` that's never invoked in the function it sits in, so nothing else relied on the old behavior. All three playlist tests pass now, including `test_empty_playlist_returns_empty_list`, which confirms the fix doesn't break the zero-song case.

### Bug #4 — Rating doesn't notify the sharer

No test covers this path, so I reproduced it manually. I picked a song shared by one user and rated it as a different user, then checked the sharer's notification count before and after calling `rate_song()` directly in a Python shell:

```
before: 1
after: 1
```

The one existing notification was a seeded `song_added_to_playlist` entry, unrelated to this call. Rating the song produced nothing new.

I'd planned to reproduce the working path (`add_to_playlist`) live through the HTTP route for contrast, but that call throws a 500 — the sixth bug described below. So instead I read `add_to_playlist()` and `rate_song()` side by side in `notification_service.py`. `add_to_playlist()` calls `create_notification()` right after committing the playlist change, gated on the adder not being the sharer. `rate_song()` saves the `Rating` row, commits, and returns. No equivalent call anywhere in it.

This is architectural, not a typo. `notification_service.py` has an established pattern: do the action, then call `create_notification()` if the actor isn't the song's original sharer. `add_to_playlist()` follows it. `rate_song()` was written without it — the call was never added.

I added a `create_notification()` call at the end of `rate_song()`, gated the same way as `add_to_playlist()`. Commit `3bb2a92`.

One decision worth stating plainly: I re-rated the same song with a different score and confirmed this fix sends a notification every time, not just on the first rating. I kept that behavior, since the issue only asks that rating notify the sharer at all, and a changed score seems worth a second notification. If "notify once per song" is the intended behavior instead, the fix needs a `not existing` check added before the notification call, using the `existing` lookup already present earlier in the function.

First rating: before 1, after 2. Second rating, same song, different score: before 2, after 3. The full suite still passes at 13/13 after all three fixes combined.

## Bug Found Outside the Assigned Five

While reproducing #4, calling `add_to_playlist()` through the HTTP route threw a 500:

```
sqlite3.IntegrityError: NOT NULL constraint failed: playlist_entries.position
[SQL: INSERT INTO playlist_entries (playlist_id, song_id, added_at) VALUES (?, ?, ?)]
```

`add_to_playlist()` does `playlist.songs.append(song)`, which inserts a row through the ORM relationship using only `playlist_id` and `song_id`. `position` and `added_by` are both `NOT NULL` with no default, so any add-to-playlist call through this function fails on a fresh association. Seed data never hits this because it inserts into `playlist_entries` directly with `position` and `added_by` set explicitly — it never calls `add_to_playlist()` at all. I didn't fix this since it's outside the five listed issues, but it blocked part of my #4 reproduction and would affect real use of the add-to-playlist endpoint.

## Plan

I fixed #1, #4, and #5 as the required three, each with a specific line-level root cause confirmed before I touched any code. #2 (stale feed) and #3 (duplicate search results) are left as stretch goals.