# $ne or $nin Operations Aren't Efficient with Indexes

You can use $ne or $nin, but know they are not going to be efficiently executed.  The best example is an easy one.  Let's
say we HATE the Cubs (which we've previously said we loved), and find all games which do not contain the Cubs:

    > db.games.find({$and: [{"away.display_code": {$ne: "CHC"}}, {"home.display_code": {$ne: "CHC"}}]})
      {
      	"cursor" : "BtreeCursor home.display_code_1_game_time_1 multi",
      	"nscanned" : 2364,
      	"nscannedObjects" : 2363,
      	"n" : 2282,
      	"millis" : 27,
      	"nYields" : 0,
      	"nChunkSkips" : 0,
      	"isMultiKey" : false,
      	"indexOnly" : false,
      	"indexBounds" : {
      		"home.display_code" : [
      			[
      				{
      					"$minElement" : 1
      				},
      				"CHC"
      			],
      			[
      				"CHC",
      				{
      					"$maxElement" : 1
      				}
      			]
      		],
      		"game_time" : [
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

That query above is obviously slow.  But is it any slower than the inverse?

    > db.games.find({$and: [{"away.display_code": {$in: db.games.distinct("away.display_code")}}, {"home.display_code": {$ne: db.games.distinct("away.display_code")}}]}).explain();
      {
      	"cursor" : "BtreeCursor home.display_code_1_game_time_1 multi",
      	"nscanned" : 2444,
      	"nscannedObjects" : 2444,
      	"n" : 2444,
      	"millis" : 31,
      	"nYields" : 0,
      	"nChunkSkips" : 0,
      	"isMultiKey" : false,
      	"indexOnly" : false,
      	"indexBounds" : {
      		"home.display_code" : [
      			[
      				{
      					"$minElement" : 1
      				},
      				[
      					"ARI",
      					...,
      					"WSH"
      				]
      			],
      			[
      				[
      					"ARI",
      					...,
      					"WSH"
      				],
      				{
      					"$maxElement" : 1
      				}
      			]
      		],
      		"game_time" : [
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

Well, it turns our answer is "no", it's not any faster.  Instead of "$ne and $nin" operators are not efficient, just say
"pick your poison."