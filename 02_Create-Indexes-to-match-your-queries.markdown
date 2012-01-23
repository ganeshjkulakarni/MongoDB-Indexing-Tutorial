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