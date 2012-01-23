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
        "nscanned" : 2444,          // Number of objects scanned
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

Take a look at the code block above.  I've added comments next to the cursor, nscanned, and nscannedObjects fields.
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
        "nscanned" : 81,                               // Number of objects after indexes
        "nscannedObjects" : 81,
        "n" : 81,                                      // Final number of objects found
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

See the number of matching objects: 81.  Wait!  There are 162 games in the MLB season, and the above code only found 81 of them.
That means we are missing 81 other games the Cubs play over the course of the season.  Since we queried the home team
above, we are missing all Cubs' road games.

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

You will see above that Mongo used two indexes for the query.  But the list above said, "2. One index per query". Since
we used the "$or" operator as a top level condition, Mongo runs two queries and concatenates the response.  Now, let's add a query
to find when the Cubs play in St. Louis at Busch Stadium:

    > db.games.find({"away.display_code": "CHC", "game_venue": "Busch Stadium"}).explain();
      {
        "cursor" : "BtreeCursor away.display_code_1",
        "nscanned" : 81,                                // Number found after indexes, thus leading to a table scan
        "nscannedObjects" : 81,
        "n" : 8,                                        // Number after all conditions
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

We have a couple of options here.  Let's see how we would traditionally handle this:

    > db.games.ensureIndex({"game_venue": 1}, {"background": 1});
    > db.games.find({"away.display_code": "CHC", "game_venue": "Busch Stadium"}).explain();
      {
        "cursor" : "BtreeCursor away.display_code_1",
        "nscanned" : 81,
        "nscannedObjects" : 81,
        "n" : 8,
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

Wait!  Only one of our queries was used -- the "away.display_code".  We'd just created a new index "game_venue_1" and it
is not being used.  This is where we run into the "One index per query."  Let's remove this index, and add a more efficient
index:

    > db.games.dropIndex({"game_venue": 1});
    > db.games.ensureIndex({"game_venue": 1, "away.display_code": 1});
    > db.games.find({"away.display_code": "CHC", "game_venue": "Busch Stadium"}).explain();
      {
        "cursor" : "BtreeCursor game_venue_1_away.display_code_1",
        "nscanned" : 8,
        "nscannedObjects" : 8,
        "n" : 8,
        "millis" : 0,
        "nYields" : 0,
        "nChunkSkips" : 0,
        "isMultiKey" : false,
        "indexOnly" : false,
        "indexBounds" : {
          "game_venue" : [
            [
              "Busch Stadium",
              "Busch Stadium"
            ]
          ],
          "away.display_code" : [
            [
              "CHC",
              "CHC"
            ]
          ]
        }
      }

You will see this time, we use the appropriate indexes.  This index serves a dual purpose: we can also use it for
"game_venue" queries:

    > db.games.find({"game_venue": "Busch Stadium"}).explain();
      {
        "cursor" : "BtreeCursor game_venue_1_away.display_code_1",
        "nscanned" : 81,
        "nscannedObjects" : 81,
        "n" : 81,
        "millis" : 0,
        "nYields" : 0,
        "nChunkSkips" : 0,
        "isMultiKey" : false,
        "indexOnly" : false,
        "indexBounds" : {
          "game_venue" : [
            [
              "Busch Stadium",
              "Busch Stadium"
            ]
          ],
          "away.display_code" : [
            [
              {
                "$minElement" : 1
              },
              {
                "$maxElement" : 1
              }
            ]
          ]
        }
      }

However, the query does have some limitations.  What if we wanted to find distinct teams visiting Busch Stadium:

    > db.games.runCommand({distinct: "games", key: "away.display_code", query: {"game_venue": "Busch Stadium"}});
      {
      	"values" : [
      		"ARI",
      		"ATL",
      		"CHC",
      		"CIN",
      		"CLE",
      		"COL",
      		"CWS",
      		"HOU",
      		"KC",
      		"LAD",
      		"MIA",
      		"MIL",
      		"NYM",
      		"PHI",
      		"PIT",
      		"SD",
      		"SF",
      		"WSH"
      	],
      	"stats" : {
      		"n" : 81,
      		"nscanned" : 81,
      		"nscannedObjects" : 0,
      		"timems" : 0,
      		"cursor" : "BtreeCursor game_venue_1_away.display_code_1"
      	},
      	"ok" : 1
      }

That uses the indexes as best it can.  What if we wanted to find games from a range of teams?  Let's query on a range of
the Atlanta Braves, Milwaukee Brewers, and San Francisco Giants:

    > db.games.find({"game_venue": "Busch Stadium", "away.display_code": {$in: ["ARI", "ATL", "SF"]}}).explain();
      {
      	"cursor" : "BtreeCursor game_venue_1_away.display_code_1 multi",
      	"nscanned" : 11,
      	"nscannedObjects" : 10,
      	"n" : 10,
      	"millis" : 0,
      	"nYields" : 0,
      	"nChunkSkips" : 0,
      	"isMultiKey" : false,
      	"indexOnly" : false,
      	"indexBounds" : {
      		"game_venue" : [
      			[
      				"Busch Stadium",
      				"Busch Stadium"
      			]
      		],
      		"away.display_code" : [
      			[
      				"ARI",
      				"ARI"
      			],
      			[
      				"ATL",
      				"ATL"
      			],
      			[
      				"SF",
      				"SF"
      			]
      		]
      	}
      }

Excellent! Again, we only scanned 11 objects.  We are still using our indexes to the maximum potential.  What if we wanted
to find all of the Stadiums the Cubs visit?

  > db.games.runCommand({distinct: "games", key: "game_venue", query: {"away.display_code": "CHC"}});
    {
      "values" : [
        "Busch Stadium",
        "New Marlins Ballpark",
        "Citizens Bank Park",
        "Great American Ball Park",
        "Miller Park",
        "Minute Maid Park",
        "PNC Park",
        "AT&T Park",
        "Target Field",
        "U.S. Cellular Field",
        "Chase Field",
        "Turner Field",
        "Citi Field",
        "Dodger Stadium",
        "PETCO Park",
        "Nationals Park",
        "Coors Field"
      ],
      "stats" : {
        "n" : 81,
        "nscanned" : 81,
        "nscannedObjects" : 81,
        "timems" : 0,
        "cursor" : "BtreeCursor away.display_code_1"
      },
      "ok" : 1
    }

Awesome!  It moves back to our "away.display_code" index.  So, we are moving along very quickly, and most indexes are working efficiently.
What if we wanted to find when the Cubs visit AT&T Park and Nationals Park?

    > db.games.find({"away.display_code": "CHC", "game_venue": {$in: ["AT&T Park", "Nationals Park"]}}).explain()
      {
        "cursor" : "BtreeCursor game_venue_1_away.display_code_1 multi",
        "nscanned" : 10,
        "nscannedObjects" : 8,
        "n" : 8,
        "millis" : 0,
        "nYields" : 0,
        "nChunkSkips" : 0,
        "isMultiKey" : false,
        "indexOnly" : false,
        "indexBounds" : {
          "game_venue" : [
            [
              "AT&T Park",
              "AT&T Park"
            ],
            [
              "Nationals Park",
              "Nationals Park"
            ]
          ],
          "away.display_code" : [
            [
              "CHC",
              "CHC"
            ]
          ]
        }
      }

Again! Awesome.  Now, want if we wanted to query on all games that happen on July 4th?

    // First, let's go ahead and create an index
    > db.games.ensureIndex({"game_time": 1});
    > db.games.find({"game_time": {$gt: new Date(2012,6,4), $lt: new Date(2012,6,5)}}).explain();
      {
      	"cursor" : "BtreeCursor game_time_1",
      	"nscanned" : 15,
      	"nscannedObjects" : 15,
      	"n" : 15,
      	"millis" : 0,
      	"nYields" : 0,
      	"nChunkSkips" : 0,
      	"isMultiKey" : false,
      	"indexOnly" : false,
      	"indexBounds" : {
      		"game_time" : [
      			[
      				ISODate("2012-07-04T05:00:00Z"),
      				ISODate("2012-07-05T05:00:00Z")
      			]
      		]
      	}
      }

Works great!  What if we wanted to find all games in July at Busch Stadium?

    > db.games.find({"game_venue": "Busch Stadium", "game_time": {$gt: new Date(2012,6,1), $lt: new Date(2012,7,1)}}).explain();
      {
      	"cursor" : "BtreeCursor game_venue_1_away.display_code_1",
      	"nscanned" : 81,
      	"nscannedObjects" : 81,
      	"n" : 15,
      	"millis" : 0,
      	"nYields" : 0,
      	"nChunkSkips" : 0,
      	"isMultiKey" : false,
      	"indexOnly" : false,
      	"indexBounds" : {
      		"game_venue" : [
      			[
      				"Busch Stadium",
      				"Busch Stadium"
      			]
      		],
      		"away.display_code" : [
      			[
      				{
      					"$minElement" : 1
      				},
      				{
      					"$maxElement" : 1
      				}
      			]
      		]
      	}
      }

That is not what we expected.  We are using the index to find all games in Busch Stadium, then doing a table scan to perform.
What if our database covered the 100 years of baseball, and we performed a table scan on all games at Busch Stadium in the
past 100 years?  What are our options?  If we were to ever have multiple years of games in one database, then date would
immediately become part of every query.  Let's try this:

    > db.games.ensureIndex({"game_time": 1, "game_venue": 1}, {"background": 1})
    > db.games.find({"game_venue": "Busch Stadium", "game_time": {$gt: new Date(2012,6,1), $lt: new Date(2012,7,1)}}).explain();
      {
      	"cursor" : "BtreeCursor game_venue_1_away.display_code_1",
      	"nscanned" : 81,
      	"nscannedObjects" : 81,
      	"n" : 15,
      	"millis" : 1,
      	"nYields" : 0,
      	"nChunkSkips" : 0,
      	"isMultiKey" : false,
      	"indexOnly" : false,
      	"indexBounds" : {
      		"game_venue" : [
      			[
      				"Busch Stadium",
      				"Busch Stadium"
      			]
      		],
      		"away.display_code" : [
      			[
      				{
      					"$minElement" : 1
      				},
      				{
      					"$maxElement" : 1
      				}
      			]
      		]
      	}
      }

What? Why isn't it using the new game_time index we have just created?  Remember rule #9: "Range query must be the last column"?
Let's remove and re-create our prior index and put the "game_time" last:

    > db.games.ensureIndex({"game_venue": 1, "game_time": 1}, {"background": 1});
    > db.games.find({"game_venue": "Busch Stadium", "game_time": {$gt: new Date(2012,6,1), $lt: new Date(2012,7,1)}}).explain();
      {
        "cursor" : "BtreeCursor game_venue_1_game_time_1",
        "nscanned" : 15,
        "nscannedObjects" : 15,
        "n" : 15,
        "millis" : 0,
        "nYields" : 0,
        "nChunkSkips" : 0,
        "isMultiKey" : false,
        "indexOnly" : false,
        "indexBounds" : {
          "game_venue" : [
            [
              "Busch Stadium",
              "Busch Stadium"
            ]
          ],
          "game_time" : [
            [
              ISODate("2012-07-01T05:00:00Z"),
              ISODate("2012-08-01T05:00:00Z")
            ]
          ]
        }
      }

Now, we know every index should contain "game_time" as the last field.  Let's rebuild indexes to ensure game_time is the last field:

    > db.system.indexes.find();
      { "v" : 1, "key" : { "_id" : 1 }, "ns" : "mlb-games.games", "name" : "_id_" }
      { "v" : 1, "key" : { "home.display_code" : 1 }, "ns" : "mlb-games.games", "name" : "home.display_code_1", "background" : 1 }
      { "v" : 1, "key" : { "away.display_code" : 1 }, "ns" : "mlb-games.games", "name" : "away.display_code_1", "background" : 1 }
      { "v" : 1, "key" : { "game_venue" : 1, "away.display_code" : 1 }, "ns" : "mlb-games.games", "name" : "game_venue_1_away.display_code_1" }
      { "v" : 1, "key" : { "game_time" : 1 }, "ns" : "mlb-games.games", "name" : "game_time_1" }
      { "v" : 1, "key" : { "game_venue" : 1, "game_time" : 1 }, "ns" : "mlb-games.games", "name" : "game_venue_1_game_time_1" }

    // Show off some Javascripting skills :)
    // Let's add a keys function real quick like:
    > if(!Object.keys) Object.keys = function(o){
         if (o !== Object(o))
            throw new TypeError('Object.keys called on non-object');
         var ret=[],p;
         for(p in o) if(Object.prototype.hasOwnProperty.call(o,p)) ret.push(p);
         return ret;
      }

    > db.system.indexes.find({"ns": "mlb-games.games", "name": {$ne: "_id_"}}).forEach(function(row) {
        if (Object.keys(row.key).indexOf("game_time") < 0) {
          db.games.dropIndex(row.key);
          row.key["game_time"] = 1;
          db.games.ensureIndex(row.key, {"background": 1});
        }
      });

    > db.system.indexes.find();

We've added game_time to the end of all of our indexes.  This will allow us to do time range queries accompanied with any of our other queries.
What if we wanted to run queries to find either the Cubs or Giants visiting Busch Stadium in July?

    > db.games.find({"game_venue": "Busch Stadium", "away.display_code": {$in: ["CHC", "SF"]}, "game_time": {$gt: new Date(2012, 6, 1), $lt: new Date(2012, 7, 1)}}).explain();
      {
      	"cursor" : "BtreeCursor game_venue_1_away.display_code_1_game_time_1 multi",
      	"nscanned" : 4,
      	"nscannedObjects" : 3,
      	"n" : 3,
      	"millis" : 0,
      	"nYields" : 0,
      	"nChunkSkips" : 0,
      	"isMultiKey" : false,
      	"indexOnly" : false,
      	"indexBounds" : {
      		"game_venue" : [
      			[
      				"Busch Stadium",
      				"Busch Stadium"
      			]
      		],
      		"away.display_code" : [
      			[
      				"CHC",
      				"CHC"
      			],
      			[
      				"SF",
      				"SF"
      			]
      		],
      		"game_time" : [
      			[
      				ISODate("2012-07-01T05:00:00Z"),
      				ISODate("2012-08-01T05:00:00Z")
      			]
      		]
      	}
      }

What happens if we remove any of the three queries?

    // Remove the Away Display Code
    > db.games.find({"game_venue": "Busch Stadium", "game_time": {$gt: new Date(2012, 6, 1), $lt: new Date(2012, 7, 1)}}).explain();
      {
        "cursor" : "BtreeCursor game_venue_1_game_time_1",
        ...
      }

    // Remove the Game Venue
    > db.games.find({"away.display_code": {$in: ["ARI", "ATL", "SF"]}, "game_time": {$gt: new Date(2012, 6, 1), $lt: new Date(2012, 7, 1)}}).explain();
      {
      	"cursor" : "BtreeCursor away.display_code_1_game_time_1 multi",
      	...
      }

    // Remove the Date Filter
    > db.games.find({"away.display_code": {$in: ["ARI", "ATL", "SF"]}}).explain();
      {
      	"cursor" : "BtreeCursor away.display_code_1_game_time_1 multi",
      	...
      }

Sweet!  What if I wanted to run a query on all games in July and sort by Home Team:

    // Sort by Game Venue
    > db.games.find({"game_time": {$gt: new Date(2012, 6, 1), $lt: new Date(2012, 7, 1)}}).sort({"game_venue": 1}).explain();
      {
        "cursor" : "BtreeCursor game_time_1",
        "nscanned" : 384,
        "nscannedObjects" : 384,
        "n" : 384,
        "scanAndOrder" : true,
        "millis" : 5,
        "nYields" : 0,
        "nChunkSkips" : 0,
        "isMultiKey" : false,
        "indexOnly" : false,
        "indexBounds" : {
          "game_time" : [
            [
              ISODate("2012-07-01T05:00:00Z"),
              ISODate("2012-08-01T05:00:00Z")
            ]
          ]
        }
      }

See the "scanAndOrder"?  That is bad.  What happens if we were sorting by date?

    // Sort by Date
    > db.games.find({"game_time": {$gt: new Date(2012, 6, 1), $lt: new Date(2012, 7, 1)}}).sort({"game_time": 1}).explain();
      {
      	"cursor" : "BtreeCursor game_time_1",
      	"nscanned" : 384,
      	"nscannedObjects" : 384,
      	"n" : 384,
      	"millis" : 0,
      	"nYields" : 0,
      	"nChunkSkips" : 0,
      	"isMultiKey" : false,
      	"indexOnly" : false,
      	"indexBounds" : {
      		"game_time" : [
      			[
      				ISODate("2012-07-01T05:00:00Z"),
      				ISODate("2012-08-01T05:00:00Z")
      			]
      		]
      	}
      }

This time -- no "scanAndOrder."  And it is faster, the will always be around the same time.  The trade-off for having
date queries at the end of the databases is slower sorts on all other fields.