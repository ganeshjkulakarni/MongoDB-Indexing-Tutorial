# One index per query

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