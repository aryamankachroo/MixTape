# Mixtape — Submission

**Branch:** `bugfix/mixtape` ·
**Submission URL:** https://github.com/aryamankachroo/MixTape/tree/bugfix/mixtape

**Bugs fixed (3):** #1 listening streak resets, #5 last playlist song missing,
#4 no notification when a friend rates your song.

**Commit history (`git log --oneline`, one `fix:` commit per bug):**

```
ae23d90 fix: notify song sharer when a friend rates their song
cd01d90 fix: return all playlist songs instead of dropping the last one
b729af9 fix: increment listening streak on Sundays instead of resetting
d37467e mil 1
```

---

## AI Usage

I used an AI coding assistant (Cursor) throughout this project. Because the assignment is a
bug hunt rather than greenfield coding, I leaned on it mostly for **codebase navigation and
debugging**, not code generation — the actual fixes were 1–8 lines each and the real work
was understanding *why* they were broken.

**Where AI genuinely helped:**

- **Orientation / navigation.** I had it summarize each `services/` file's responsibility
  and trace the call chains end to end (e.g. `POST /songs/<id>/rate` → `routes/songs.py`
  → `notification_service.rate_song()`). This is what let me write the Milestone 1 codebase
  map with real data flows instead of just a file listing.
- **Explaining code I'd already found.** For the streak bug I asked it to confirm the
  semantics of `datetime.weekday()` (Mon=0 … Sun=6) once I'd narrowed the problem to the
  `today.weekday() != 6` clause. Knowing Sunday == 6 is what turned "this looks suspicious"
  into "this is definitely the cause."
- **Reproduction tooling.** It helped me drive the live app through Flask's test client to
  reproduce Issue #4 (rate a song, check the sharer's notification count before/after)
  rather than firing manual HTTP requests.

**Where I had to verify things myself / where the AI was wrong or incomplete:**

- **The AI's first diagnosis of the search bug (#3) was wrong.** Reading the code, the
  `outerjoin(song_tags)` fan-out *looks* like it should return a song once per tag, and that
  was the initial hypothesis. But when I actually ran `tests/test_search.py` and hit
  `GET /songs/search?q=heights`, there were **no duplicates** — SQLAlchemy 2.0 auto-dedupes
  identical entities in a single-entity query, so the bug doesn't manifest in this
  environment. This is exactly the "plausible but wrong" trap: I only caught it by running
  the code. I dropped #3 and picked #4 instead.
- **A bug the AI didn't predict surfaced only by running things.** While side-effect-checking
  Issue #4, the add-to-playlist path threw a pre-existing `IntegrityError` (the `position`
  column on `playlist_entries` is never populated by the ORM `append`). Reasoning about the
  code hadn't flagged it; executing it did. I confirmed via `git log` that it wasn't caused
  by my change and left it out of scope.
- **Every fix was verified by running tests, not by trusting the explanation.** For each bug
  I ran the relevant test file (and the full 13-test suite at the end) and checked both sides
  of the boundary conditions (#1: Saturday→Sunday *and* skipped-day reset; #5: full playlist
  *and* empty playlist).

Overall the workflow that worked was: **I locate the suspicious code → AI helps me
understand it → I verify the diagnosis by actually running the code.** Asking the AI to find
a bug before I'd read the relevant code consistently pointed somewhere plausible but not
always correct (the #3 case being the clearest example).

---

## Milestone 1: Codebase Map

Mixtape is a social music app (Flask + SQLAlchemy + SQLite) where users share songs,
rate them, build collaborative playlists, keep listening streaks, and see what their
friends are listening to. This document maps the codebase: what each file is responsible
for, how a request flows from route to database, and the structural patterns the app
follows.

---

### High-level architecture

The app has three clear layers, and every request passes through them in the same order:

```
HTTP request
   → routes/*.py      (thin: parse input, call one service, format JSON response)
      → services/*.py  (all business logic lives here)
         → models.py   (SQLAlchemy ORM; the only thing that talks to the DB)
```

`app.py` wires the whole thing together with the application-factory pattern, and
`seed_data.py` fills the SQLite database with realistic test data.

---

### Main files and their responsibilities

**`app.py` — application factory + DB setup**
- Defines the global `db = SQLAlchemy()` object that every other module imports.
- `create_app(config=None)` builds the Flask app, configures the SQLite URI
  (`sqlite:///mixtape.db`, overridable via `DATABASE_URL`), initializes `db`, registers
  the four blueprints under URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and
  runs `db.create_all()`.
- This factory is why the app must be started with `FLASK_APP=app:create_app flask run`
  and not `python app.py` — importing the module directly triggers a SQLAlchemy
  double-import error.

**`models.py` — the data model (7 entities + 3 association tables)**
- **User** — username, email, `listening_streak`, `last_listened_at`. Self-referential
  many-to-many `friends` relationship via the `friendships` association table.
- **Song** — title, artist, album, genre, `shared_by` (FK to the User who shared it),
  `share_note`. Many-to-many `tags` via `song_tags`.
- **Tag** — a simple named label; many-to-many with Song.
- **ListeningEvent** — one row per listen: `user_id`, `song_id`, `listened_at`. This is
  the raw data behind streaks and the "listening now" / activity feeds.
- **Rating** — a user's 1–5 score for a song, with a `UniqueConstraint(user_id, song_id)`
  so a user has at most one rating per song. Ratings are their own table (not a column on
  Song).
- **Playlist** — name, `created_by`, `is_collaborative`. Songs are attached through the
  `playlist_entries` association table, which carries extra columns: **`position`** (the
  explicit 1-based order of a song in the playlist), `added_by`, and `added_at`. So
  playlist order is stored explicitly, not derived from insertion order.
- **Notification** — `user_id` (recipient), `notification_type`, `body`, `read` flag.
- Every model has a `to_dict()` used by services to serialize into JSON.

**`routes/` — HTTP layer (blueprints, one per domain)**
- `songs.py` — `GET /songs/search?q=`, `GET /songs/<id>`, `POST /songs/<id>/rate`,
  `POST /songs/<id>/listen`.
- `playlists.py` — `POST /playlists/`, `GET /playlists/<id>`,
  `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs`.
- `users.py` — `GET /users/<id>`, `GET /users/<id>/streak`,
  `GET /users/<id>/notifications`, `POST /users/notifications/<id>/read`.
- `feed.py` — `GET /feed/<id>/listening-now`, `GET /feed/<id>/activity`.
- Every route does the same three things: pull params from the request, call exactly one
  service function, and turn the result (or a raised `ValueError`) into JSON with an
  appropriate status code. There is no business logic in the routes.

**`services/` — business logic (where the bugs live)**
- `streak_service.py` — records listening events and updates a user's consecutive-day
  streak; exposes `get_streak()`.
- `feed_service.py` — `get_friends_listening_now()` (friends active within a recency
  window, deduped to one song per friend) and `get_activity_feed()` (most recent N events
  from friends).
- `search_service.py` — `search_songs()` matches title/artist case-insensitively and
  returns each song with its tags; `get_song()` fetches one song.
- `notification_service.py` — `create_notification()`, `add_to_playlist()` (adds a song
  and notifies the original sharer), `rate_song()` (upsert a rating), plus
  `get_notifications()` / `mark_as_read()`.
- `playlist_service.py` — `create_playlist()`, `get_playlist_songs()` (ordered by
  `position`), `get_playlist()`, `get_user_playlists()`.

**`seed_data.py` — test data generator**
- Drops and recreates all tables, then creates 5 users (nova, darius, simone, kenji,
  aaliya) with friendships, 10 tags, 13 songs (deliberately with 0 / 1 / 3+ tags), 3
  playlists ordered by `position`, listening events both recent and old, streak state, and
  a sample notification. The comments even flag which data is meant to exercise which
  issue.

**`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py`. Run with
`pytest tests/`.

---

### Data flow trace #1 — a user rates a song

1. Client sends `POST /songs/<song_id>/rate` with JSON `{ "user_id", "score" }`.
2. `routes/songs.py::rate()` validates that both fields are present, then calls
   `notification_service.rate_song(user_id, song_id, int(score))`.
3. `rate_song()` checks the score is 1–5, loads the Song and User (raising `ValueError` if
   either is missing), then looks for an existing `Rating` for that `(user_id, song_id)`.
   If found it updates the score (upsert); otherwise it creates a new `Rating`. It commits
   and returns the `Rating`.
4. The route serializes `rating.to_dict()` and returns `201`.

> Worth noting for later milestones: the README's example call chain says rating a song
> should notify the song's sharer, and `rate_song` lives in `notification_service.py`, but
> the current `rate_song()` never calls `create_notification()`. That lines up with
> Issue #4.

### Data flow trace #2 — viewing a playlist's songs

1. Client sends `GET /playlists/<playlist_id>/songs`.
2. `routes/playlists.py::get_songs()` calls `playlist_service.get_playlist_songs(id)`.
3. `get_playlist_songs()` loads the Playlist (404 via `ValueError` if missing), then joins
   `Song` to the `playlist_entries` association table, filters by `playlist_id`, and orders
   by `playlist_entries.position` ascending.
4. It returns `[song.to_dict() for song in songs]`, and the route wraps it as
   `{ "songs": [...], "count": N }`.

---

### Patterns I noticed

- **Strict layering.** Routes never touch the DB directly (the one small exception is
  `users.py::get_user`, which does a simple `db.session.get(User, ...)`). All real logic is
  in `services/`, so when an endpoint misbehaves the fix almost always belongs in a
  service function.
- **`ValueError` as the error channel.** Services raise `ValueError` for "not found" /
  "bad input", and routes catch it and map it to a `400` or `404`. There are no custom
  exception classes.
- **`to_dict()` everywhere.** Serialization is a method on each model, so services return
  plain dicts/lists and routes just `jsonify` them.
- **Association tables carry data.** `playlist_entries` (position/added_by/added_at) and
  `friendships` (symmetric, inserted in both directions by the seeder) are more than plain
  join tables.
- **UUID string primary keys** are generated in Python via `generate_uuid()`, not by the
  database.
- **Timezone-aware UTC** timestamps are used consistently (`datetime.now(timezone.utc)`),
  and streak logic even re-attaches `tzinfo` to naive DB values before comparing.

---

### The five open issues (from README)

| # | Title | Affected service |
|---|-------|-----------------|
| 1 | My listening streak keeps resetting | `streak_service.py` |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` |
| 3 | The same song keeps showing up twice in search | `search_service.py` |
| 4 | Notified when a friend added my song to a playlist, but not when they rated it | `notification_service.py` |
| 5 | The last song in a playlist never shows up | `playlist_service.py` |

**Rough plan:** each issue maps cleanly to one service function, which makes them
independent and easy to tackle one commit at a time. I'll start with the three whose root
cause is easiest to pin down from the data flow above.

---

## Milestone 2: Reproducing the Bugs (before any fix)

**Chosen bugs: #1 (streak resets), #4 (no notification on rate), #5 (last playlist song missing).**

I originally intended to take Issue #3 (search duplicates) but could not reproduce it in
this environment (see the note at the bottom of this section), so I swapped it for Issue #4.

No code has been changed yet — everything below is reproduction only. Two of the three
bugs have pre-written tests that encode the *correct* behavior, so a failing test is a
clean, repeatable reproduction. The third (#4) I reproduced against the live app.

Baseline: `python seed_data.py` then `python -m pytest tests/ -v` →
**3 failed, 10 passed**, matching the three chosen bugs.

### Issue #1 — "My listening streak keeps resetting" (`streak_service.py`)

- **How I reproduced it:** Run the pre-written test
  `tests/test_streaks.py::test_streak_increments_on_sunday`. It listens on a Saturday
  (`2024-06-15`, `weekday()==5`) and then the next day, Sunday (`2024-06-16`,
  `weekday()==6`).
- **Data condition that triggers it:** the *second* consecutive listen lands on a **Sunday**.
  Any consecutive-day pair where the second day is a Sunday hits the bad branch.
- **Expected:** streak goes 1 → 2 (two consecutive days).
- **Actual:** streak stays at **1** (`assert 1 == 2` fails). On Sundays the streak resets
  instead of incrementing. This is a specific-condition bug: it only shows up one day of
  the week, which is why users report it as intermittent.

### Issue #5 — "The last song in a playlist never shows up" (`playlist_service.py`)

- **How I reproduced it:** Run `tests/test_playlists.py`. `test_playlist_returns_all_songs`
  builds a playlist of 5 songs and asks for them back;
  `test_playlist_returns_songs_in_order` checks the returned titles.
- **Data condition that triggers it:** any non-empty playlist (the more songs, the more
  obvious). With the seed data, `GET /playlists/<id>/songs` returns one fewer song than
  were added.
- **Expected:** all 5 songs, ordered `Track 1 … Track 5`.
- **Actual:** only **4** songs returned (`Right contains one more item: 'Track 5'`); the
  final song by `position` is always dropped. Off-by-one at the end of the ordered list.

### Issue #4 — "Notified when a friend adds my song to a playlist, but not when they rate it" (`notification_service.py`)

- **How I reproduced it (live app, via Flask test client):**
  1. Seed the DB. Nova shared the song "Midnight Drive"; Darius is her friend.
  2. Check Nova's notifications: `GET /users/<nova_id>/notifications` → **count = 1**
     (the pre-seeded playlist-add notification).
  3. Darius rates Nova's song: `POST /songs/<song_id>/rate`
     with `{"user_id": <darius_id>, "score": 5}` → `201 Created`.
  4. Re-check Nova's notifications → **count is still 1**.
- **Expected:** the rating succeeds *and* Nova receives a new "song rated" notification
  (count → 2), mirroring how add-to-playlist already notifies the sharer.
- **Actual:** the rating is saved, but **no notification is created** — the count does not
  change. `rate_song()` never calls `create_notification()`, whereas `add_to_playlist()`
  in the same file does. This is a missing-side-effect bug, not a crash.

### Issue I attempted but could NOT reproduce — #3 "Same song shows up twice in search"

- **Attempt:** the search suite `tests/test_search.py` (which asserts multi-tag songs
  appear exactly once) **all passes**, and hitting the live endpoint
  `GET /songs/search?q=heights` for "Crown Heights Anthem" (3 tags) returns
  `{"count": 1, ...}` — a single result, no duplicates.
- **Why:** the code does `db.session.query(Song).outerjoin(song_tags)...` which fans out to
  one row per tag, but on SQLAlchemy 2.0 the ORM auto-deduplicates identical entities in a
  single-entity query result, so the duplicates never surface here. The latent bug exists in
  the query shape, but it does not manifest in this environment.
- **Decision:** per the milestone's guidance ("if you can't reproduce a bug after a genuine
  attempt, try a different one"), I dropped #3 and chose #4 instead.

**Checkpoint status:** I can deliberately trigger all three chosen bugs (#1 and #5 via
failing tests, #4 via the API), I know the exact inputs/conditions for each, and no code
has been changed yet.

---

## Milestone 3: Root Cause Analysis and Fixes

Each bug below was traced from symptom to root cause, fixed with the smallest possible
change, verified (including both sides of any boundary), and committed on its own.

### Issue #1 — "My listening streak keeps resetting"

**Affected file:** `services/streak_service.py`

**1. How I reproduced it:** Ran `tests/test_streaks.py::test_streak_increments_on_sunday`,
which listens on Saturday `2024-06-15` then Sunday `2024-06-16` and asserts the streak goes
1 → 2. The trigger condition is a consecutive-day listen where the *second* day is a Sunday.

**2. How I found the root cause:** Started from `routes/songs.py::listen()` →
`streak_service.record_listening_event()` → `update_listening_streak()`. Reading that
function top-down, the "listened yesterday" branch was
`elif days_since_last == 1 and today.weekday() != 6:`. The moment I saw the extra
`today.weekday() != 6` clause I was confident: `datetime.weekday()` returns 6 for Sunday, so
this clause makes the consecutive-day case *false* on Sundays, dropping execution into the
`else` branch that resets the streak. Nothing else in the function touches the streak value,
so this single comparison is the cause.

**3. The root cause:** For a genuine consecutive-day listen, the streak should always
increment. But the increment branch had an extra guard, `today.weekday() != 6`, that
excluded Sundays (`weekday()` numbers days Mon=0 … Sun=6). Whenever a user's second
consecutive day landed on a Sunday, the `days_since_last == 1` branch was skipped and the
`else` branch ran instead, resetting the streak to 1. There is no calendar reason a streak
should break on Sundays — the weekday check was simply wrong logic.

**4. The fix and side-effect check:** Removed the `and today.weekday() != 6` clause so the
branch is just `elif days_since_last == 1:`. Verified the whole `test_streaks.py` suite
passes (5/5), which covers both sides of the boundary: new user → 1, consecutive day → +1,
same day → no change, skipped day → reset to 1, and now Saturday→Sunday → +1. The change is
confined to one boolean condition and touches no other feature.

### Issue #5 — "The last song in a playlist never shows up"

**Affected file:** `services/playlist_service.py`

**1. How I reproduced it:** Ran `tests/test_playlists.py`. `test_playlist_returns_all_songs`
adds 5 songs and expects 5 back; `test_playlist_returns_songs_in_order` expects
`Track 1 … Track 5`. Both failed, returning only 4 songs (missing `Track 5`). Any non-empty
playlist triggers it.

**2. How I found the root cause:** Traced `GET /playlists/<id>/songs` →
`routes/playlists.py::get_songs()` → `playlist_service.get_playlist_songs()`. The query
itself was correct — it joins `playlist_entries`, filters by playlist, and orders by
`position` ascending. The problem was on the very last line: the return statement sliced the
result with `songs[:-1]`. Seeing the `[:-1]` slice made it obvious — it discards the last
element of an already-correct, position-ordered list.

**3. The root cause:** `get_playlist_songs()` built the correct ordered list of songs but
then returned `songs[:-1]` instead of `songs`. The `[:-1]` slice drops the final element, so
the song with the highest `position` (the last one added) was always omitted. It was a
straightforward off-by-one at the boundary of the list, not a query or ordering problem.

**4. The fix and side-effect check:** Changed the return to
`[song.to_dict() for song in songs]` (removed the `[:-1]`). All 3 playlist tests pass,
including `test_empty_playlist_returns_empty_list` — the important boundary check, since an
empty playlist must still return `[]` (it does). Ordering is unaffected because the `ORDER
BY position` in the query was never the problem.

### Issue #4 — "I got notified when a friend added my song to a playlist but not when they rated it"

**Affected file:** `services/notification_service.py`

**1. How I reproduced it:** Against the live app (Flask test client) with seed data: Nova
shared "Midnight Drive"; her friend Darius rated it. `GET /users/<nova>/notifications`
before = 1, `POST /songs/<song>/rate` with Darius + score 5 returned `201`, and after the
notification count was **still 1** — the rating saved but Nova got nothing.

**2. How I found the root cause:** Traced `POST /songs/<id>/rate` →
`routes/songs.py::rate()` → `notification_service.rate_song()`. Notably `rate_song` already
lives in `notification_service.py` alongside `add_to_playlist`. Reading both side by side,
`add_to_playlist()` ends with a `create_notification(...)` call guarded by
`if song.shared_by != added_by_user_id:`, while `rate_song()` commits the rating and returns
— it never calls `create_notification` at all. That contrast (same file, same "notify the
sharer" responsibility, one path does it and the other doesn't) is what made me certain this
was the exact cause rather than, say, a filtering issue in `get_notifications`.

**3. The root cause:** Notifications to a song's original sharer are created explicitly by
each interaction handler. The playlist-add handler does this; the rating handler simply
omitted the step. So rating a song had no side effect beyond saving the score — there was no
missing model or broken query, just an absent `create_notification` call in `rate_song()`.

**4. The fix and side-effect check:** After the rating commits, added a `create_notification`
call mirroring the playlist-add pattern — notifying `song.shared_by` with type
`song_rated`, but only `if song.shared_by != user_id` so users aren't notified for rating
their own songs. Verified against the live app:
- Friend rates Nova's song → notification count 1 → 2, body
  `"darius rated your song 'Midnight Drive' 5 out of 5."` ✅
- Nova rates her **own** song → count unchanged (no self-notification) ✅
- Full test suite still green (13/13).

> **Separate bug found (not one of my three, left as-is):** while side-effect-checking the
> add-to-playlist notification path, I hit a pre-existing `IntegrityError: NOT NULL
> constraint failed: playlist_entries.position`. `add_to_playlist()` adds the song via the
> ORM relationship (`playlist.songs.append(song)`), which cannot populate the non-null
> `position`/`added_by` columns on the `playlist_entries` association table. This is
> independent of my change — `git log` shows I never modified `add_to_playlist`, and it
> fails on the original code too. Noting it here for completeness; it is outside the three
> issues I chose to fix.

---

### Setup confirmation

- Virtual environment created, `requirements.txt` installed.
- `python seed_data.py` runs successfully (5 users, 13 songs, 3 playlists, 10 tags).
- App starts with `FLASK_APP=app:create_app flask run` and responds to requests
  (verified `GET /songs/search?q=heights` returns valid JSON). On macOS, port 5000 is often
  taken by AirPlay Receiver, so I run with `--port 5001` and use `http://127.0.0.1:5001`.
- Working branch `bugfix/mixtape` is checked out.
