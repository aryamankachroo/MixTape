# Mixtape — Submission

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

### Setup confirmation

- Virtual environment created, `requirements.txt` installed.
- `python seed_data.py` runs successfully (5 users, 13 songs, 3 playlists, 10 tags).
- App starts with `FLASK_APP=app:create_app flask run` and responds to requests
  (verified `GET /songs/search?q=heights` returns valid JSON). On macOS, port 5000 is often
  taken by AirPlay Receiver, so I run with `--port 5001` and use `http://127.0.0.1:5001`.
- Working branch `bugfix/mixtape` is checked out.
