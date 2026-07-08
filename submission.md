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
routes/playlists.py: POST /playlists/<playlist_id>/songs. ROute pulls song_id and added_by from JSON data, calls add_to_playlist(playlist_id, song_id, added_by) in notification_service. That service (add_to_playlist speficially) adds the song in the playlist and checks if the person who shared the song is the same person who added it to the playlist. If not, it notifies the user who shared a song that the song has been added to someone's playlist. Route returns 201

## Patterns
I noticed:
1) The routes do input parsin and response formatting, but all real logic lives in services
2) Errors flow via ValueError (services raise, routes translate to 400/404). Routes call service -> service reports something wrong -> route catches that and raises 400/404
3) Serialization via each model's to_dict(). Each model can be represented as dictionary and has a function to be converted to one. Very good design choice and makes future developers' work easier