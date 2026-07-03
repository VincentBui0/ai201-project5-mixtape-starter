# Mixtape — Bug Hunt Submission

## Codebase Map

**app.py** — Flask app factory. Creates the SQLAlchemy `db` object, wires config
(DB URI defaults to a local sqlite file), registers the four blueprints, and
calls `db.create_all()` on startup. Nothing else lives here.

**models.py** — Seven models: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`,
`Playlist`, `Notification`. Three association tables handle the many-to-many
relationships. `friendships` is symmetric (user_id/friend_id, no direction).
`song_tags` is a plain join table. `playlist_entries` is the interesting one —
it's not just a join table, it carries `position`, `added_by`, and `added_at`
columns, so a playlist's song order is explicit state, not insertion order.
`Rating` has a unique constraint on `(user_id, song_id)`, so a user can only
have one rating per song; a second rating updates the first rather than
creating a duplicate row.

**routes/** — Four blueprints (`songs`, `playlists`, `users`, `feed`), one file
each. Every route follows the same shape: pull params off the request, call
exactly one service function, catch `ValueError` and turn it into a 4xx JSON
response. No business logic lives in a route file — that's a deliberate
pattern, not an accident, and it's why the README says to trace bugs into
`services/`.

**services/** — Where the actual logic and all five bugs live.
- `streak_service.py` — tracks consecutive-day listening streaks.
- `feed_service.py` — "friends listening now" (24hr window) and a general
  activity feed (no time filter, just most-recent-N).
- `search_service.py` — title/artist search with `ilike`, joined against tags.
- `notification_service.py` — creates and reads `Notification` rows;
  also owns `add_to_playlist` and `rate_song`, which is a naming choice worth
  noting since those read as playlist/rating logic but live in the
  notification file because they're the trigger points for notifications.
- `playlist_service.py` — playlist creation and retrieval, including the
  ordered song list.

**seed_data.py** — populates the DB with test users, songs, friendships, and
events so the app has something to query against.

**tests/** — `test_streaks.py`, `test_search.py`, `test_playlists.py`. Existing
coverage to check before and after a fix, so I don't break something that
already passes.

## Data Flow: Adding a Song to a Playlist

`POST /playlists/<playlist_id>/songs` in `routes/playlists.py` reads
`song_id` and `added_by` from the JSON body, validates both are present, and
calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.

Inside that function: it loads the `Song`, the adding `User`, and the
`Playlist`, raising `ValueError` (→ 404/400 at the route layer) if any is
missing. If the song isn't already on the playlist, it appends it to
`playlist.songs` and commits. Then, only if the person adding the song isn't
the person who originally shared it, it calls `create_notification()` with
type `song_added_to_playlist` and a body naming the adder, the song, and the
playlist.

Two things stand out. First, the playlist mutation and the notification are
two separate commits with no shared transaction — if the notification insert
failed, the song would already be on the playlist. Second, this path is the
one that *works*, which is why the notification bug (friend rates a song, no
notification fires) points at `rate_song` in the same file: it saves the
`Rating` row but never calls `create_notification` at all. Same file, same
pattern available to copy, just not used.

## Pattern Notes

- Routes are thin, services are fat. Every business rule sits in `services/`.
- Service functions raise `ValueError` for anything routes should surface as
  a 4xx; routes never do their own validation beyond checking required
  fields are present.
- `notification_service.py` isn't just "handles notifications" — it's the
  home for any action that *causes* a notification, even if that action
  (rating, adding to playlist) belongs conceptually to another domain.

## The Five Issues (read before choosing)

1. Streak resets unexpectedly
2. Friends Listening Now shows stale (yesterday's) activity
3. Search returns duplicate songs
4. Rating a song doesn't notify the sharer (but playlist-add does)
5. Last song in a playlist never appears

## Plan

Fixing #1, #4, #5 first — I have a specific line-level hypothesis for each
from reading the code, and #4 gives me a working pattern (`add_to_playlist`)
to diff against. #2 and #3 as stretch, after I've actually reproduced them
against seeded data rather than guessed.