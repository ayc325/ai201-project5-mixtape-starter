# Mixtape — Codebase Map

## Main files and their responsibilities

**`app.py`** — Flask application factory. Creates the Flask app, configures the SQLAlchemy database URI (SQLite by default, overridable via `DATABASE_URL`), initializes the shared `db` object, registers the four route blueprints (`songs`, `playlists`, `users`, `feed`) under their URL prefixes, and calls `db.create_all()` on startup. This is the single entry point that wires everything else together.

**`models.py`** — SQLAlchemy models for every entity in the app:
- `User` — has a `listening_streak` counter and `last_listened_at` timestamp (used by the streak feature), plus relationships to shared songs, ratings, listening events, notifications, and playlists. `friends` is a self-referential many-to-many via the `friendships` association table.
- `Song` — has `shared_by` (FK to the user who shared it) and relates to `Rating`, `ListeningEvent`, and `Tag` (many-to-many via `song_tags`).
- `Tag` — a simple label, joined to songs via `song_tags`.
- `ListeningEvent` — one row per (user, song, timestamp) play; this is what streak and feed logic query against.
- `Rating` — one row per (user, song) pair, enforced by a unique constraint, storing a 1–5 `score`.
- `Playlist` — has `created_by` and `is_collaborative`; its songs relationship goes through `playlist_entries`, which (unlike the other association tables) carries extra columns: `position` (explicit ordering, not insertion order), `added_by`, and `added_at`.
- `Notification` — belongs to a `user_id` (the recipient), with a `notification_type`, free-text `body`, and a `read` flag.

Every model has a `to_dict()` used to serialize it directly into JSON responses — there's no separate serialization layer.

**`routes/`** — one blueprint per resource area, each doing only request parsing and response formatting:
- `songs.py` — search, get song detail, rate a song, record a listen.
- `playlists.py` — create a playlist, get playlist metadata, list a playlist's songs, add a song to a playlist.
- `users.py` — get a user, get their streak, list/mark-read their notifications.
- `feed.py` — friends' current listening activity, and a general activity feed.

**`services/`** — all business logic, one module per concern:
- `streak_service.py` — records listening events and recalculates `listening_streak` based on the gap since `last_listened_at`.
- `feed_service.py` — builds the "friends listening now" (recent, deduplicated per friend) and "activity feed" (recent N events, no dedup) views.
- `search_service.py` — case-insensitive song search by title/artist, joined against tags; single-song lookup.
- `notification_service.py` — low-level notification creation, plus the two actions that currently trigger notifications (`add_to_playlist`) or don't (`rate_song`), and notification retrieval/read-marking.
- `playlist_service.py` — playlist creation and retrieval, including ordered song lookup via the `playlist_entries` join table.

**`seed_data.py`** — populates the database with sample users, friendships, songs, playlists, and events for local testing.

**`tests/`** — pytest suites for streaks, search, and playlists, exercising the service layer directly.

## Data flow: adding a song to a playlist (triggers a notification)

1. Client sends `POST /playlists/<playlist_id>/songs` with `song_id` and `added_by` in the JSON body.
2. `routes/playlists.py::add_song` parses the request, validates the two required fields are present, and calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.
3. Inside `add_to_playlist` (`services/notification_service.py`):
   - Looks up the `Song`, the adding `User`, and the `Playlist` by ID, raising `ValueError` (surfaced as a 400 by the route) if any is missing.
   - If the song isn't already in `playlist.songs`, appends it (via the `playlist_entries` association) and commits — this is the actual "add to playlist" side effect.
   - Compares `song.shared_by` to `added_by_user_id`. If the person adding the song isn't the one who originally shared it, calls `create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", body=...)`.
4. `create_notification` builds a `Notification` row and commits it — this is the only place `Notification` rows are written from application logic.
5. The route returns `{"message": "Song added to playlist"}` with a 201; the notification itself is only visible later, when the recipient calls `GET /users/<user_id>/notifications`, which goes through `routes/users.py::notifications` → `notification_service.get_notifications`.

By contrast, `rate_song` (also in `notification_service.py`, invoked from `POST /songs/<song_id>/rate`) only writes/updates a `Rating` row and never calls `create_notification` — rating a song and adding it to a playlist are structurally similar "friend interacts with your shared song" actions, but only one of them currently produces a notification.

```text
Client
  │  POST /playlists/<playlist_id>/songs
  │  { song_id, added_by }
  ▼
routes/playlists.py :: add_song
  │  parses + validates request body
  ▼
services/notification_service.py :: add_to_playlist(playlist_id, song_id, added_by)
  │
  ├─▶ look up Song, User, Playlist by ID ──▶ raise ValueError if missing ──▶ route returns 400
  │
  ├─▶ if song not in playlist.songs:
  │       append via playlist_entries association
  │       db.session.commit()            ◀── actual "add to playlist" side effect
  │
  └─▶ compare song.shared_by vs added_by_user_id
          │
          │ different user (not the original sharer)
          ▼
      create_notification(user_id=song.shared_by,
                           notification_type="song_added_to_playlist",
                           body=...)
          │
          ▼
      Notification row created + committed
          │
          ▼
route returns 201 { "message": "Song added to playlist" }

  ⋯ later, asynchronously ⋯

Recipient (song.shared_by)
  │  GET /users/<user_id>/notifications
  ▼
routes/users.py :: notifications
  ▼
notification_service.py :: get_notifications
  ▼
Notification row surfaced to recipient
```

## Patterns observed

- **Routes are thin, services own all logic.** Every route handler's job is: parse the request body/query params, call exactly one service function, and translate its return value or raised `ValueError` into a JSON response (400/404 on `ValueError`, otherwise 200/201). No route touches `db.session` or a model directly except the simplest `GET /users/<id>` lookup in `routes/users.py`.
- **`ValueError` is the app-wide "not found / invalid input" signal.** Services raise it for missing users/songs/playlists/notifications or invalid input (e.g. an out-of-range rating score); routes uniformly catch it and turn it into an HTTP error response. There's no custom exception hierarchy.
- **Association tables carry metadata when needed.** `friendships` and `song_tags` are plain many-to-many join tables, but `playlist_entries` adds `position`, `added_by`, and `added_at` — reflecting that playlist membership needs explicit ordering and provenance, unlike friendships or tags.
- **`to_dict()` on every model is the serialization boundary.** Services return plain dicts (via `to_dict()`) wherever a route needs to jsonify a collection; single-object routes sometimes return the ORM object directly (e.g. `rate` returns `rating.to_dict()`, `listen` returns `event.to_dict()`) and Flask serializes it via `jsonify`.
- **Cross-service imports happen locally, not at module top-level.** `notification_service.add_to_playlist` imports `Playlist` and `playlist_service.get_playlist_songs` inside the function body rather than at the top of the file, avoiding a circular import between `notification_service` and `playlist_service`.
- **No dedicated migration tooling.** `db.create_all()` runs on every app startup in `app.py`, and `seed_data.py` is the mechanism for getting sample data in — there's no Alembic/migration directory.
