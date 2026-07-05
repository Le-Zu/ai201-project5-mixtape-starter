# Project 5: Mixtape Bug Hunt Submission

## AI Usage Section

During this project, I leveraged **Antigravity** (Gemini 3.5 Flash) to assist with codebase navigation, analysis, and locating test failures:
- **Codebase Navigation & Summarization**: We used the agent to explore the directory structure, read core files like [models.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/models.py), and trace the request/response flow from the [routes/](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/routes/) blueprint controllers to the [services/](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/) business logic.
- **Root Cause & Test Analysis**: Executed `pytest` through terminal integration to run the suite, identifying 3 specific test failures out-of-the-box. I used code-reading prompts to inspect the logic and determine the exact conditions causing the bugs.
- **Verification**: All findings and code logic were manually reviewed and verified by checking standard datetime rules, SQL join mechanics, and list slicing parameters in Python.

---

## Codebase Map

### Main Files and Roles
- **[app.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/app.py)**: Configures the Flask application factory, initializes SQLAlchemy (`db`), registers routes from blueprint files, and sets up the SQLite database context.
- **[models.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/models.py)**: Defines the SQLAlchemy database schemas and relationships for:
  - `User`: Handles accounts, listening streaks, friends (symmetric self-referential association table `friendships`), and relationships to playlists, ratings, and events.
  - `Song`: Tracks shared music metadata and holds many-to-many associations to `Tag` via `song_tags`.
  - `ListeningEvent`: Log entries of when a user listens to a song.
  - `Rating`: Stores song reviews (1-5 score) and unique constraints to ensure one rating per song per user.
  - `Playlist`: Captures user-generated collections of songs linked via `playlist_entries` which maintains absolute song order/positions.
  - `Notification`: Stores notifications sent to users when friends rate or add their shared songs.
- **[routes/](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/routes/)**: HTTP request handlers (blueprints) that parse JSON payloads, URL parameters, and map responses to JSON:
  - [routes/songs.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/routes/songs.py): Endpoints for searching, rating, and registering song listen events.
  - [routes/playlists.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/routes/playlists.py): Endpoints for creating playlists and adding/getting songs.
  - [routes/users.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/routes/users.py): Endpoints for fetching profiles, listening streaks, and retrieving or marking notifications as read.
  - [routes/feed.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/routes/feed.py): Endpoints for activity feeds and active friends.
- **[services/](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/)**: Encapsulates all transactional core business logic:
  - [services/streak_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/streak_service.py): Evaluates calendar dates to manage and increment/reset user listening streaks.
  - [services/feed_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/feed_service.py): Assembles feed streams for active friends and general activity list.
  - [services/search_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/search_service.py): Performs SQL queries to locate songs by title or artist.
  - [services/notification_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/notification_service.py): Handles user ratings and inserts notifications for actions.
  - [services/playlist_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/playlist_service.py): Manages playlist structures and retrieval.
- **[seed_data.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/seed_data.py)**: Seeds mock database rows to support manual testing and local exploration.
- **[tests/](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/tests/)**: Automated pytest test suite.

### Data Flow Example: Adding a Song to a Playlist & Notifying
1. The client issues a `POST /playlists/<playlist_id>/songs` request with `song_id` and `added_by` in the request body.
2. [routes/playlists.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/routes/playlists.py) intercepts the request, verifies arguments, and forwards execution by calling `add_to_playlist(playlist_id, song_id, added_by)` in [services/notification_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/notification_service.py).
3. `add_to_playlist` retrieves the `Song`, the adding `User`, and the `Playlist` models using SQLAlchemy.
4. The song is added to the playlist if it does not already exist: `playlist.songs.append(song)`.
5. A check verifies if the song is being added by someone other than the original sharer: `if song.shared_by != added_by_user_id`.
6. If the condition is met, `create_notification` is called to insert a new `Notification` row in the database, with `notification_type="song_added_to_playlist"`, linking it to the song's original owner (`song.shared_by`).
7. Finally, the database transaction is committed, and a success response is returned to the user.

### Architectural Patterns
- **Separation of Concerns (Route/Service Layering)**: Route files under `routes/` serve strictly as API gateways (deserializing JSON, validating input bounds, returning standard HTTP codes, serializing models to dicts). Service modules under `services/` are pure business workflows isolated from HTTP/Flask context.
- **Rich Models & Utility Serialization**: Database relationships and foreign key models are concentrated in [models.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/models.py). Each model implements a `.to_dict()` helper, enabling consistent dictionary output formatting across routes.
- **Inter-service Dependencies**: Certain service actions cross boundaries to trigger side-effects (e.g. adding a song in `notification_service` retrieves playlist songs and updates playlists, coupling operations with corresponding event logs).

---

## Root Cause Analysis (RCA) Templates

*(Note: These templates are ready to be completed as fixes are implemented in subsequent milestones)*

### Issue #1: My listening streak keeps resetting
1. **Issue number and title**: Issue #1: My listening streak keeps resetting
2. **How you reproduced it**: Run the test suite: `.venv/bin/pytest tests/test_streaks.py` -> `test_streak_increments_on_sunday` fails.
3. **How you found the root cause**: Checked [services/streak_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/streak_service.py) in `update_listening_streak()`. Saw the day comparison logic on line 73.
4. **The root cause**: Python's `datetime.weekday()` returns `6` for Sunday. The condition `elif days_since_last == 1 and today.weekday() != 6` explicitly prevents streaks from incrementing on Sundays, resetting them instead.


### Issue #2: Friends Listening Now shows people from yesterday
1. **Issue number and title**: Issue #2: Friends Listening Now shows people from yesterday
2. **How you reproduced it**: *[To be completed after reproduction]*
3. **How you found the root cause**: Checked [services/feed_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/feed_service.py).
4. **The root cause**: The `RECENT_THRESHOLD` used for filtering current listen events is set to `timedelta(hours=24)`. This window is far too wide for a real-time "Listening Now" feature, showing activity from the previous day.


### Issue #3: The same song keeps showing up twice in search
1. **Issue number and title**: Issue #3: The same song keeps showing up twice in search
2. **How you reproduced it**: Run `.venv/bin/pytest tests/test_search.py` -> `test_search_no_duplicates_multi_tag_song` fails.
3. **How you found the root cause**: Inspected [services/search_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/search_service.py) in `search_songs()`.
4. **The root cause**: The database query performs an outer join with `song_tags`: `db.session.query(Song).outerjoin(song_tags, Song.id == song_tags.c.song_id)`. Because a song can have multiple tags, this join generates multiple rows for the same song. Since distinct is not enforced, duplicate objects are returned.


### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it
1. **Issue number and title**: Issue #4: No notification when friend rates a song
2. **How you reproduced it**: *[To be completed after reproduction]*
3. **How you found the root cause**: Checked [services/notification_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/notification_service.py) in `rate_song()`.
4. **The root cause**: The `rate_song` function does not trigger notification creation. Unlike `add_to_playlist`, there is no call to `create_notification()` when a rating transaction completes.


### Issue #5: The last song in a playlist never shows up
1. **Issue number and title**: Issue #5: The last song in a playlist never shows up
2. **How you reproduced it**: Run `.venv/bin/pytest tests/test_playlists.py` -> `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` fail.
3. **How you found the root cause**: Checked [services/playlist_service.py](file:///home/lezu/Projects/codepath/ai201/ai201-project5-mixtape-starter/services/playlist_service.py) in `get_playlist_songs()`.
4. **The root cause**: The function returns `songs[:-1]`, which is a slice that intentionally excludes the last element of the songs list.
