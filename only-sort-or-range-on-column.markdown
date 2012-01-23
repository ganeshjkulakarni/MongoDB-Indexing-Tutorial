# Only Sort or Range on One Column

Going into this Chapter, the following queries should be created:

    > db.ensureIndex({ "game_time" : 1 });
    > db.ensureIndex({ "game_venue" : 1, "game_time" : 1 });
    > db.ensureIndex({ "home.display_code" : 1, "game_time" : 1 });
    > db.ensureIndex({ "away.display_code" : 1, "game_time" : 1 });
    > db.ensureIndex({ "game_venue" : 1, "away.display_code" : 1, "game_time" : 1 });

Given the chapter heading, and the examples above, you can see where we are going.  We are going to find all games
in the month of July, then sort them based on the home team and the game_time.

    > db.games.find({ "game_time" : {$gt: new Date(2012, 6, 1), $lt: new Date(2012, 7, 1)}}).sort({"home.display_code": 1, "game_time": 1}).explain();
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

We had another scanAndOrder.  Therefore, it will never use a truly optimized sort.