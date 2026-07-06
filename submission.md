# Project 5 — Mixtape Bug Hunt: Submission

Mixtape is a small Flask + SQLAlchemy social music app where friends share songs,
rate them, build collaborative playlists, and keep listening streaks. This document
has three parts:

1. **Codebase Map** — how the app is put together and how data flows through it.
2. **Bug Fix Write-Ups** — the three bugs I fixed (Issues #1, #4, #5), plus one
   issue I investigated and found was not actually a bug (#3).
3. **AI Usage** (below, first) — an honest account of how I used an AI assistant.

After all three fixes, the full test suite is green: **13 passed**.

---

## AI Usage

I used an AI coding assistant (Claude Code, running in my IDE) throughout this
project — mostly as a guide for navigating an unfamiliar codebase and as a pair for
reproducing and fixing the bugs. Being specific about the collaboration:

**What I asked it to explain, trace, or summarize.**

- Explain [`models.py`](models.py) — the entities and their relationships — so I
  understood the data model before reading any logic.
- Summarize what the [`services/`](services/) and [`routes/`](routes/) directories
  do. This gave me the routes → services → models mental model I then used to hunt
  every bug.
- Trace Issue #1 from symptom to root cause: I asked it to follow the call chain
  from the `/listen` endpoint into the service. It walked me
  `routes/songs.py` → `record_listening_event()` → `update_listening_streak()` and
  pointed at the weekday clause.
- For the boundary bugs (#1, #5), I asked it to reason through "what state does the
  app need to be in to hit this code path" *before* I trusted any fix.
- I also had it draft the one-line fixes and the first draft of this write-up, which
  I then reviewed and checked against the passing test suite.

**What it helped me understand.**

- The strict three-layer architecture, which made the codebase predictable — once I
  saw that routes never hold logic, I stopped reading routes closely and jumped
  straight to the services, which is where all the real bugs live.
- A SQLAlchemy detail I didn't know: the legacy `db.session.query(Model).all()` API
  de-duplicates entity rows by primary key. That single fact is what explained the
  Issue #3 result.

**Where I had to verify things myself, or the AI was incomplete or wrong.**

- **Issue #3 was the biggest one.** When the assistant first summarized the services,
  it confidently listed the search `outerjoin` as a likely duplicate bug ("a song
  with N tags appears N times"). That's true of the raw SQL but *wrong about the
  observable behavior*. I only caught it by running `pytest tests/test_search.py`
  (all passing) and a direct probe: `search_songs("Crown Heights")` returned 1 row
  while the raw join returned 3. So the AI's first explanation pointed me in the
  wrong direction until the ORM de-duplication was accounted for — running the code
  is what corrected it.
- When explaining `models.py`, it flagged friendships as "not truly symmetric," but
  [`seed_data.py`](seed_data.py) inserts both directions manually, so with the real
  data it works fine — the AI's first pass overstated that as a defect.
- For the data-flow diagram, the project prompt suggested "sharing a song triggers a
  notification." I had the AI check, and sharing does **not** emit a notification in
  this code (only rating and playlist-adds do), so I documented a real flow instead
  of one that doesn't exist.
- **The UTC streak sub-issue:** the AI reasoned it through, but neither of us could
  reproduce it at runtime, because the `User` model stores no timezone. I confirmed
  it by reading the code rather than trusting the explanation, and deliberately left
  it unfixed (a proper fix needs a schema change).

**Overall.** The AI was strongest at fast navigation and explaining framework
behavior, and weakest when it reasoned from the code *without running it* — every
"this is a bug" claim only became trustworthy after I reproduced it with a test or a
script. I treated its explanations as leads to verify, not as conclusions.

---

## 1. Codebase Map

### 1.1 Main files and what each does

| File | Layer | Responsibility |
|------|-------|----------------|
| [`app.py`](app.py) | App setup | Flask application factory (`create_app`). Configures the SQLite DB, registers the four blueprints under their URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and creates tables. |
| [`models.py`](models.py) | Data | All SQLAlchemy models and association tables. Every entity uses a string UUID primary key and UTC timestamps. |
| [`seed_data.py`](seed_data.py) | Data | Populates the DB with realistic test data — 5 users with friendships, 25 songs with varying tag counts, 3 playlists, listening events, streaks, and a sample notification. |
| [`routes/songs.py`](routes/songs.py) | HTTP | Song search, detail, rating, and listen endpoints. |
| [`routes/playlists.py`](routes/playlists.py) | HTTP | Playlist create, detail, list-songs, and add-song endpoints. |
| [`routes/users.py`](routes/users.py) | HTTP | User profile, streak, and notification endpoints. |
| [`routes/feed.py`](routes/feed.py) | HTTP | "Friends listening now" and activity feed endpoints. |
| [`services/streak_service.py`](services/streak_service.py) | Logic | Records listening events and maintains the consecutive-day listening streak. |
| [`services/notification_service.py`](services/notification_service.py) | Logic | Creates/reads notifications and the actions that trigger them (adding a song to a playlist, rating a song). |
| [`services/feed_service.py`](services/feed_service.py) | Logic | Builds the "friends listening now" (last 24h, one song per friend) and general activity feeds. |
| [`services/playlist_service.py`](services/playlist_service.py) | Logic | Playlist creation and retrieval, including songs in `position` order. |
| [`services/search_service.py`](services/search_service.py) | Logic | Song search by title/artist (case-insensitive) and single-song lookup. |
| [`tests/`](tests/) | Tests | `test_streaks.py`, `test_search.py`, `test_playlists.py` — pytest suites against an in-memory SQLite DB. |

### 1.2 The data model

The core entities in [`models.py`](models.py):

| Model | Represents | Key fields |
|-------|-----------|-----------|
| `User` | An account | `username`, `email`, `listening_streak`, `last_listened_at` |
| `Song` | A shared track | `title`, `artist`, `album`, `genre`, `shared_by` (→ `User`), `share_note` |
| `Tag` | A label for songs | `name` (unique) |
| `ListeningEvent` | A play/listen log | `user_id`, `song_id`, `listened_at` |
| `Rating` | A user's score of a song | `score` (1–5), unique per (user, song) |
| `Playlist` | A song collection | `name`, `created_by`, `is_collaborative` |
| `Notification` | An in-app message | `notification_type`, `body`, `read` |

Relationships are wired through association tables: `friendships` (self-referential
many-to-many on `User`), `song_tags` (`Song` ↔ `Tag`), and `playlist_entries`
(`Playlist` ↔ `Song`, carrying `position`, `added_by`, and `added_at`).

### 1.3 Data flow example: rating a song triggers a notification

This traces one full feature from HTTP request to database side effect:

```
POST /songs/<song_id>/rate   { "user_id": B, "score": 5 }
        │
        ▼
routes/songs.py  ·  rate()                      # parse JSON, validate presence of args
        │  calls
        ▼
notification_service.rate_song(B, song_id, 5)   # business logic
        ├─ validate score is 1–5
        ├─ load Song and rater (User)
        ├─ upsert the Rating row  ──► db.session.commit()      # (1) rating saved
        └─ if song.shared_by != B:
               create_notification(...)                        # (2) notify the sharer
                    └─ Notification row  ──► db.session.commit()
        │  returns Rating
        ▼
routes/songs.py returns  201  { rating.to_dict() }
```

The song's original sharer (`song.shared_by`) later reads that notification via
`GET /users/<id>/notifications` → `notification_service.get_notifications()`. The
"add a song to a playlist" feature follows the identical shape: the route calls
`add_to_playlist()`, which mutates data and then calls `create_notification()` for
the sharer using the same `song.shared_by != actor` guard.

The listening-streak feature is the same pattern too:
`POST /songs/<id>/listen` → `streak_service.record_listening_event()` →
`update_listening_streak()`, which writes the streak back onto the `User` row.

### 1.4 Patterns I noticed

- **Strict three-layer split: routes → services → models.** Routes never contain
  business logic — they parse the request, call exactly one service function, and
  translate the result (or a `ValueError`) into JSON. All real logic lives in
  `services/`, and all persistence lives in `models.py`. This is why every bug in
  this project lives in the service layer, and why the fastest way to find one is
  to trace from the route into the service it calls.
- **Consistent error handling.** Services raise `ValueError` for "not found" or bad
  input; routes wrap the call in `try/except ValueError` and return `404`/`400`.
- **`to_dict()` on every model.** Serialization lives on the model, so services and
  routes pass plain dicts around and never leak ORM objects into responses.
- **UUID primary keys + UTC timestamps everywhere**, generated by defaults on the
  models (`generate_uuid`, `datetime.now(timezone.utc)`).
- **Notifications are a shared side effect.** Multiple actions (playlist-add, rating)
  converge on the same `create_notification()` helper with the same
  "don't notify yourself" guard — a reusable pattern rather than one-off code.

---

## 2. Bug Fix Write-Ups

I fixed **Issues #1, #4, and #5**. Each write-up below has the five required fields.
Reproduction evidence comes from `pytest tests/` and short scripts run against a
fresh in-memory DB (same data shape as [`seed_data.py`](seed_data.py)).

### Issue #1 — "My listening streak keeps resetting"

**How I reproduced it.** Before changing anything I ran the existing streak tests to
watch the bug fail on its own: `pytest tests/test_streaks.py -v`. Four tests passed
but `test_streak_increments_on_sunday` failed. It performs the reported action:

```
update_listening_streak(u, Sat 2024-06-15 12:00 UTC)  → streak becomes 1
update_listening_streak(u, Sun 2024-06-16 12:00 UTC)  → streak stays 1  (assert 1 == 2 fails)
```

Saturday-then-Sunday is a consecutive day, so it should reach 2. The trigger is very
specific: an existing streak **+** a listen exactly one day after the last one **+**
that day being a **Sunday**. The "only on Sundays" detail is why a user perceives it
as random resetting.

**How I found the root cause.** Starting from the README issue table (which pointed
at `streak_service.py`) I traced route → service:

1. [`routes/songs.py:43`](routes/songs.py#L43) — `POST /songs/<song_id>/listen` does
   no logic; it just calls `record_listening_event()`.
2. [`streak_service.py:14`](services/streak_service.py#L14) — `record_listening_event()`
   creates the `ListeningEvent`, then delegates to `update_listening_streak(user, now)`.
3. [`streak_service.py:42`](services/streak_service.py#L42) — `update_listening_streak()`
   is the end of the chain, where the date math lives.

The moment I was sure: line 73 read
`elif days_since_last == 1 and today.weekday() != 6:`. The failing test used a Sunday,
and `weekday() == 6` **is** Sunday — so that `and` clause was exactly what pushed a
consecutive-day Sunday into the reset branch. That matched the "only Sundays" pattern
precisely.

**The root cause.** The consecutive-day increment carried an extra, undocumented
condition: `and today.weekday() != 6`. On any Sunday, even when the user listened the
day before (`days_since_last == 1`, which should increment), the condition was `False`,
so control fell into the `else` and reset the streak to 1. Nothing in the streak rules
mentions weekdays — the clause simply should not exist.

**My fix and side-effect check.** One-line change in
[`streak_service.py:73`](services/streak_service.py#L73):

```python
elif days_since_last == 1:          # was: ... and today.weekday() != 6
    user.listening_streak += 1
```

This is a boundary-condition bug, so I verified both sides of the `days_since_last`
boundary — all existing tests pass now:

| Condition | Expected | Test |
|-----------|----------|------|
| `== 0` (same day) | no change | `test_streak_does_not_double_count_same_day` |
| `== 1` (consecutive) | +1, on every weekday | `test_streak_increments_on_consecutive_day`, `_on_sunday` |
| `> 1` (skipped a day) | reset to 1 | `test_streak_resets_after_skipped_day` |
| first-ever listen | 1 | `test_streak_starts_at_1_for_new_user` |

The only other reader of this data, `get_streak()`
([`streak_service.py:81`](services/streak_service.py#L81)), just returns the stored
value and is unaffected. `weekday()` appears nowhere else, so no companion change was
needed.

> **Out of scope, left unfixed on purpose:** the same function computes "today" from a
> UTC datetime (`now.date()`, line 56), so day boundaries are UTC midnight, not the
> user's local midnight. Fixing that correctly requires a per-user timezone, which the
> `User` model doesn't store — that's a schema change, not a targeted fix, so I flagged
> it rather than restructure the model.

### Issue #4 — "I got notified when a friend added my song to a playlist, but not when they rated it"

**How I reproduced it.** There's no test for this, so I wrote a short script against a
fresh in-memory DB to act out the report:

- user A ("alice") shares song S
- a different user B ("bob") calls `rate_song(B, S, 5)`
- then `get_notifications(A)` to see if A was told

Observed: A had **0** notifications before the rating and **still 0** after. As a
control I ran the working half of the report — `add_to_playlist()` for the same song —
and that **did** create a notification for A. So rating specifically produced nothing.

**How I found the root cause.** Navigation path:

1. README issue table → `notification_service.py`.
2. [`routes/songs.py:29`](routes/songs.py#L29) — the rate action `POST /songs/<id>/rate`
   calls `rate_song()`; the working playlist-add action
   ([`routes/playlists.py:43`](routes/playlists.py#L43)) calls `add_to_playlist()`.
   Both live in the same service file, so I could read the broken one next to the
   working one.
3. [`notification_service.py`](services/notification_service.py) — `add_to_playlist()`
   (line 35) ends with a call to `create_notification()` to tell the sharer. But
   `rate_song()` (line 73) validates the score, upserts the `Rating`, commits, and
   returns — and never calls `create_notification()` at all.

The moment I was sure: side by side, the working function had the
`create_notification()` call and the broken one simply didn't. It wasn't a wrong
condition — it was a **missing step**.

**The root cause.** `rate_song()` never creates a notification. The rating path saves
the `Rating` row and returns; no code produces a `"song_rated"` notification, so the
person who shared the song is never told it was rated.

**My fix and side-effect check.** Right after `rate_song()` commits the rating, I added
a notification to the sharer, copying the exact pattern and self-action guard that
`add_to_playlist()` already uses:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

This adds the missing step. The `shared_by != user_id` guard means rating your own
song doesn't notify you — consistent with the playlist path. Side-effect check: I
re-ran my script — B rating A's song now creates exactly one `"song_rated"`
notification, and A rating their own song creates none. I placed the call **after** the
rating's `commit()` (and `create_notification()` runs its own commit), so the rating is
saved regardless. `get_notifications()` and `mark_as_read()` only read/update rows and
are unaffected; the full suite still passes (13 passed).

> **Behavior choice:** re-rating an already-rated song fires another notification, which
> matches "notify when rated." If notifying only on the first rating is preferred, it's
> a one-line move into the new-rating branch.

### Issue #5 — "The last song in a playlist never shows up"

**How I reproduced it.** I ran `pytest tests/test_playlists.py -v`; two tests failed
against a seeded 5-song playlist:

```
test_playlist_returns_all_songs       → got 4 songs, expected 5
test_playlist_returns_songs_in_order  → got ['Track 1'..'Track 4'], expected ['Track 1'..'Track 5']
```

The trigger is simply any playlist with at least one song — the last song by position
is missing every time. (With the seed data, playlist "Late Night Vibes" has 7 songs but
the endpoint returns 6, dropping the position-7 song "Free Throws" by Hoop Dreams.)

**How I found the root cause.** Navigation path:

1. README issue table → `playlist_service.py`.
2. [`routes/playlists.py:34`](routes/playlists.py#L34) — `GET /playlists/<id>/songs`
   just calls `get_playlist_songs()` and wraps the result with a `count`.
3. [`playlist_service.py:38`](services/playlist_service.py#L38) — `get_playlist_songs()`.
   The query is correct: it joins `playlist_entries` and orders by `position` ascending,
   so all rows come back in order. But the return statement at line 66 was:

   ```python
   return [song.to_dict() for song in songs[:-1]]
   ```

The moment I was sure: the `[:-1]` slice. The query returned every song, then the
return line deliberately sliced off the last element — the exact spot the last song
disappears, and why the "in order" test loses "Track 5" (the highest position).

**The root cause.** The slice `songs[:-1]` drops the final element of the result list.
Because songs are ordered ascending by `position`, the last element is the
highest-position (most recently added) song, so it's always omitted — even though the
function's docstring promises "all songs in the playlist." For N songs it returns N-1;
for a 1-song playlist it returns an empty list.

**My fix and side-effect check.** One-line change in
[`playlist_service.py:66`](services/playlist_service.py#L66):

```python
return [song.to_dict() for song in songs]     # was: songs[:-1]
```

This is a boundary bug at the end of the list, so I verified both sides — all tests
pass now:

| Condition | Expected | Test |
|-----------|----------|------|
| non-empty playlist | returns ALL N songs, in order | `test_playlist_returns_all_songs`, `_in_order` |
| empty playlist | returns `[]` | `test_empty_playlist_returns_empty_list` |

The empty case matters: the old `[][:-1]` was also `[]`, so an empty playlist looked
fine before and must still be fine after — it is. Other playlist code is unaffected:
the route's `count` is now accurate, and `get_playlist()` / `get_user_playlists()`
return metadata without calling `get_playlist_songs()`. Full suite: 13 passed.

---

## 3. Investigated but Not a Bug — Issue #3

**Issue #3 — "The same song keeps showing up twice in search."** I investigated this
and could **not** reproduce it, so I did not change any code. I set up a song with 3
tags next to 0-tag and 1-tag songs and searched for it:

- `pytest tests/test_search.py` passes, including `test_search_no_duplicates_multi_tag_song`.
- A direct probe showed `search_songs("Crown Heights")` returns **1** row, while the
  raw outer-join `SELECT` returns **3** rows.

The reason: `search_songs()` uses the legacy `db.session.query(Song).all()` API, which
de-duplicates entity rows by primary key, so the 3 join rows collapse back to a single
`Song`. The `outerjoin` on `song_tags` is effectively dead code but currently harmless.
If I wanted to harden it against a future port to 2.0-style `select()` I'd drop the
join or add `.distinct()` — but that's cleanup, not a behavior fix, so I left the code
as-is and recorded the finding here.
