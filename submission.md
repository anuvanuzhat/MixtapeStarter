# Codebase Map
seed_data.py: Populates the database with realistic test data.
This script creates:
- 5 users with established friendships
- 25 songs with varying tag counts (0, 1, and 3+ tags)
- 3 playlists with 5-10 songs each
- Listening events spanning the past 2 weeks (including recent ones)
- Existing streaks for some users
- Existing playlist-add notifications

Data Flow: starts by creating the app context and resetting the database, then creates users, friendships, tags, and songs with different tag counts. It adds recent and older listening events, sets some user streak metadata, builds playlists with entries, adds a sample notification, and finally commits everything so the database is populated with realistic test data.

///

models.py: defines 5 SQLAlchemy models: User, Song, Playlist, PlaylistSong, and Notification. The PlaylistSong table is a join table that adds an order column — songs in a playlist have an explicit position, not just insertion order.

Data flow — user rates a song: POST /songs/<id>/rate in routes/songs.py calls notification_service.notify_song_rated(). That function creates a Notification record for the song's original sharer. There's no separate rating model — the rating is stored directly on the Song.

Pattern I noticed: every route delegates immediately to a service function. The routes do input parsing and response formatting; all business logic lives in services/."

///

app.py: is responsible for creating and configuring the Flask application and initializing the SQLAlchemy database connection.

Data flow:
create_app() makes a new Flask app
loads config from environment or defaults
initializes db with that app
imports and registers route blueprints for songs, playlists, users, and feed
creates database tables inside the app context
returns the configured app
When run directly, it calls create_app() and launches the Flask development server.

# Reproduce Code
1. Called update_listening_streak() directly with a User whose last_listened_at was set to Saturday, July 4, 2026 and listening_streak = 12. Passed in a now value of Sunday, July 5, 2026 (one calendar day later) to simulate the user listening again the next morning. Expected listening_streak to become 13 (consecutive day), but it reset to 1. Root cause: the increment condition days_since_last == 1 and today.weekday() != 6 explicitly excludes Sundays from incrementing, so any consecutive-day listen that lands on a Sunday falls through to the reset branch instead.

2. Created a ListeningEvent for a friend with listened_at set to 11:00 PM the previous day, then called get_friends_listening_now() for the current user at roughly 9:00 AM the next morning (~10 hours later). Expected the friend to be excluded, since their listen happened on a different calendar day. Actual result: the friend still appeared in the results. Root cause: get_friends_listening_now() filters on ListeningEvent.listened_at >= cutoff, where cutoff = datetime.now(timezone.utc) - timedelta(hours=24) — a rolling 24-hour window rather than a calendar-day boundary. Any listen from "yesterday evening" stays inside that 24-hour window until the exact same clock time rolls around the next day, which is why the friend's stale listen kept showing up as "listening now."

4. Called rate_song(user_id, song_id, score) for a song shared by another user, then called get_notifications(sharer_id) to check for a new notification. Expected a notification of type "song_rated" (mirroring the existing "song_added_to_playlist" notification), but no notification was created — get_notifications() returned the same list as before the rating. By contrast, calling add_to_playlist() under the same conditions does create a notification. Root cause: rate_song() saves the Rating record and commits, but never calls create_notification() — unlike add_to_playlist(), which explicitly notifies song.shared_by after adding the song. The notification-creation step for ratings was simply never written.

