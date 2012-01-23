## Loading the Data

First, clone this repository to your local machine.  Then, move into that directoy and run the following command:

    $ mongod --dbpath=data/mongodb-data/

The command above starts the mongod process using the data/mongodb-data directory for the storage.  Next, we will load
our sample data with the following command:

    $ mongoimport -d mlb-games -c games --file=data/games.json --type json --drop

We have just imported into the mlb-games database and the games collection.  Now, we can query the database for a layout
of our data:

    $ echo "db.games.findOne()" | mongo 127.0.0.1:27017/mlb-games
      {
        "_id" : ObjectId("4ee29e4f6cc66f5ec0d9ef74"),
        "game_id" : "2012/03/28/seamlb-oakmlb-1",
        "game_pk" : "317775",
        "game_time_is_tbd" : true,
        "game_venue" : "Tokyo Dome",
        "game_time" : ISODate("2012-03-28T03:33:00Z"),
        "game_time_offset_eastern" : "+0",
        "pitcher" : {
          "loss_id" : null,
          "win_id" : null,
          "win" : null,
          "loss_stat" : null,
          "win_stat" : null,
          "save" : null,
          "save_stat" : null,
          "loss" : null,
          "save_id" : null
        },
        "game_location" : "Tokyo",
        "home" : {
          "file_code" : "oak",
          "probable_stat" : null,
          "tv" : null,
          "id" : "133",
          "split" : false,
          "probable_name_display_first_last" : null,
          "probable_report" : null,
          "recap" : null,
          "full" : "Athletics",
          "radio" : null,
          "tickets" : null,
          "game_time_offset" : "-7",
          "display_code" : "OAK",
          "wrapup" : null,
          "audio_uri" : "",
          "probable" : null,
          "probable_era" : null,
          "league" : "103",
          "probable_id" : null,
          "result" : null
        },
        "division_id" : "200",
        "game_time_offset_local" : null,
        "scheduledTime" : null,
        "game_status" : "S",
        "mlbtv" : true,
        "is_suspension_resumption" : false,
        "video_uri" : "/mediacenter/index.jsp?ymd=20120328",
        "sport_code" : "mlb",
        "venue_id" : "2397",
        "wrapup" : null,
        "preview" : null,
        "game_type" : "R",
        "resumptionTime" : null,
        "away" : {
          "file_code" : "sea",
          "probable_stat" : null,
          "tv" : null,
          "id" : "136",
          "split" : false,
          "probable_name_display_first_last" : null,
          "probable_report" : null,
          "recap" : null,
          "full" : "Mariners",
          "radio" : null,
          "tickets" : null,
          "game_time_offset" : "-7",
          "display_code" : "SEA",
          "wrapup" : null,
          "audio_uri" : "",
          "probable" : null,
          "probable_era" : null,
          "league" : "103",
          "probable_id" : null,
          "result" : null
        },
        "game_dh" : null,
        "game_num" : "1"
      }

You will see that each "game" document has the "home" and "away" team nested attributes, "game_venue" as well as other
attributes that we will want to query.