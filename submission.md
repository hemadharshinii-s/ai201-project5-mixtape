# **Project 5 - Mixtape**

For the playlist bug, I traced the endpoint into get_playlist_songs() myself before asking ChatGPT to review whether the list slicing operation (songs[:-1]) explained the reported behavior. After the explanation, I verified that Python's slicing semantics matched the observed behavior by rerunning the endpoint after making the change.

## **Codebase Map (Milestone 1)**

###  **Main Files and Their Roles**

### `README.md`

The README provides the overall architecture of the application, setup instructions, and the list of the five known bugs that need to be investigated. It also demonstrates example call chains (such as rating a song or viewing a playlist), which helps explain how requests flow through the application.

### `app.py`

`app.py` is the Flask application factory. It creates the Flask application, configures SQLAlchemy, initializes the database connection, registers every blueprint from the `routes/` directory, and creates the database tables. This file contains almost no application logic—it is responsible only for configuring and starting the application.

### `models.py`

`models.py` defines every SQLAlchemy model and the relationships between them. The application's data model consists of:

* **User** – stores account information, listening streak, friendships, playlists, listening history, ratings, and notifications.
* **Song** – stores shared songs, including title, artist, album, genre, tags, and the user who originally shared the song.
* **ListeningEvent** – records every time a user listens to a song. These records are used for listening streaks and activity feeds.
* **Rating** – stores a user's rating for a song (1–5 stars). Each user can only rate a song once due to a unique database constraint.
* **Playlist** – stores collaborative playlists created by users.
* **Notification** – stores notifications sent to users when other users interact with their songs.
* **Tag** – stores song tags used during search.

Several association tables connect these models:

* **friendships** implements the many-to-many relationship between users.
* **song_tags** connects songs with their tags.
* **playlist_entries** connects playlists and songs while also storing additional information such as song position, who added the song, and when it was added. This allows playlists to preserve a specific song ordering instead of relying on insertion order alone.


### **Routes Layer**

The `routes/` directory exposes the application's HTTP API. The route files perform request validation, extract request data, call service-layer functions, and return JSON responses. Very little business logic exists inside the routes themselves.

### `routes/songs.py`

Handles song-related endpoints, including:

* Searching for songs
* Viewing song details
* Rating a song
* Recording listening events

The routes delegate work to `search_service`, `notification_service`, and `streak_service`.

### `routes/playlists.py`

Handles playlist operations, including:

* Creating playlists
* Viewing playlist information
* Retrieving songs within a playlist
* Adding songs to playlists

Most playlist operations are delegated to `playlist_service`, while adding songs also calls `notification_service` to notify the original song owner.

### `routes/users.py`

Handles user-related endpoints, including:

* Retrieving user information
* Viewing listening streaks
* Viewing notifications
* Marking notifications as read

This route layer primarily delegates to `streak_service` and `notification_service`.

### `routes/feed.py`

Handles social feed functionality, including:

* Friends Listening Now
* General activity feed

Both endpoints delegate directly to `feed_service`.

### **Services Layer**

The `services/` directory contains the application's business logic. Every major feature is implemented inside a service module, while the routes simply call these functions.

### `streak_service.py`

Responsible for recording listening events and maintaining each user's listening streak.

Main responsibilities:

* Create new `ListeningEvent` records.
* Update a user's listening streak.
* Return the user's current streak.

### `feed_service.py`

Builds the application's social feed.

Main responsibilities:

* Retrieve friends who have listened recently.
* Generate the user's activity feed.
* Sort listening events by recency.
* Remove duplicate friend entries in the "Listening Now" view.

### `search_service.py`

Implements song searching.

Main responsibilities:

* Search songs by title or artist.
* Retrieve individual song information.
* Join song records with their associated tags.

### `notification_service.py`

Handles all notification-related logic.

Main responsibilities:

* Create notifications.
* Notify song owners when another user adds their song to a playlist.
* Record song ratings.
* Retrieve notifications.
* Mark notifications as read.

This service acts as the central location for notification creation.

### `playlist_service.py`

Implements playlist functionality.

Main responsibilities:

* Create playlists.
* Retrieve playlist metadata.
* Retrieve playlist songs in playlist order.
* Retrieve playlists created by a user.

## **Data Flow Example – User Rates a Song (Milestone 1)**

One complete request path through the application is when a user rates a song.

1. The client sends a **POST** request to:

   `/songs/<song_id>/rate`

2. The request is handled by the `rate()` function in `routes/songs.py`.

3. The route validates that both `user_id` and `score` were provided.

4. The route calls:

   `notification_service.rate_song(user_id, song_id, score)`

5. Inside `notification_service.py`:

   * The score is validated.
   * The song is retrieved from the database.
   * The user submitting the rating is retrieved.
   * The service checks whether the user has already rated the song.
   * If a rating already exists, it updates the existing record.
   * Otherwise, it creates a new `Rating` object.
   * The database transaction is committed.

6. The updated or newly created `Rating` object is returned to the route.

7. The route converts the object into JSON and returns it to the client.

This request demonstrates the application's layered architecture: the route performs request validation and delegates the business logic entirely to the service layer, while the service interacts with the database models.

## **Data Flow Example – User Adds a Song to a Playlist (Milestone 1)**

Another complete feature begins when a user adds a song to a playlist.

1. The client sends a **POST** request to:

   `/playlists/<playlist_id>/songs`

2. The request is handled by `add_song()` in `routes/playlists.py`.

3. The route validates that both `song_id` and `added_by` are present.

4. The route calls:

   `notification_service.add_to_playlist()`

5. The notification service:

   * Retrieves the playlist.
   * Retrieves the song.
   * Retrieves the user adding the song.
   * Adds the song to the playlist if it is not already present.
   * Commits the playlist update.
   * If someone other than the original owner added the song, creates a new notification for the original sharer.

6. The route returns a success response to the client.

This feature demonstrates that one service can both update application data and trigger additional actions (creating notifications) as part of a single request.

## **Architecture Patterns I Observed (Milestone 1)**

Several consistent architectural patterns appear throughout the codebase.

* The application follows a layered architecture consisting of **Routes → Services → Models → Database**.
* Routes are intentionally lightweight. Their primary responsibilities are parsing requests, validating input, handling exceptions, and formatting JSON responses.
* Nearly all business logic is centralized inside the `services/` directory.
* Database operations are performed through SQLAlchemy models rather than raw SQL.
* Each service module owns one major feature of the application, making responsibilities well separated.
* Models focus only on representing database entities and relationships. They contain helper methods such as `to_dict()` but very little application logic.
* Most route functions immediately delegate to exactly one service function, making the execution flow easy to trace from an HTTP endpoint into the business logic.
* Several features build on one another. For example, playlist operations can trigger notification creation, and listening events both record user activity and update listening streaks. This indicates that services are designed to coordinate related pieces of application behavior while still keeping the route layer simple.

## **Root Cause Analysis Entries (Milestones 2 & 3)**

### **Issue #1 — My listening streak keeps resetting**

#### **How I Reproduced It**

I simulated listening activity across consecutive days, including transitions over a Sunday boundary. When a user listened on consecutive days but one of the days was Sunday, the streak unexpectedly reset to 1 instead of incrementing.

#### **How I Found The Root Cause**
I traced the `record_listening_event()` function in `services/streak_service.py`, focusing on how `update_listening_streak()` determines whether to increment or reset the streak. I examined the conditional logic around `days_since_last` and noticed an additional weekday-based condition affecting streak updates.

#### **The Root Cause**

The streak update logic incorrectly included a weekday check: `today.weekday() != 6`. This was intended to handle a perceived Sunday boundary case, but it incorrectly prevented valid streak increments when a user listened on consecutive days that included Sunday. As a result, even when `days_since_last == 1`, the streak would reset instead of incrementing whenever the condition failed on Sunday, breaking normal consecutive-day streak behavior.

#### **My Fix and Side-Effect Check**

I removed the unnecessary weekday condition and made streak updates depend only on whether `days_since_last == 1`. This ensures streaks increment strictly based on consecutive calendar days, regardless of weekday. I verified that:
- consecutive-day listening increments the streak correctly,
- gaps greater than 1 day reset the streak,
- same-day listening does not change the streak,
- Sunday transitions behave consistently with other weekdays.

### AI usage
I used AI assistance to interpret how the weekday-based condition could interact with date arithmetic in streak tracking. I verified the final logic by reasoning through date differences across a Sunday boundary scenario.

### **Issue #2 – Friends Listening Now shows people from yesterday**

#### **How I Reproduced It**

I modified a seeded `ListeningEvent` so that a friend's `listened_at` timestamp was approximately 23 hours in the past. After triggering the `/feed/<user_id>/listening-now` endpoint, the friend still appeared in the "Listening Now" feed, even though their activity occurred the previous day.

#### **How I Found The Root Cause**

I traced the endpoint from `routes/feed.py` into `get_friends_listening_now()` in `services/feed_service.py`. The query correctly filtered `ListeningEvent` records using a cutoff time based on `datetime.now(timezone.utc) - RECENT_THRESHOLD`. The ordering and deduplication logic were correct. The key observation was that `RECENT_THRESHOLD` was set to 24 hours, which did not align with the expected meaning of "Listening Now."

#### **The Root Cause**

The function defined "recent listening activity" as any event within the last 24 hours. This caused users who had listened to music many hours earlier—even from the previous day—to still qualify as "currently listening." While the filtering logic was correct, the time window used was too broad for a feature intended to represent live or near-real-time activity.

#### **My Fix and Side-Effect Check**

I reduced the `RECENT_THRESHOLD` from 24 hours to 30 minutes so that only genuinely recent listening activity is included in the "Listening Now" feed. This preserves the existing filtering, ordering, and deduplication logic while making the definition of "now" consistent with the feature’s intent.

After the fix, I verified that users with listening events older than 30 minutes no longer appear in the feed, while recent activity still appears correctly. I also confirmed that the general activity feed remains unchanged and still includes older listening events, ensuring no unintended side effects.

### **Issue #3 — The same song keeps showing up twice in search**

#### **How I Reproduced It**

I tested the `/songs/search` endpoint using a broad query and observed that some songs appeared multiple times in the results when those songs had multiple associated tags in the seed data. The duplication only occurred for songs with more than one tag, while songs with zero or one tag appeared once as expected.

#### **How I Found The Root Cause**

I traced the `search_songs()` function in `services/search_service.py` and focused on the SQLAlchemy query structure. The key observation was the `outerjoin` between `Song` and the `song_tags` association table. I compared query behavior for songs with different numbers of tags and identified that songs with multiple tags produced multiple SQL rows before being converted into ORM objects.

#### **The Root Cause**

The join between `Song` and `song_tags` creates a one-to-many expansion at the SQL row level. For songs with multiple tags, this results in multiple identical `Song` ORM objects being returned in the query result set (one per tag association). Because no deduplication was applied at the song level, songs with multiple tags appeared multiple times in search results even though they represent a single logical entity.

#### **My Fix and Side-Effect Check**

I added `.distinct(Song.id)` to the SQLAlchemy query to ensure each song appears only once in the result set regardless of how many tag associations exist. I verified that tag data is still correctly included via the ORM relationship / `to_dict()` method, and confirmed that songs with zero, one, and multiple tags all appear exactly once in search results after the fix.

### **Issue #4 – I got notified when a friend added my song to a playlist but not when they rated it**

#### **How I Reproduced It**

I first confirmed that playlist notifications worked by having one user add another user's shared song to a playlist and then viewing the recipient's notifications. A notification was created as expected. Next, I rated another user's song using the rating endpoint (`POST /songs/<song_id>/rate`). The rating was successfully saved to the database, but no notification appeared for the original song owner, reproducing the reported issue.

#### **How I Found The Root Cause**

I traced both features through `services/notification_service.py` and compared the two code paths. I first examined `add_to_playlist()`, which correctly creates a notification after updating the playlist. I then followed the execution of `rate_song()`. The rating logic validated the input, updated or created the `Rating` record, and committed the transaction, but then immediately returned the rating. The moment I became confident I had found the root cause was when I compared the two functions line by line and saw that `rate_song()` never called `create_notification()`.

#### **The Root Cause**

The notification system itself was functioning correctly, but the rating workflow never used it. After saving a rating, `rate_song()` committed the database transaction and returned the `Rating` object without creating a notification for the user who originally shared the song. As a result, ratings were recorded successfully, but no notification was ever generated because that step was completely missing from the execution path.

#### **My Fix and Side-Effect Check**

I added a call to `create_notification()` after the rating is successfully committed. The notification is sent to the original song owner when another user rates their song, following the same architectural pattern already used by the playlist notification workflow. I also kept the existing behavior of not notifying users about actions they perform on their own songs.

After making the change, I verified that ratings were still saved correctly, that rating another user's song now created a notification, that rating my own song did not create a notification, and that playlist notifications continued to work as before.

### **Issue #5 – The last song in a playlist never shows up**

#### **How I Reproduced It**

I requested the playlist songs endpoint (`GET /playlists/<playlist_id>/songs`) for one of the seeded playlists and compared the API response against the playlist contents stored in the database using the Flask shell. The database contained seven songs (`len(playlist.songs) == 7`), but the endpoint only returned six songs. The final song in the playlist was consistently missing, confirming the reported bug.

#### **How I Found The Root Cause**

I started by tracing the endpoint from the route into the service layer. The playlist songs endpoint delegates directly to `get_playlist_songs()` in `services/playlist_service.py`, so I focused my investigation there. I followed the function from the database query through to the returned response. The SQLAlchemy query correctly retrieved every song in the playlist and ordered them by position. The point where I became confident I had found the root cause was the final return statement, which sliced the list before converting the songs to dictionaries.

#### **The Root Cause**

The function returned `songs[:-1]` instead of `songs`. In Python, the slice `[:-1]` returns every element except the last one. Although the database query correctly retrieved every song, the final list comprehension intentionally omitted the last element before returning the response. As a result, every playlist returned one fewer song than actually existed, regardless of playlist size.

#### **My Fix and Side-Effect Check**

I removed the unnecessary slice so the function now returns every song retrieved by the query (`songs`) instead of `songs[:-1]`. This allows the endpoint to return the complete playlist without altering the query or playlist ordering logic.

After making the change, I requested the same playlist again and confirmed that all seven songs were returned. I also verified that the songs remained in the correct order, confirming that only the missing-song bug was fixed and that the existing ordering behavior was unaffected.