# AI Usage

# Codebase Map

## App.py
app.py - Flask application factory and database setup. create_app() configures SQLite, initializes db, registers the four blueprints under URL prefixes (/songs, /playlists, /users, /feed), and calls db.create_all()

## Models.py
models.py - SQLAlchemy models for all database entities. It defines 7 SQLAlchemy models: User, Tag, Song, ListeningEvent, Rating, Playlist, and Notification. There is 3 tables: friendships, song_tags, and playlist_entries. 

Models:
1) User - model of the user entity. Stores user id, username, email, listening_streak, last_listened_at, created_at. Defines relationship with shared_songs, ratings, listening_events, notifications, playlists, friends. Has to_dict() serialization.
2) Tag - model of tag. Stores tag id and name
3) Song - model of the song entity. Stores song id, title, artist, album, genre, shared_by (which user), shared_at (what time), share_note. Defines relationships with ratings, listening_events, and tags. Has to_dict() serialization.
4) ListeningEvent - model of the listening event. Stores id, user_id, song_id, listened_at. Has to_dict() serialization.
5) Rating - model of a rating. Stores rating id, user_id, song_id, score, rated_at (time). Has to_dict() serialization
6) Playlist - model of a playlist. Stores id, name, created_by, created_at, is_collaborative (true or false - can someone collaborate on it or not). Defines relationship with song. Has to_dict() serialization.
7) Notifications - model of a notification. Has id, user_id, notification_type, body, created_at (time), read (true or false). Has to_dict() serialization

Tables:
1) frienships is self-referential and symmetric (user to user).
2) song_tags is many to many
3) playlist_entries isn't a plain join table: it carries position, added_by, added_at. So playlist order is an explicit stored integer, not insertion order. 

## Routes folder
4 files: each file parses the request, calls one service function, formats the JSON response, and maps ValueError → a 400/404. Each file is just a route, the logic doesn't actually live here. Every route delegates immediately to a service function. The routes do input parsing and response formatting; all business logic lives in services/.

## Services
5 files: all the main logic that is getting called from the routes folders. All the work gets done here

1) search_service - song search + fetching a single song
2) streak_service - recording listening events + computing the streak
3) feed_service - building "friends listening now" + the activity feed
4) notification_service - creating notifications (on rating, on playlist-add), plus retrieving/marking them
5) playlist_service - creating playlists and retrieving their songs

## Data Flow
routes/playlists.py: POST /playlists/<playlist_id>/songs. Route pulls song_id and added_by from JSON data, calls add_to_playlist(playlist_id, song_id, added_by) in notification_service. That service (add_to_playlist speficially) adds the song in the playlist and checks if the person who shared the song is the same person who added it to the playlist. If not, it notifies the user who shared a song that the song has been added to someone's playlist. Route returns {"message": "Song added to playlist"}, 201

## Patterns
I noticed:
1) The routes do input parsin and response formatting, but all real logic lives in services
2) Errors flow via ValueError (services raise, routes translate to 400/404). Routes call service -> service reports something wrong -> route catches that and raises 400/404
3) Serialization via each model's to_dict(). Each model can be represented as dictionary and has a function to be converted to one. Very good design choice and makes future developers' work easier

# Root Cause Analysis

## Issue #2 - Friends Listening Now shows people from yesterday

**How I reproduced it:** Reseeded the DB, then recreated nova's reported scenario: I
gave darius a single listening event dated ~20 hours ago (yesterday evening) and
removed his other events, so his most recent listen was "last night" and he hadn't
listened since. I then fetched nova's feed (`GET /feed/<nova_id>/listening-now`).
darius still appeared as "listening now" (last listen 20h ago) alongside simone and
kenji who had listened minutes earlier. This matches nova's report that friends whose
last listen was yesterday evening still show up the next morning.

**How I found the root cause:** _(TODO)_

**The root cause:** _(TODO)_

**My fix and side-effect check:** _(TODO)_

## Issue #4 - Missing notification when a friend rates a song

**How I reproduced it:** Reseeded the DB, confirmed simone (who shared "Crown Heights
Anthem") had 0 notifications. Had kenji rate that song 5 stars via
`POST /songs/<song_id>/rate`. The rating was saved successfully (a Rating row with an
id was returned). I then checked simone's notifications
(`GET /users/<simone_id>/notifications`): still 0, with zero `song_rated`
notifications. For contrast, the working `song_added_to_playlist` notification type
does get created. So a rating is persisted but no notification is ever generated for
the song's sharer.

**How I found the root cause:** I started from the working case. In
`notification_service.py`, `add_to_playlist()` ends by calling `create_notification()`
to notify the song's sharer. I then read `rate_song()` in the same file and compared
the two line-by-line. `rate_song()` saves the `Rating` and commits, but its function
body ends there - there is no `create_notification()` call anywhere in it. That
missing step is what made me confident this was the exact cause, not just a suspicious
area: the working path has the call and the broken path simply doesn't.

**The root cause:** `rate_song()` in `notification_service.py` persists a `Rating`
row but never calls `create_notification()`. Because no `Notification` record is ever
created, the song's sharer gets nothing - even though the rating itself is saved
correctly (which is why the score shows on the song but no notification appears).
The parallel action, `add_to_playlist()`, does create a notification, so ratings were
the only interaction missing that step.

**My fix and side-effect check:** I mirrored the pattern from `add_to_playlist()`.
After the rating is saved and committed, I added a guard - `if song.shared_by !=
user_id` - so a user rating their own shared song doesn't notify themselves, and when
the rater is someone else I call `create_notification()` with type `"song_rated"` and
a body of `f"{rater.username} rated your song '{song.title}' {score} stars."`.
Side-effect check: I reran my reproduction and confirmed the sharer now receives
exactly one `song_rated` notification with readable text (username and score, not
raw UUIDs). I also confirmed the existing `song_added_to_playlist` notification still
works, that the `Rating` is still saved (including the update-existing-rating branch),
and that the self-rating case correctly produces no notification.

## Issue #5 - Last song in a playlist does not show up

**How I reproduced it:** Reseeded the DB and inspected the "Friday Energy" playlist.
Direct query of the `playlist_entries` table showed **7** entries (positions 1-7,
position 7 = "Harlem Renaissance"). But `GET /playlists/<id>/songs` returned only
**6** songs, and the endpoint's own `count` field was also 6 - so the shortfall is
invisible from the API alone. The dropped song is always the one at the highest
position (the most recently ordered / newest), matching darius's report that "the
missing one is always whatever was added most recently."

**How I found the root cause:** _(TODO)_

**The root cause:** _(TODO)_

**My fix and side-effect check:** _(TODO)_