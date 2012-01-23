However, the indexes do have some limitations and trade-offs.  What if we wanted to find distinct teams visiting Busch Stadium:

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

Again! Awesome.  Now, what if we wanted to query on all games that happen on July 4th?

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

What happens if we remove any of the three conditionals?

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



What if I wanted to run a query on all games in July and sort by Home Team:

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