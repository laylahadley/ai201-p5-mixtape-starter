# Mixtape Bug Hunt Submission

## Codebase Map

### Main files and their roles
- `app.py`: <what it does>
    - This module is the application entry point and wiring for Mixtape. It creates the shared database object, builds and configures the Flask app, connects the route modules, and makes sure the tables exist. It contains no routes or business logic of its own.
        - Module-level object --> db = SQLAlchemy() creates the Flask-SQLAlchemy instance, deliberately unbound to any app. models.py imports it (from app import db) to define its models, and create_app binds it to a real app later. Creating it at module level is what lets the models and the factory share it without circular setup problems.

        - functions:
            - create_app(config=None) is the application factory. It builds a fresh, fully configured Flask app each time it’s called. In order, it:
                1. Creates the Flask app with Flask(__name__).
                2. Sets default config from environment variables, with fallbacks:
                    - SQLALCHEMY_DATABASE_URI comes from DATABASE_URL, defaulting to a local SQLite file, mixtape.db
                    - SECRET_KEY comes from the SECRET_KEY env var, defaulting to "dev-secret-key".
                    - SQLALCHEMY_TRACK_MODIFICATIONS is turned off to avoid unneeded overhead.
                3. Applies overrides. If a config dict is passed in, it’s merged over the defaults. This is how tests can point at an in-memory database, for example.
                4. Initializes the database with db.init_app(app).
                5. Registers blueprints, each under its own URL prefix:
                    - songs_bp at /songs
                    - playlists_bp at /playlists
                    - users_bp at /users
                    - feed_bp at /feed 
                    - The imports happen inside the function, which avoids circular imports, since the route modules presumably import db and the models from this module.
                6. Creates tables by calling db.create_all() inside an app context. This makes any missing tables from the models, but it doesn’t alter existing ones.
                7. Returns the app.
            - The __main__ block
                - When the file is run directly (python app.py), it calls create_app() and starts Flask’s development server with debug=True.

- `models.py`: <models, association tables, key columns>
        - This module is the data layer for Mixtape, a social music-sharing app. It defines every SQLAlchemy model (database table) the app uses, plus the association tables that link them. It holds no business logic, only the schema, relationships, and serialization helpers.
        - helper function: generate_uuid() returns a random UUID string. It’s the default primary-key generator for every model, so IDs are 36-character strings rather than auto-incrementing integers.
        - association tables: 
            - 'friendships' links two users (user_id, friend_id). Despite the “symmetric” comment, the table itself doesn’t enforce symmetry, so the app code has to insert both directions.
            - song_tags links songs to tags.
            - playlist_entries links playlists to songs, and also records the song’s position in the playlist, who added it (added_by), and when (added_at).
        - Models: 
            - User is an app member with a unique username and email, plus listening-streak tracking (listening_streak, last_listened_at)
                - It has relationships to the songs they’ve shared, their ratings, listening events, notifications, and playlists they created.
                - friends is a dynamic relationship through friendships, so it returns a query you can filter rather than a loaded list.
                - to_dict() returns id, username, streak, and last-listened time (email is deliberately left out).
            - Tag is a simple label (unique name) that can be attached to songs.
            - Song is a shared track with title, artist, optional album and genre, who shared it (shared_by), when, and an optional share_note
                - It relates to its ratings, listening events, and tags (tags are eagerly loaded via a subquery).
                - to_dict() serializes the song, including tag names as a list of strings.
            - ListeningEvent records that a user listened to a song at a given time. This is the raw data behind streaks and listening history. to_dict() serializes it.
            - Rating is a user’s 1–5 score for a song. A unique constraint (unique_user_song_rating) limits each user to one rating per song. The 1–5 range is only a comment here, not enforced at the database level. to_dict() serializes it.
            - Playlist is a named collection of songs created by a user. is_collaborative defaults to True, meaning others can add songs. Its songs relationship goes through playlist_entries. to_dict() returns metadata only (not the songs).
            - Notification is a message for a user, with a notification_type, a body, a read flag, and a timestamp. to_dict() serializes it, exposing notification_type as type.

- `routes/<file>.py`: <what it does>
    - Together, these files are the HTTP layer of Mixtape. Each one is a Flask blueprint that maps URLs to handler functions. The handlers are thin: they parse the request, call a function in services/, and turn the result or a ValueError into a JSON response. The business logic lives elsewhere.
        - routes/feed.py (mounted at /feed)
            - Read-only social feeds for a user. Both routes are GETs that call the feed service and return {"feed": [...], "count": n}. An unknown user (the service raises ValueError) gives a 404.
            - listening_now(user_id) at /feed/<user_id>/listening-now returns what the user’s friends are listening to right now.
            - activity(user_id) at /feed/<user_id>/activity returns the user’s friends’ recent activity, such as shares, ratings, and listens.
        - routes/playlists.py (mounted at /playlists)
            - create(), POST /playlists/: reads name, created_by, and optional is_collaborative (default True) from the JSON body. It returns 400 if name or created_by is missing, otherwise calls create_playlist and returns the playlist’s to_dict() with 201.
            - get_detail(playlist_id), GET /playlists/<id>: returns the playlist from get_playlist, or 404 if it isn’t found.
            - get_songs(playlist_id), GET /playlists/<id>/songs: returns {"songs": [...], "count": n}, or 404.
            - add_song(playlist_id), POST /playlists/<id>/songs: reads song_id and added_by from the body (400 if either is missing) and calls add_to_playlist. It returns 201 on success.
        - routes/users.py (mounted at /users)
            - get_user(user_id), GET /users/<id>: returns the user’s to_dict(), or 404. It’s the only handler in these files that queries the database directly (db.session.get(User, ...)) instead of going through a service.
            - streak(user_id), GET /users/<id>/streak: returns {"user_id", "streak"} from the streak service, or 404.
            - notifications(user_id), GET /users/<id>/notifications: lists the user’s notifications. The optional ?unread_only=true query param filters to unread ones. Returns the list plus a count, or 404.
            - read_notification(notification_id), POST /users/notifications/<id>/read: marks one notification as read, or returns 404 if it doesn’t exist.
        
- `services/<file>.py`: <what it does>
    - services/feed_service.py
        - Builds the social feeds from friends’ listening events. Both functions return dicts with friend, song, and listened_at, newest first, and return [] if the user has no friends.
        - get_friends_listening_now(user_id) finds friends with a listening event in the last 24 hours (RECENT_THRESHOLD). It keeps only each friend’s most recent event, so each friend appears once.
        - get_activity_feed(user_id, limit=20) returns the latest limit listening events from all friends, with no time cutoff. The same friend can appear several times.
    - services/playlist_service.py
        - creates and reads playlists
        - create_playlist(name, created_by_user_id, is_collaborative=True) checks the user exists, then creates and commits the playlist.
        - get_playlist_songs(playlist_id) returns the playlist’s songs as dicts, ordered by playlist_entries.position.
        - get_playlist(playlist_id) returns the playlist’s metadata dict, without songs.
        - get_user_playlists(user_id) returns all playlists a user created. No route calls it.
    - services/search_service.py
        - song look up
        - search_songs(query) does a case-insensitive substring match (ILIKE) on title or artist and returns song dicts, tags included.
        - get_song(song_id) returns one song dict, or raises ValueError.
    - services/streak_service.py
        - Listening streaks, defined as consecutive UTC calendar days.
        - record_listening_event(user_id, song_id) creates a ListeningEvent, updates the user’s streak, and commits.
        - update_listening_streak(user, now) applies the rules: first listen sets 1, same day changes nothing, the next day increments, and a longer gap resets to 1. It also handles naive datetimes by assuming UTC.
        - get_streak(user_id) returns the stored listening_streak.
    - services/notification_service.py
        - holds three kinds of logic
            - create_notification(user_id, notification_type, body) creates and commits a Notification.
            - add_to_playlist(playlist_id, song_id, added_by_user_id) validates the song, user, and playlist, adds the song if it isn’t already there, and notifies the song’s original sharer. There’s no notification if they added it themselves.
            - rate_song(user_id, song_id, score) validates the score (1–5), song, and user, then creates or updates the user’s rating.
            - get_notifications(user_id, unread_only=False) returns a user’s notifications as dicts, newest first.
            - mark_as_read(notification_id) sets read = True, or raises ValueError if the notification doesn’t exist.

### Data flow: <feature name, e.g., user rates a song>
<Route> in <routes file> calls <function> in <services file>. That function <step 1>, <step 2>, <step 3>.

### Patterns I noticed

- Architecture and conventions:
    - Layered structure. Dependencies run one way: routes/ calls services/, which uses models.py and db. Routes stay thin and services hold the logic. Only get_user breaks this by querying the DB directly.
    - App factory with blueprints. A single module-level db is shared by everything, and create_app binds it later. Imports inside functions (blueprints in create_app, Playlist inside add_to_playlist) work around circular imports.
    - ValueError as the domain error. Services raise it for “not found” or invalid input, and routes catch it and map it to a status code. It’s simple and consistent, but it can’t distinguish a missing record from a validation failure (hence the 400/404 inconsistency).
    - to_dict() serialization. Models serialize themselves and services return dicts, not ORM objects. Routes mostly just jsonify the result.
    - Other conventions. String UUID primary keys, timezone-aware UTC defaults, type hints on all service functions, and Google-style docstrings with Args/Returns throughout.

- Habits that may create risk:
    - Every service function commits on its own. There are no transaction boundaries, so multi-step operations like add_to_playlist (commit, then commit again inside create_notification) can half-complete.
    - N+1 lookups. Both feeds fetch the friend and song one event at a time, where a join or eager load would do.
    - Validation lives in different places. The route checks for missing fields, the service checks existence, and the 1–5 rating range is enforced in rate_song but not in the database. Nothing enforces it consistently.
    - No auth. User identity comes from the URL or request body everywhere.
    - Mixed SQLAlchemy styles. db.session.get and select() (2.0 style) sit next to db.session.query (legacy style), sometimes in the same file.

- Drift between intent and code (this was the most notable pattern)
    - get_playlist_songs says it returns all songs but slices off the last one.
    - update_listening_streak documents simple consecutive-day rules but has a hidden Sunday exception.
    - search_songs suggests tag-aware results but doesn’t search tags, and its join creates duplicates.
    - get_friends_listening_now is really “last 24 hours”.
    - models.py calls the friendship table symmetric, but nothing enforces it.
    - rate_song’s docs mention a 'song_rated' notification that never gets sent.

- organization drift
    - add_to_playlist and rate_song live in notification_service.py, probably because they started as “do X, then notify”.
    - There are leftovers: unused imports, get_user_playlists with no route, and routes/playlists.py pasted twice.
    - Pieces are missing, notably routes/songs.py, which app.py imports.

## Root Cause Analyses
(added in later milestones)