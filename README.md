# Spotify-Personal-Assistant
Application to maintain a Spotify database, generate listening reports and build playlists. Harvard CS50 final project.


## Table of contents
* [Description, programming language and libraries](#Description,-programming-language-and-libraries)
* [Database creation and maintenance](#database-creation-and-maintenance)
* [Reports](#Reports)
* [Playlist creation](#Playlist-creation)
* [Thanks](#Thanks)


## Description, programming language and libraries
My final CS50 project is an application suite developed in Phyton that assists the Spotify power user.
It gives users access to the extensive Spotify for developers' API, taking the Spotify experience to a whole new level.

It uses the following Python libraries:

* calendar
* CSV
* datetime
* json
* matplotlib
* PYsimpleGUI
* spotipy.oauth2
* spotipy
* sqlite3
* Tkinter
* os
* PIL
* urllib.request

Further analysis of data through visualizations in

* Tableau

## Database creation and maintenance

### Schema

The application relational database has the following schema:

    CREATE TABLE tracks (id VARCHAR(62), song TEXT, album TEXT, release VARCHAR(32), popularity INTEGER,
            main_artist_id	TEXT, main_genre TEXT, acousticness REAL, danceability REAL, energy REAL,
            instrumentalness REAL, liveness	REAL, loudness REAL, speechiness REAL, valence REAL,
            tempo REAL, duration INTEGER, PRIMARY KEY(id))
    CREATE TABLE history(id VARCHAR(62), timestamp VARCHAR(24) PRIMARY KEY)
    CREATE TABLE saved_tracks(id VARCHAR(62) PRIMARY KEY)
    CREATE TABLE tracks_artists(id VARCHAR(62), artist_id VARCHAR(62), PRIMARY KEY(id,artist_id))
    CREATE TABLE artists(artist_id VARCHAR(62) PRIMARY KEY, artist TEXT, image_url TEXT, popularity INT)
    CREATE TABLE genres(artist_id VARCHAR(62), genre TEXT, PRIMARY KEY(artist_id, genre))

### Populating the database for the first time

Populating the database for the first time requires the following steps:

1. The user should request her/his Spotify user data (On Spotify account under privacy settings). Spotify will email a zip file containing -among other files- 12 months of listening history in JSON format.

2. Running saved_songs_upload.py. The python program accesses the users liked tracks in the Spotify API and populates the tables as follows:
    1. Saved tracks.
    1. Tracks. Iterating id from saved tracks the program populates information on each track.
    1. Artists. Iterating artist id from each track the program populates information on each artist.
    1. Genres. Iterating artist id the program populates the genres for each artist

3. Running search_song_info.py Creating listening history from JSON data provided by Spotify via email as requested in step 1.
    1. The JSON file contains data on each listening event in the last 12 months as follows:
        ```
            {
                "endTime" : "2020-05-30 18:40",
                "artistName" : "Bruce Springsteen",
                  "trackName" : "Thunder Road",
                "msPlayed" : 288720
            },
        ```
    2.  Unfortunately the JSON file does not contain the track id, so the program needs to find it. First the program checks if a track with the same name and artists exists in the saved tracks table. If it is already in saved tracks, then it uses the existing track id. Else, conducts same query on the Spotify API and then searches the API for the rest of the information necessary to populate the tables. The syntax in the spotipy library to find a track ID is:

                q = ("artist:%s track:%s" % (track_artist, track_name))
                search_results = sp.search(q, limit=50, market='US')

    The query returns a JSON object with information on tracks that match the query. The program collects the track id of the first item with an identical match to the query.

### Maintainig the database
The app contains two programs to maintain the database.
1. keeping_track.py. Keeping track of listening history. The program periodically uploads from the Spotify API the list of last tracks played and adds them to the history. Spotify API provides the most recent 50 tracks ids in a listeners' history. If the id is not present in the track table, the program searches Spotify API for all the required fields to populate the database.
2. saved_songs_upload.py. Updating saved tracks table.

## Reports
Program: statistics_gui.py. The app provides a Graphic User Interface (GUI) developed in PYsimpleGUI to report on the user listening activity. The user selects to conduct the query on either a specific year or the entire listening history.
The program conducts the following queries:
1. Simple queries on total listening activity displayed as sentences in the GUI. (Total tracks and total distinct tracks)
2. Ten most listened songs in the requested period. Conducts the following query:

    ```
        command = f""" SELECT strftime ('%Y', timestamp) as year, song, artist,
        count(song) as n FROM history
        INNER JOIN tracks on tracks.id = history.id
        INNER JOIN artists ON tracks.main_artist_id = artists.artist_id
        where year = {year} group by song order by n desc limit 10 """

    ```
    The results are displayed in the GUI first column.
3. Most listened genres. Aggregates spotify defined genres conducting queries on the total duration of tracks listened. The following example for rock total time spent listening to rock tracks.
    ```
        command = f"""SELECT strftime ('%Y', timestamp) as year, sum(duration)
        FROM history INNER JOIN tracks on tracks.id = history.id
        WHERE (main_genre LIKE '%rock%' OR main_genre = 'beatlesque' OR main_genre = 'alternative dance')
        AND (year = {year})""

    ```
    The results for each genre aggregate are divided by the total listening time of the period and a pie chart is generated using the matplotlib library. The graph is saved in png format and then displayed in the second column of the GUI.
4. Most listened artists. Finds the top listened artists per period using the following query:
    ```
            command = f"""SELECT strftime ('%Y', timestamp) as year, artist, image_url, sum(duration) FROM history
                    INNER JOIN tracks on tracks.id = history.id
                    INNER JOIN tracks_artists on tracks.id = tracks_artists.id
                    INNER JOIN artists on artists.artist_id = tracks_artists.artist_id
                    WHERE year = {year}
                    GROUP BY artist
                    ORDER BY sum(duration)
                    DESC limit 5  """
    ```
    The program fetches the url jpg file for each of the top artists, converts them to PNG and displays them in the bottom frame of the GUI.
5. The user has the option to request a CSV file containing detailed information on listening history for further data analytics. For this project Tableau was used.
6. The following image shows the GUI output.

## Playlist creation
Program: playlist_gui.py The cornerstone of the application suite is the ability to create dynamic playlists. Several playlist creators are available online, however I have not been able to find any that gives the user the ability to dynamically create playlists that take into consideration the play history. This playlist generator gives the user the ability to limit song repetition if she/he so desires.

The playlist generator is presented in a GUI developed in Tkinter. The user inputs the following fields:

* Playlist name,
* Genre
* Exclude tracks played in the last x days
* Maximum amount of tracks in the playlist

Subsequently is presented with doubled entry sliders to choose the range of the following attributes:
* Danceability
* Valence
* Energy
* Loudness
* Accousticness
* Instrumentalness
* Tempo

Acknowledgement: The multiple slider widget is not Tkinter native. Developed by MengxunLi. https://github.com/MenxLi/tkSliderWidget

Based on the user inputs the program conducts the following query:

            SELECT DISTINCT saved_tracks.id FROM saved_tracks
            inner join tracks on saved_tracks.id = tracks.id
            inner join tracks_artists on saved_tracks.id = tracks_artists.id
            inner join genres on tracks_artists.artist_id = genres.artist_id
            WHERE genre like ?
            AND danceability BETWEEN ? AND ?
            AND valence BETWEEN ? AND ?
            AND energy BETWEEN ? AND ?
            AND tempo BETWEEN ? AND ?
            AND acousticness BETWEEN ? AND ?
            AND instrumentalness BETWEEN ? AND ?
            AND saved_tracks.id NOT IN
            (SELECT DISTINCT id FROM history WHERE  (SELECT JULIANDAY('now')  - JULIANDAY(timestamp) < ?)) LIMIT ?""",
               (play_string, dance_lower, dance_upper, valence_lower, valence_upper, energy_lower, energy_upper,
                tempo_lower, tempo_upper, acousticness_lower, acousticness_upper,
                instrumentalness_lower, instrumentalness_upper, days, limit), )


The results from the query are uploaded through the API to a Spotify playlist in the user's account.

## Thanks
A special word of thanks to the Harvard CS50 staff to put together such an amazing course. Cheers.
