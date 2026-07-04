# **Project 5 - Mixtape**

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

# Milestone 2 – Bug Reproduction

The following notes describe how I reproduced (or attempted to reproduce) each of the five reported issues before making any code changes.

---

## Issue #1 – My listening streak keeps resetting

### How I reproduced it

I examined the listening streak feature by looking at a seeded user's current streak and listening history. I also identified the endpoint responsible for recording listening events (`POST /songs/<song_id>/listen`) and confirmed that this endpoint updates the user's streak. Because this bug only occurs under a specific Sunday boundary condition described in the project brief, I noted that additional investigation would be needed during the debugging phase to recreate the exact scenario before implementing a fix.

---

## Issue #2 – Friends Listening Now shows people from yesterday

### How I reproduced it

Using the Flask shell, I modified one of the seeded `ListeningEvent` records so that its `listened_at` timestamp was approximately 23 hours old. After committing the change, I requested the `/feed/<user_id>/listening-now` endpoint in the browser.

The response still included users whose listening activity occurred many hours earlier instead of only users who were currently listening, confirming the reported issue.

---

## Issue #3 – The same song keeps showing up twice in search

### How I reproduced it

I tested the search endpoint (`/songs/search?q=...`) using several different search terms against the seeded data. Although I did not immediately observe duplicate songs in every search result, I confirmed that the issue is conditional, as described in the project brief. I recorded this behavior for further investigation during the debugging phase.

---

## Issue #4 – I got notified when a friend added my song to a playlist but not when they rated it

### How I reproduced it

I first verified that playlist notifications worked by viewing the notification list for a user after another user added one of their songs to a playlist. The notification appeared correctly.

Next, I rated another user's song using the rating endpoint. The rating was successfully stored, but no notification appeared for the original song owner, matching the behavior described in the issue.

---

## Issue #5 – The last song in a playlist never shows up

### How I reproduced it

I requested the songs for an existing playlist using the `/playlists/<playlist_id>/songs` endpoint and compared the response with the playlist stored in the database using the Flask shell.

The API response returned only six songs, while `len(playlist.songs)` in the database returned seven songs. The final song in the playlist was missing from the endpoint response, confirming the reported issue.