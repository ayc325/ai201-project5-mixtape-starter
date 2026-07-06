# Mixtape

## AI Usage

I used Claude (via Claude Code) throughout this project, mainly in three ways: understanding the codebase, generating and running reproduction steps against the live server, and documenting the root cause analysis.

**Tracing data flow.** Before touching any bug, I asked it to explain how a request actually moves through the app — for example, the full path for "add a song to a playlist," from `POST /playlists/<playlist_id>/songs` through `routes/playlists.py::add_song`, into `notification_service.add_to_playlist`, and finally to where a `Notification` row gets created and later surfaced via `GET /users/<user_id>/notifications`. It laid this out as a step-by-step diagram, which is what let me see that `add_to_playlist` and `rate_song` are structurally the same kind of action ("a friend interacts with your shared song") but only one of them actually notifies the sharer — that comparison is what pointed me at Issue #4 as a real, missing-code bug rather than a stylistic quirk.

**Reproducing bugs against the running server.** Once I had a suspected bug, it was genuinely useful for turning "I think this is broken" into an actual before/after test: pulling real seeded IDs out of `instance/mixtape.db` with `sqlite3`, building the exact `curl` commands with those IDs already substituted in, and telling me what response to expect versus what the bug would actually produce. This was the fastest part of the workflow — for Issue #5, comparing the `playlist_entries` row count (7) against the API's reported count (6) via curl immediately confirmed the bug was real and isolated it to the service layer, not the seed data.

**Where it steered me wrong.** At one point, while writing repro steps for the Sunday streak bug, it told me to restart the server with `python app.py`. That's not how I'd been running the app — both the README and what I'd told it earlier specify `FLASK_APP=app:create_app flask run`. It should have picked up on that from context (I'd referenced the `flask run` command earlier in the session) and didn't, so I had to correct it and re-run the reproduction using the right command. Separately, more than once after I edited a service file, the AI's first instinct was to assume the fix "should" be reflected immediately — it didn't flag that the running Flask process wasn't using the reloader and needed a manual restart, so I ran into stale-code confusion (still seeing the buggy behavior after a fix) a couple of times before we figured out the server itself needed restarting.

**Where I verified independently.** I didn't take any "bug confirmed" or "fix confirmed" claim at face value — every repro and every fix verification in this document was checked against an actual `curl` response, an actual `sqlite3` query against the seeded database, or an actual `pytest` run, not just the AI's description of what should happen. I also double-checked the git history operations (rewording commit messages, force-pushing) by reviewing the diffs myself before agreeing to push.

## Codebase Map - Main files and their responsibilities

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

## Root cause analysis

### Issue #5 — `GET /playlists/<playlist_id>/songs` drops the last song

**How I reproduced it:** With the app running locally at `http://127.0.0.1:5000` (seeded via `python seed_data.py`), I picked the "Late Night Vibes" playlist (`id = b085dadd-d864-4e77-93bb-81637c3e0200`, `created_by = nova`) and compared two independent sources of truth for the same playlist:

1. Queried the actual link-table state directly against the seeded database:
   `sqlite3 instance/mixtape.db "SELECT COUNT(*) FROM playlist_entries WHERE playlist_id = 'b085dadd-d864-4e77-93bb-81637c3e0200';"` → **7** rows (i.e. 7 songs genuinely linked to this playlist).
2. Called the live endpoint for the same playlist:
   `curl http://127.0.0.1:5000/playlists/b085dadd-d864-4e77-93bb-81637c3e0200/songs` → response reported `"count": 6` and listed only 6 songs (ending at "Golden Hour"; the 7th linked song was missing entirely from the JSON).
3. The mismatch between the DB's row count (7) and the API's reported count (6), for a playlist seeded with no gaps or deletions, confirmed the bug was real and lived in the service/route layer rather than the data.

**How I found the root cause:** The mismatch pointed at whichever function turns the DB rows into the JSON response, so I went straight to `routes/playlists.py::get_songs`, which just calls `services/playlist_service.py::get_playlist_songs` and wraps the result — no filtering happens in the route. Reading `get_playlist_songs` top to bottom, the SQLAlchemy query itself (`.join(...).filter(...).order_by(asc(playlist_entries.c.position)).all()`) has no `LIMIT` or extra filter that would explain a missing row, so the 7 rows from the DB should already be present in `songs` at that point. The very next and only remaining line, `return [song.to_dict() for song in songs[:-1]]`, was the one place after the query where an item could be dropped — and `[:-1]` is a plain Python slice that removes exactly one item (the last), which matches the exact off-by-one (7 vs. 6) I observed. That match between the slice's known behavior and the observed discrepancy is what confirmed this line, not just the general function, as the cause.

**The root cause:** `get_playlist_songs` (`services/playlist_service.py:66`) builds the correct, correctly-ordered list of songs from the database, but then discards the last element before returning it: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice has nothing to do with pagination, deduplication, or any documented business rule — it unconditionally excludes whatever song happens to be last in position order, contradicting the function's own docstring ("returns all songs in the playlist").

**My fix and side-effect check:** Changed line 66 to `return [song.to_dict() for song in songs]`, removing the slice so all queried songs are serialized. To check for side effects, I ran the full test suite (`.venv/bin/python -m pytest tests/`): `tests/test_playlists.py` (3 tests covering playlist creation and song ordering) passed, and the only failure in the full suite (`test_streak_increments_on_sunday`) is the still-unfixed Issue #1, unrelated to this change. I also re-ran the live repro curl against `/playlists/b085dadd-d864-4e77-93bb-81637c3e0200/songs` after restarting the server and confirmed `"count"` now reads `7`, matching the DB row count exactly.

### Issue #1 — listening streak wrongly resets when "yesterday" lands on a Sunday

**How I reproduced it:** The real-world date at the time of testing was Sunday, 2026-07-05, so I didn't need to fake the clock — I just needed a seeded user whose `last_listened_at` was exactly one calendar day prior (Saturday, 2026-07-04).

1. Queried the seeded database directly to find a user meeting that data condition:
   `sqlite3 instance/mixtape.db "SELECT username, last_listened_at, listening_streak FROM user;"` → `nova` had `last_listened_at = 2026-07-04` (Saturday) and `listening_streak = 7`, satisfying `days_since_last == 1` relative to the real current date.
2. Checked nova's streak via the live endpoint before acting:
   `curl http://127.0.0.1:5000/users/557e9251-9ddb-43b7-a5b4-ea0e4f3a3ce9/streak` → `7`.
3. Logged a new listen for nova today (Sunday) via the live endpoint:
   `curl -X POST http://127.0.0.1:5000/songs/0c36f239-0a61-46c2-8148-204b3f19b8f1/listen -H "Content-Type: application/json" -d '{"user_id": "557e9251-9ddb-43b7-a5b4-ea0e4f3a3ce9"}'`
4. Checked the streak again: `curl http://127.0.0.1:5000/users/557e9251-9ddb-43b7-a5b4-ea0e4f3a3ce9/streak`.
   **Expected** (per the documented rule "If the user listened yesterday: streak increments by 1"): `8`.
   **Actual:** `1` — the streak was reset instead of incremented.

**How I found the root cause:** Since the increment happens purely server-side with no unusual input, I went to `services/streak_service.py::update_listening_streak`, the only function that touches `listening_streak`. Walking through it with nova's actual values (`days_since_last == 1`, since Saturday → Sunday is one calendar day), the code should hit the `elif days_since_last == 1: ...` branch and increment — except the branch is actually `elif days_since_last == 1 and today.weekday() != 6:`. Plugging in the reproduction's real date (Sunday, `weekday() == 6`) makes that added condition `False`, so control falls through to `else: user.listening_streak = 1`. That's the exact moment I was confident: the extra `and today.weekday() != 6` clause is the only thing in the function that isn't described anywhere in the docstring's three stated rules, and it's the only thing that changes behavior specifically on a Sunday — which is exactly the day my repro failed on. I also confirmed this wasn't a pre-existing intentional rule by checking `tests/test_streaks.py`, which has a dedicated (currently failing) test `test_streak_increments_on_sunday` asserting the increment should happen — the test file itself documents the intended behavior and currently fails for exactly this reason.

**The root cause:** `update_listening_streak` (`services/streak_service.py:73`) is supposed to increment the streak whenever the gap since the last listen is exactly one calendar day. Instead of checking only `days_since_last == 1`, the condition also requires `today.weekday() != 6` (today is not a Sunday). When someone listens on consecutive days and the second day happens to be a Sunday, that added clause evaluates to `False`, so the `elif` is skipped and execution falls into the `else` branch, which resets `listening_streak` to `1` instead of incrementing it.

**My fix and side-effect check:** Changed line 73 from `elif days_since_last == 1 and today.weekday() != 6:` to `elif days_since_last == 1:`, removing the day-of-week clause so the increment applies uniformly regardless of what day it lands on. Verified two ways:

1. Ran `.venv/bin/python -m pytest tests/test_streaks.py -v` — all 5 tests pass, including the previously-failing `test_streak_increments_on_sunday`, while the other rules (no-prior-listen → streak starts at 1, same-day → no change, gap of more than one day → reset to 1) remain unaffected.
2. Re-ran the live repro against the reseeded DB: nova (`user_id = de9177ca-190d-43ed-b993-d9642027b4b3`) had `last_listened_at` one calendar day before the server's current time, streak `7`. `curl http://127.0.0.1:5000/users/de9177ca-190d-43ed-b993-d9642027b4b3/streak` → `7`; `curl -X POST http://127.0.0.1:5000/songs/d5603720-3d42-4582-a577-b4b05420e3b4/listen -H "Content-Type: application/json" -d '{"user_id": "de9177ca-190d-43ed-b993-d9642027b4b3"}'`; then the streak endpoint again → `8`. (Note: by the time of this re-test the server clock had rolled from Sunday into Monday, so this specific run exercises the general "listened yesterday" path rather than the Sunday edge case — but since the fix removes the day-of-week check entirely, the increment is now correct on every day, which is exactly what `test_streak_increments_on_sunday` confirms deterministically.)

### Issue #4 — rating a song never notifies the sharer

**How I reproduced it:** Using two friended seeded users — `nova` (the sharer) and `darius` (the rater) — and a song nova shared ("Still Waters," `id = 316ae5be-ca9f-4abe-bfb2-bfc93cb999a7`):

1. Checked nova's notifications before the rating, via the live endpoint:
   `curl http://127.0.0.1:5000/users/557e9251-9ddb-43b7-a5b4-ea0e4f3a3ce9/notifications` → baseline `count`.
2. Had darius rate nova's song:
   `curl -X POST http://127.0.0.1:5000/songs/316ae5be-ca9f-4abe-bfb2-bfc93cb999a7/rate -H "Content-Type: application/json" -d '{"user_id": "ac8d68bc-9713-4c41-87fc-d2c820ccb0b7", "score": 5}'` → returned `201` with the new `Rating`, confirming the rating itself was saved.
3. Checked nova's notifications again:
   `curl http://127.0.0.1:5000/users/557e9251-9ddb-43b7-a5b4-ea0e4f3a3ce9/notifications`.
   **Expected:** `count` increased by 1, with a new `song_rated`-style notification about darius's rating.
   **Actual:** `count` was unchanged — no notification was created for nova.

**How I found the root cause:** I opened `services/notification_service.py::rate_song`, the function invoked by `POST /songs/<song_id>/rate`, and read it end to end: it validates the score, looks up the song and rater, then either updates an existing `Rating` or creates a new one, commits, and returns — there is no call to `create_notification` anywhere in the function. To be sure this was a gap and not, say, a notification path guarded behind some condition I'd missed, I compared it against `add_to_playlist` in the same file, which handles a structurally identical scenario (a friend interacting with a song someone else shared): it performs its primary side effect (appending to the playlist), then explicitly checks `song.shared_by != added_by_user_id` and calls `create_notification(...)`. `rate_song` has the equivalent primary side effect (saving the `Rating`) but no equivalent notify step at all — not a broken condition, an entirely missing call. The module docstring's mention of `'song_rated'` as an example `notification_type` (a type that appears nowhere in the actual code) confirmed this was a planned-but-unimplemented feature rather than a deliberate omission.

**The root cause:** `rate_song` (`services/notification_service.py:73-110`) never calls `create_notification`. Every other "friend interacts with your shared song" action in this module (`add_to_playlist`) notifies the original sharer after its side effect completes; `rate_song` performs its side effect (creating/updating the `Rating`) but has no corresponding call to notify `song.shared_by`, so the sharer is never told their song was rated.

**My fix and side-effect check:** After the `db.session.commit()` in `rate_song`, added:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by, notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5."
    )
```

mirroring `add_to_playlist`'s guard so a user rating their own shared song doesn't notify themselves.

Verified against the (reseeded) live server: nova (`user_id = de9177ca-190d-43ed-b993-d9642027b4b3`) started at `count: 1` (the seeded `song_added_to_playlist` notification). After darius (`user_id = 43c43078-70f5-4ff1-b7e5-ad97b33b521a`) rated nova's "Midnight Drive" (`song_id = d5603720-3d42-4582-a577-b4b05420e3b4`) via `POST /songs/d5603720-3d42-4582-a577-b4b05420e3b4/rate`, `GET /users/de9177ca-190d-43ed-b993-d9642027b4b3/notifications` returned `count: 2`, with a new entry: `{"type": "song_rated", "body": "darius rated your song 'Midnight Drive' 3/5."}`. (First attempt showed no change because the running server process hadn't picked up the code change — restarting it resolved that and the notification then appeared as expected.) I also confirmed the existing `add_to_playlist` notification path still fires correctly (it's the older entry still present in the same response), so the fix didn't disturb that unrelated pattern.

## Patterns observed

- **Routes are thin, services own all logic.** Every route handler's job is: parse the request body/query params, call exactly one service function, and translate its return value or raised `ValueError` into a JSON response (400/404 on `ValueError`, otherwise 200/201). No route touches `db.session` or a model directly except the simplest `GET /users/<id>` lookup in `routes/users.py`.
- **`ValueError` is the app-wide "not found / invalid input" signal.** Services raise it for missing users/songs/playlists/notifications or invalid input (e.g. an out-of-range rating score); routes uniformly catch it and turn it into an HTTP error response. There's no custom exception hierarchy.
- **Association tables carry metadata when needed.** `friendships` and `song_tags` are plain many-to-many join tables, but `playlist_entries` adds `position`, `added_by`, and `added_at` — reflecting that playlist membership needs explicit ordering and provenance, unlike friendships or tags.
- **`to_dict()` on every model is the serialization boundary.** Services return plain dicts (via `to_dict()`) wherever a route needs to jsonify a collection; single-object routes sometimes return the ORM object directly (e.g. `rate` returns `rating.to_dict()`, `listen` returns `event.to_dict()`) and Flask serializes it via `jsonify`.
- **Cross-service imports happen locally, not at module top-level.** `notification_service.add_to_playlist` imports `Playlist` and `playlist_service.get_playlist_songs` inside the function body rather than at the top of the file, avoiding a circular import between `notification_service` and `playlist_service`.
- **No dedicated migration tooling.** `db.create_all()` runs on every app startup in `app.py`, and `seed_data.py` is the mechanism for getting sample data in — there's no Alembic/migration directory.
