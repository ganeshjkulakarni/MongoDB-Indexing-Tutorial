# Welcome to MongoHQ's Index Tutorial for MongoDB

Welcome to our tutorial for MongoDB.  The purpose of this project is to layout some examples that can be found at

http://www.mongodb.org/display/DOCS/Indexing+Advice+and+FAQ

We will provide some use cases based on the provided JSON document in data/games.json.

## Assumptions

Please install MongoDB for your operating system prior to begining this tutorial.

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
attributes that we will want to query.  As stated in the beginning, we will follow the general rules for creating indexes
and queries with Mongo:

  1. Create indexes to match your queries
  2. One index per query
  3. Make sure your indexes can fit in RAM
  4. Be careful about single-key indexes with low selectivity
  5. Use explain
  6. Read - Write Ration
  7. Sort column must be teh last column
  8. Range query must be the last column
  9. Only sort or range on one column
  10. $ne or $nin operators aren't efficient with indexes

# Create Indexes to Match Your Queries

When people use my application, the application should be able to query games based on:

  1. Teams -- People may have a favorite team and want their schedule for the entire season.
  2. Location -- People may want to visit their favorite stadium
  3. Dates -- People may have certain times they are available to go to games, and want to query on that.
  4. People may want to query any combination fo the above

Let's run our queries and determine how some of them run:

    > db.games.findOne({"home.display_code": "CHC"});                                      // Chicago Cubs Games
    > db.games.findOne({"game_venue": "Wrigley Field"});                                   // Games at Wrigley Field
    > db.games.findOne({"game_time": {$gt: new Date(2012,6,4), $lt: new Date(2012,6,5)}}); // July 4th Games

    // Chicago Cub Game at Wrigley Field Before Nearest to July 4th Weekend
    > db.games.find({"home.display_code": "CHC", "game_venue": "Wrigley Field", "game_time": {$lt: new Date(2012,6,5)}}).sort({game_time: -1}).limit(1);

Before we create an index, let's explain one of them:

    > db.games.find({"home.display_code": "CHC"}).explain();
    {
      "cursor" : "BasicCursor",   // Not Indexed
      "nscanned" : 2444,          // Number Scanned
      "nscannedObjects" : 2444,
      "n" : 81,
      "millis" : 5,               // Length of Time
      "nYields" : 0,
      "nChunkSkips" : 0,
      "isMultiKey" : false,
      "indexOnly" : false,
      "indexBounds" : {

      }
    }

Let's create a query on home team, and redo our last query:

    // While background is not required it, it should always be used when application not in maintenance mode
    > db.games.ensureIndex({"home.display_code": 1}, {"background": 1})

    // List our Indexes
    > db.system.indexes.find();
    { "v" : 1, "key" : { "_id" : 1 }, "ns" : "mlb-games.games", "name" : "_id_" }
    { "v" : 1, "key" : { "home.display_code" : 1 }, "ns" : "mlb-games.games", "name" : "home.display_code_1", "background" : 1 }

    // Now, rerun our prior query
    > db.games.find({"home.display_code": "CHC"}).explain();
    {
      "cursor" : "BtreeCursor home.display_code_1",  // Indexed and which index you used
      "nscanned" : 81,                               // Number of objects matching
      "nscannedObjects" : 81,
      "n" : 81,
      "millis" : 2,                                  // Length of time reduced by 1/2, run again and you will see it runs much faster
      "nYields" : 0,
      "nChunkSkips" : 0,
      "isMultiKey" : false,
      "indexOnly" : false,
      "indexBounds" : {
        "home.display_code" : [
          [
            "CHC",
            "CHC"
          ]
        ]
      }
    }

Wait!  But there are 162 games in the MLB season, and the above code only found 81 of them.  That means we are missing
81 other games the Cubs play over the course of the season.  Since we queried the home team above, we are missing all
Cubs' road games.

    > db.games.find({"away.display_code": "CHC"}).explain();
    {
      "cursor" : "BasicCursor",                      // Index is not used because we haven't created on yet
      "nscanned" : 2444,
      "nscannedObjects" : 2444,
      "n" : 81,
      "millis" : 5,
      "nYields" : 0,
      "nChunkSkips" : 0,
      "isMultiKey" : false,
      "indexOnly" : false,
      "indexBounds" : {

      }
    }
    > db.games.ensureIndex({"away.display_code": 1}, {"background": 1});

    // Now, we should be able to run a query and it use the index
    > db.games.find({"away.display_code": "CHC"}).explain();
    {
      "cursor" : "BtreeCursor away.display_code_1",
      "nscanned" : 81,
      "nscannedObjects" : 81,
      "n" : 81,
      "millis" : 0,
      "nYields" : 0,
      "nChunkSkips" : 0,
      "isMultiKey" : false,
      "indexOnly" : false,
      "indexBounds" : {
        "away.display_code" : [
          [
            "CHC",
            "CHC"
          ]
        ]
      }
    }

    > db.games.find({$or: [{"away.display_code": "CHC"}, {"home.display_code": "CHC"}]})
    {
      "clauses" : [
        {
          "cursor" : "BtreeCursor away.display_code_1",
          "nscanned" : 81,
          "nscannedObjects" : 81,
          "n" : 81,
          "millis" : 0,
          "nYields" : 0,
          "nChunkSkips" : 0,
          "isMultiKey" : false,
          "indexOnly" : false,
          "indexBounds" : {
            "away.display_code" : [
              [
                "CHC",
                "CHC"
              ]
            ]
          }
        },
        {
          "cursor" : "BtreeCursor home.display_code_1",
          "nscanned" : 81,
          "nscannedObjects" : 81,
          "n" : 81,
          "millis" : 1,
          "nYields" : 0,
          "nChunkSkips" : 0,
          "isMultiKey" : false,
          "indexOnly" : false,
          "indexBounds" : {
            "home.display_code" : [
              [
                "CHC",
                "CHC"
              ]
            ]
          }
        }
      ],
      "nscanned" : 162,
      "nscannedObjects" : 162,
      "n" : 162,
      "millis" : 1
    }

You will see above that Mongo used two indexes for the query.  Since they were top level conditions, it concatenated the
results of two queries to get the response.