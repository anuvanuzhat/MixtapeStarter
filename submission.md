# AI Usage
Bug tracing and root cause analysis: For each bug I chose (streak reset, stale feed entries, missing rating notifications), I pasted the relevant service file and asked Claude to help identify where the bug likely lived and why. For the streak bug, I initially didn't understand why the report was Sunday-specific — Claude walked me through how datetime.weekday() returns 6 for Sunday and pointed out that the increment condition had an unrelated day-of-week check bolted onto it. For the feed bug, Claude connected a specific detail in the bug report ("stale entries disappear at the same time the next day") to the idea that the cutoff was a rolling 24-hour window rather than a calendar-day boundary — that framing helped me see it wasn't a date-comparison bug so much as a wrong definition of "recent." For the notification bug, Claude suggested comparing the working function (add_to_playlist) against the broken one (rate_song) side by side, which made the missing create_notification() call obvious once I looked.

Reproduction strategy: I didn't initially know how to "reproduce" a bug that depends on a specific date (Sunday) without waiting for an actual Sunday. Claude suggested calling the relevant function directly with a manually constructed datetime instead of going through the real app clock — that's the approach I used for both the streak and feed bugs.

Where I verified or corrected things myself: After I wrote my own fixes, I asked Claude to check them rather than write them for me outright — I wanted to confirm the logic (e.g., removing the weekday check, changing the cutoff calculation, adding the notification call with the right guard condition) actually matched the root cause I'd identified, not just take its word for it. I also caught and worked through my own git mistakes independently in real time — when my first attempt at splitting commits accidentally bundled all four changed files into one commit, I ran git status myself to see the actual state before acting on Claude's next suggestion, rather than blindly following instructions. When a git rebase -i got stuck mid-process, I reported the exact terminal output back and we worked through aborting it together — Claude's first suggested approach (interactive rebase) turned out to be more failure-prone in practice than the simpler reset --soft + re-commit approach we switched to, so I ended up using the simpler method instead of the first one offered.

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


# Root Analysis Format

Issue #1 — My listening streak keeps resetting
Reproduced by: Set a user's last_listened_at to Saturday with listening_streak = 12, then called update_listening_streak() with now = the following Sunday. Expected streak 13, got 1.
Root cause found in: services/streak_service.py → update_listening_streak(). The increment branch was days_since_last == 1 and today.weekday() != 6. Confidence came from realizing "consecutive day" logic has no valid reason to also check weekday — that extra clause was the whole bug.
Root cause: weekday() returns 6 for Sunday. The increment condition required today ≠ Sunday, so any consecutive-day listen landing on a Sunday failed the check and fell through to the reset branch, wiping the streak to 1.
Fix: Removed and today.weekday() != 6, leaving increment on days_since_last == 1 only.
Checked: Same-day no-op branch and multi-day-skip reset branch still work correctly; re-ran repro and confirmed streak now goes 12 → 13 across a Sunday.

Issue #2 — Friends Listening Now shows people from yesterday
Reproduced by: Created a friend's ListeningEvent at 11 PM the prior day, then called get_friends_listening_now() ~10 hours later (9 AM next day). Friend still appeared — expected them excluded.
Root cause found in: services/feed_service.py → get_friends_listening_now(). Traced the cutoff variable back to RECENT_THRESHOLD = timedelta(hours=24). Confidence came from the report detail that stale entries vanish "at the same time the next day" — a signature of a rolling time window, not a calendar-day filter.
Root cause: Cutoff was now - 24 hours, a rolling window, not tied to calendar day. An 11 PM listen checked at 9 AM (~10 hrs later) is still within 24 hours, so it incorrectly counts as "recent."
Fix: Changed cutoff to start of current day (UTC): now.replace(hour=0, minute=0, second=0, microsecond=0).
Checked: get_activity_feed() doesn't use this cutoff, unaffected. Same-day recent listens still appear correctly. Noted UTC-vs-local-timezone as a separate follow-up, not fixed here.

Issue #4 — No notification on song rating
Reproduced by: Called rate_song() on a song shared by another user, then checked get_notifications(sharer_id). Rating saved successfully, but no notification appeared.
Root cause found in: services/notification_service.py. Compared rate_song() against add_to_playlist() side by side. Confidence came from seeing add_to_playlist() ends with a create_notification() call that rate_song() simply never has — not a logic error, a missing step.
Root cause: rate_song() saves the Rating and commits but never calls create_notification(), unlike add_to_playlist() which notifies song.shared_by. The notification step for ratings was never implemented.
Fix: Added create_notification() call at the end of rate_song(), guarded by song.shared_by != user_id (same self-notification guard as playlist adds), with type "song_rated".
Checked: Self-rating case correctly produces no notification; add_to_playlist() still works unchanged; re-rating an existing score still triggers a notification.