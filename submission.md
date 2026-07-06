models.py:


Model	Represents	Key fields
User	An account	username, email, listening_streak, last_listened_at
Song	A shared track	title, artist, album, genre, shared_by (→ User), share_note
Tag	A label for songs	name (unique)
ListeningEvent	A play/listen log	user_id, song_id, listened_at
Rating	A user's score of a song	score (1–5), unique per (user, song)
Playlist	A song collection	name, created_by, is_collaborative
Notification	An in-app message	notification_type, body, read flag


Services directory:
 
File	Responsibility
feed_service.py	Builds social feeds — "Friends Listening Now" (friends active in last 24h, deduped to one song each) and a general activity feed of recent friend listens.
notification_service.py	Creates/reads notifications, plus the interactions that trigger them: adding a song to a playlist and rating a song (1–5, upsert per user/song).
playlist_service.py	Playlist CRUD — create, fetch metadata, fetch a user's playlists, and get a playlist's songs in position order.
search_service.py	Song search by title/artist (case-insensitive ILIKE) and single-song lookup.
streak_service.py	Records listening events and maintains the consecutive-day listening streak on the user.


/routes:
The routes/ directory is the HTTP layer — it defines the app's REST API endpoints. Each file is a Flask Blueprint that maps URLs to handler functions, which parse the request, delegate to a service, and return JSON.

It's a clean three-layer split: routes (HTTP) → services (business logic) → models (database). Routes contain no business logic — they just validate input, call a service, and translate the result (or a ValueError) into a JSON response with the right status code.

The four blueprints
File	Blueprint	Endpoints
users.py	users_bp	GET /<id>, GET /<id>/streak, GET /<id>/notifications, POST /notifications/<id>/read
feed.py	feed_bp	GET /<id>/listening-now, GET /<id>/activity
songs.py	songs_bp	GET /search, GET /<id>, POST /<id>/rate, POST /<id>/listen
playlists.py	playlists_bp	POST /, GET /<id>, GET /<id>/songs, POST /<id>/songs


===========================================================
BUG WRITE-UPS
===========================================================
Each issue is traced route -> service -> function, with the exact
state/data condition needed to hit the code path, how I reproduced it,
and the fix. Repro evidence is from `pytest tests/` and small scripts
run against a fresh in-memory DB (same shape as seed_data.py).


-----------------------------------------------------------
ISSUE #1 — "My listening streak keeps resetting" (streak_service.py)
-----------------------------------------------------------
Call chain (shared by both sub-bugs below):
  POST /songs/<song_id>/listen   (routes/songs.py:43)
    -> record_listening_event(user_id, song_id)   (streak_service.py:14)
       -> update_listening_streak(user, now)      (streak_service.py:42)
Precondition for either bug: user.last_listened_at is NOT None. A first-ever
listen short-circuits at line 58 (streak = 1) and never reaches the buggy code.

--- 1a. Bogus Sunday rule (streak_service.py:73) ---
Code:
    elif days_since_last == 1 and today.weekday() != 6:   # 6 == Sunday
        user.listening_streak += 1
    else:
        user.listening_streak = 1
Root cause: the extra `and today.weekday() != 6` clause has nothing to do with
the documented rules. When today is a Sunday the increment branch is skipped
and control falls to `else`, resetting a valid streak to 1.

State needed to hit it:
  - listening_streak already > 0 (an existing streak to lose),
  - last_listened_at is exactly one calendar day before now (days_since_last == 1),
  - now falls on a Sunday (today.weekday() == 6).
It is invisible 6 days out of 7 — only a Saturday->Sunday consecutive listen
triggers it, which is why the report says "keeps resetting" without an obvious cause.

How I reproduced it:
  `pytest tests/test_streaks.py::test_streak_increments_on_sunday` -> FAILED.
  Sequence inside the test:
    update_listening_streak(u, Sat 2024-06-15 12:00 UTC)  -> streak == 1  (ok)
    update_listening_streak(u, Sun 2024-06-16 12:00 UTC)  -> streak == 1
  Observed: `assert 1 == 2` fails. Expected 2 (consecutive day), got 1 (reset).

Fix: delete the weekday clause.
    elif days_since_last == 1:
        user.listening_streak += 1

--- 1b. Day boundaries computed in UTC (streak_service.py:56) ---
Code:
    today = now.date()            # now = datetime.now(timezone.utc)
    last_date = last_listened.date()
    days_since_last = (today - last_date).days
Root cause: "which day is it" is decided at UTC midnight, not the user's local
midnight, so evening / early-morning listens land in the wrong calendar day.

State needed to expose it: a user in a non-UTC timezone listening near their
local midnight, so the UTC date and their local date disagree. Example (UTC-5):
  - local Mon 9:00 PM = Tue 02:00 UTC  and  local Tue 8:00 PM = Wed 01:00 UTC
    -> seen as Tue and Wed = consecutive (accidentally fine), but
  - local Mon 9:00 PM (Tue 02:00 UTC) then local Tue 7:00 AM (Tue 12:00 UTC)
    -> both map to Tue UTC, days_since_last == 0, so a genuine new-day listen
       is treated as "already listened today" and the streak never advances.

How I reproduced it: NOT reproduced at runtime — the User model stores no
timezone and the /listen endpoint uses the real clock, so I can't drive a
specific local-vs-UTC boundary through the API. Confirmed by reading the code:
`now.date()` on a UTC datetime is a UTC calendar date. This is a latent
correctness issue rather than one the current tests exercise.

Fix: needs a user timezone to be correct. (a) add a `timezone` field to User,
(b) convert `now` and `last_listened_at` into that zone before `.date()`.
Without a stored tz there is no correct local date to compute, so the honest
minimum is to document the UTC assumption; the real fix requires the tz field.


-----------------------------------------------------------
ISSUE #3 — "The same song shows up twice in search" (search_service.py)
-----------------------------------------------------------
Call chain:
  GET /songs/search?q=...   (routes/songs.py:11)
    -> search_songs(query)  (search_service.py:11)
Suspect code:
    db.session.query(Song)
      .outerjoin(song_tags, Song.id == song_tags.c.song_id)
      .filter(or_(Song.title.ilike(...), Song.artist.ilike(...)))
      .all()
Theory of the bug: the outerjoin to song_tags produces one SQL row per
(song, tag) pair, so a song with 3 tags should come back 3 times — and only
tagged songs would duplicate, matching the "inconsistent" wording.

State I set up to hit it: a song ("Crown Heights Anthem" / "Borough Kings")
with 3 tags, alongside 0-tag and 1-tag songs, then searched a term matching
the multi-tag song's title/artist.

How I reproduced it: ATTEMPTED, but the bug DID NOT manifest.
  - `pytest tests/test_search.py` -> all pass, including
    test_search_no_duplicates_multi_tag_song (asserts exactly 1 result).
  - Direct probe on a fresh DB:
        search_songs("Crown Heights")            -> 1 row
        raw outerjoin SELECT (pre-ORM-uniquing)  -> 3 rows
Explanation: the join really does emit 3 rows, but `db.session.query(Song).all()`
uses SQLAlchemy's legacy Query API, which de-duplicates entity rows by primary-key
identity. So the 3 rows collapse back to 1 Song object and the user never sees a
duplicate. The outerjoin is effectively dead code (nothing filters or selects on
it), but it is currently harmless.

Fix (cleanup / defensive, not a behavior change today): remove the pointless
outerjoin, or make the de-dup explicit with `.distinct()`, so correctness no
longer relies on the ORM's implicit uniquing (which does NOT apply if this is
ever ported to 2.0-style `select()` without `.unique()`).


-----------------------------------------------------------
ISSUE #4 — "Notified when a song is added to a playlist, but not when
            it is rated" (notification_service.py)
-----------------------------------------------------------
Call chains:
  works:  POST /playlists/<id>/songs -> add_to_playlist() -> create_notification()
  broken: POST /songs/<id>/rate      -> rate_song()       -> (no notification)
Root cause: add_to_playlist (notification_service.py:65) notifies the song's
sharer, but rate_song (notification_service.py:73) saves the Rating and returns
— it never calls create_notification, so the sharer is never told.

State needed to hit it: user A shares a song; a different user B rates it
(score 1-5). Expected: A receives a "song_rated" notification. Actual: none.
(If the rater IS the sharer, no notification is desired anyway — mirror the
`song.shared_by != user_id` guard that add_to_playlist already uses.)

How I reproduced it: script on a fresh DB —
    A shares song S;  B calls rate_song(B, S, 5);  get_notifications(A)
  Observed: A's notification count = 0 before AND 0 after the rating; types = [].
  Contrast: calling add_to_playlist for the same song DOES create one. This
  matches the seeded "song_added_to_playlist" notification that exists while no
  "song_rated" notification type is ever produced.

Fix: in rate_song, after the commit, notify the sharer (guarding self-ratings):
    if song.shared_by != user_id:
        create_notification(
            user_id=song.shared_by,
            notification_type="song_rated",
            body=f"{rater.username} rated your song '{song.title}' {score}/5.",
        )


-----------------------------------------------------------
ISSUE #5 — "The last song in a playlist never shows up" (playlist_service.py)
-----------------------------------------------------------
Call chain:
  GET /playlists/<id>/songs   (routes/playlists.py:34)
    -> get_playlist_songs(playlist_id)   (playlist_service.py:38)
Root cause (playlist_service.py:66):
    return [song.to_dict() for song in songs[:-1]]
The `[:-1]` slice drops the last element. Since songs are ordered ascending by
position, the highest-position (most recently added) song is always omitted,
even though the docstring promises "all songs in the playlist."

State needed to hit it: any playlist with >= 1 song. With N songs the endpoint
returns N-1; a 1-song playlist returns an empty list. (An already-empty playlist
returns [] either way, so it looks fine — masking the bug for empty playlists.)

How I reproduced it: `pytest tests/test_playlists.py` -> 2 failures:
  - test_playlist_returns_all_songs:   len(songs) == 4, expected 5.
  - test_playlist_returns_songs_in_order: got ['Track 1'..'Track 4'],
    expected ['Track 1'..'Track 5'] — "Track 5" (last position) is missing.
  With seed_data, playlist "Late Night Vibes" holds 7 songs but the endpoint
  returns 6, dropping the position-7 song ("Free Throws" by Hoop Dreams).

Fix: iterate over the full list.
    return [song.to_dict() for song in songs]