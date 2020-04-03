# Data Modelling and ETL in Postgres + Python

This is the first project submission for the Udacity Data Engineering Nanodegree.
This project consists of putting into practice the following concepts:
- Data modeling and creating a star-schema database with Postgres (using the `psycopg2` Python driver) 
- Building an ETL pipeline using Python

## Context

### Project Specification

A startup called Sparkify wants to analyze the data they've been collecting on songs and user activity on their new music streaming app. 

Currently they don't have an easy way to query their data, which consists of:
```
i) a directory of JSON logs on user activity on the app (*/data/log_data*)
ii) a directory with JSON metadata on the songs in their app (*/data/song_data*)
```

The minimum project specification is to create a database schema and ETL pipeline for this analysis.

### Data
- **Song datasets**: all json files are nested in subdirectories under */data/song_data*. A sample a single row of each file is:

```
{"num_songs": 1, "artist_id": "ARJIE2Y1187B994AB7", "artist_latitude": null, "artist_longitude": null, "artist_location": "", "artist_name": "Line Renaud", "song_id": "SOUPIRU12A6D4FA1E1", "title": "Der Kleine Dompfaff", "duration": 152.92036, "year": 0}
```

- **Log datasets**: all json files are nested in subdirectories under */data/log_data*. A sample of a single row of each file is:

```
{"artist":"Sydney Youngblood", "auth":"Logged In", "firstName":"Jacob", "gender":"M", "itemInSession":53, "lastName":"Klein", "length":238.07955, "level":"paid", "location":"Tampa-St. Petersburg-Clearwater, FL", "method":"PUT", "page":"NextSong", "registration":1.540558e+12, "sessionId":954, "song":"Ain't No Sunshine", "status":200, "ts":1543449657796, "userAgent":"\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_4...)"", "userId":"73"}
```

## Database Schema
There is one main fact table containing all the measures associated to each event (`songplays`), 
and 4 dimensional tables, each with a primary key that is being referenced from the fact table. The database follows a star schema.

On why to use a relational database for this case:
- The ability to use SQL queries is sufficient for this type of analysis, and the ability to JOIN tables is necessary
- There are no concerns around data availability, query latency or horizontal scalability 
- The data types are structured
- The amount of data we need to analyze is not big enough to require big data related solutions


#### Fact Table
**songplays** - records in log data associated with song plays i.e. records with page NextSong
- songplay_id (VARCHAR) PRIMARY KEY: ID of each user song play 
- start_time (TIMESTAMP): Timestamp of beginning of user activity
- user_id (INT) NOT NULL: ID of user
- level (VARCHAR): User level {free | paid}
- song_id (VARCHAR) NOT NULL: ID of Song played
- artist_id (VARCHAR) NOT NULL: ID of Artist of the song played
- session_id (INT): ID of the user Session 
- location (VARCHAR): User location 
- user_agent (VARCHAR): Agent used by user to access Sparkify platform

#### Dimension Tables
**users** - data on individual Sparkify users
- user_id (INT) PRIMARY KEY: ID of user
- first_name (VARCHAR) NOT NULL: First Name of user
- last_name (VARCHAR) NOT NULL: Last Name of user
- gender (VARCHAR): Gender of user {M | F}
- level (VARCHAR): User level {free | paid}

**songs** - data on individual songs in the Sparkify database
- song_id (VARCHAR) PRIMARY KEY: ID of Song
- title (VARCHAR) NOT NULL: Title of Song
- artist_id (VARCHAR) NOT NULL: ID of Artist
- year (INT): Year of song release
- duration (FLOAT) NOT NULL: Song duration in milliseconds

**artists** - data on artists in the Sparkify database
- artist_id (VARCHAR) PRIMARY KEY: ID of Artist
- name (VARCHAR) NOT NULL: Name of Artist
- location (VARCHAR): Name of Artist city
- lattitude (FLOAT): Lattitude location of artist
- longitude (FLOAT): Longitude location of artist

**time** - timestamps of records in songplays broken down into specific units
- start_time (DATE) PRIMARY KEY: Timestamp of row
- hour (INT): Hour associated to start_time
- day (INT): Day associated to start_time
- week (INT): Week of year associated to start_time
- month (INT): Month associated to start_time 
- year (INT): Year associated to start_time
- weekday (VARCHAR): Name of weekday associated to start_time (eg Monday, Tuesday etc)


## Project structure

### Files used on the project

1. **data** folder nested at the home of the project
2. **sql_queries.py** contains all SQL queries needed to initiate the tables, and is imported into the files below.
3. **create_tables.py** drops and creates tables. This file is run to reset tables each time before the ETL scripts are run.
4. **test.ipynb** displays the first few rows of each table to inspect the database.
5. **etl.ipynb** reads and processes a single file from song_data and log_data and loads the data into tables. 
6. **etl.py** reads and processes all files from `song_data` and `log_data` and loads them into your tables. 
7. **README.md** current file, provides overview of the project.

### Step-by-step overview of project

1. Write DROP, CREATE and INSERT query statements in `sql_queries.py`
2. Run in terminal
 ```
python create_tables.py
```
3. Use `test.ipynb` Jupyter Notebook to interactively verify that all tables were created correctly.
4. Follow the instructions and complete `etl.ipynb` Notebook to create the blueprint of the pipeline to process and insert all data into the tables.
5. Verify that base steps are correct by checking with `test.ipynb`
6. Fill in `etl.py` script using `etl.ipynb` as a base.
6. Run etl in terminal, and verify results:
 ```
python etl.py
```

### ETL pipeline logic

In-depth breakdown of each step is available in `etl.ipynb`, but a summary of the overall logic:

1. Connect to the Sparkify database.
2. Cycle through the files under `/data/song_data`, and for each JSON file encountered send the file to a function called `process_song_file`.
3. This function loads each song file into a dataframe, and selects the fields needed to generate the tables before inserting into the respective databases:  
    ```
    song_data = [song_id, title, artist_id, year, duration]
    ```
    ```
    artist_data = [artist_id, artist_name, artist_location, artist_longitude, artist_latitude]
    ``` 
4. Cycle through the tree files under `/data/log_data`, and for each JSON file encountered send the file to a function called `process_log_file`.
5. This function loads each song file where `page = 'NextSong'` into a dataframe, and selects the fields needed to generate the tables before inserting into the respective databases.  
    1. We convert the `ts` attribute from Unix time to a timestamp. This is then used as the `start_time` field in the `time_data` and all additional fields are generated using the `datetime` library. 
    ```
    time_df = [timestamp, hour, day, week_of_year, month, year, weekday]
    ```
    ```
    user_df = [userId, firstName, lastName, gender, level]
    ```
6. Lookup the `songId` and `artistId` from their respective tables by song name, artist name and song duration in order to complete the `songplay` fact table. The query used is the following:
    ```
    song_select = ("""
        SELECT song_id, artists.artist_id
        FROM songs JOIN artists ON songs.artist_id = artists.artist_id
        WHERE songs.title = %s
        AND artists.name = %s
        AND songs.duration = %s
    """)
    ```
7. The last step is inserting everything into the `songplay` fact table.

## Project checklist

  - [x] Table creation script (`create_table.py`) runs without errors
  - [x] Fact and dimensional tables for a star schema are properly defined
  - [x] ETL script (`etl.py`) runs without errors
  - [x] ETL script properly processes transformations in Python 
  - [x] The project shows proper use of documentation
  - [x] The project code is clean and modular
  - [ ] (BONUS) Insert data using the COPY command to bulk insert log files instead of using INSERT on one row at a time
  - [ ] (BONUS) Add data quality checks
  - [ ] (BONUS) Create a dashboard for analytic queries on your new database
