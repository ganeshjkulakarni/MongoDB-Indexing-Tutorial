# Welcome to MongoHQ's Index Tutorial for MongoDB

Welcome to our tutorial for MongoDB.  The purpose of this project is to layout some examples that can be found at

http://www.mongodb.org/display/DOCS/Indexing+Advice+and+FAQ

We will provide some use cases based on the provided JSON document in data/games.json.

## Assumptions

Please install MongoDB for your operating system prior to begining this tutorial.

All of the chapters assume you have completed the prior chapter, and have loaded data.

[Loading Data](https://github.com/MongoHQ/MongoHQ-Index-Tutorial/blob/master/Loading-Data.markdown)

As stated in the beginning, we will follow the general rules for creating indexes
and queries with Mongo:

  1. Use explain
    - We will use explain so much that you will get tired of seeing it :)
  2. [Create indexes to match your queries](https://github.com/MongoHQ/MongoHQ-Index-Tutorial/blob/master/Create-Indexes-to-match-your-queries.markdown)
  3. [One index per query](https://github.com/MongoHQ/MongoHQ-Index-Tutorial/blob/master/One-Index-Per-Query.markdown)
  4. Be careful about single-key indexes with low selectivity
  5. [Sort column and Range Queries must be the last column](https://github.com/MongoHQ/MongoHQ-Index-Tutorial/blob/master/Sort-Column-and-Range-Queries-msut-be-the-Last-Column.markdown)
  7. [Only Sort or Range on One Column](https://github.com/MongoHQ/MongoHQ-Index-Tutorial/blob/master/only-sort-or-range-on-column.markdown)
  8. $ne or $nin operators aren't efficient with indexes
  9. Make sure your indexes can fit in RAM
  10. Read - Write Ration

