Using the `{pool}` package for managing database connections
================
Nov 20, 2020



This is a synopsis of the longer `{pool}` article from RStudio posted [here](!https://shiny.rstudio.com/articles/pool-basics.html).

## Database Connections

To communicate with a database, we need to first create a `connection` to the database server.

Database connections are persistent, meaning that we may reuse one connection for mutliple queries and must take special care to make sure we close connections when we no longer need them.

Databases impose a limit on the number of connections allowed at any give moment - for Postgres databases, the default limit is 100 connections.


## Challenges With Connection Limits
When the maximum number of connections is already reached for a database, the next client that tries to create a connection will receive a max limit connection error and will fail to connect.

For example, suppose we had 100 apps, each of which grabs one database connection to a Postgres DB to use for all communication to the DB (this isn't best practice as we'll see below). The 101st app to try to connect to the same DB would fail.

Further, if an app makes a connection to a database but does not have a robust cleanup routine to close the connection once the app is terminated, it could occupy a connection indefinitely (or until a DB admin restarts the DB server or purges it's connections, likely causing disruptions to other apps).

This is why our apps typically include some DB cleanup code:

``` {R}
# When this shiny session ends (eg. when user closes their browser), close the
# database connection.
session$onSessionEnded(function(){
  DBI:dbDisconnect(con)
})
```

## Problem
How many connections should I maintain for my app?

:x: Too many and my app may 'hog' more of the max connections than it really needs.

:x: Too few and my app may be slow since multiple queries from different users may have to share a single DB connection (# of connections can be a bottleneck).

:heavy_check_mark: Ideally, # of connections would scale according to need.

To flush this out a bit more into examples below:
1. A greedy approach (too many connections)
2. An under-resourced approach (too few connections)
3. Connection pooling (scaling!)

### :x: Approach 1: A Greedy Approach - 1 Connection per 'user'/session

1. Every new `session` (ie. every new user to visit a dashboard) gets a new DB connection, which is used for all queries needed for that session.
2. When the session ends (eg. the user closes their window), some cleanup routine closes the connection (such as with `session$onSessionEnded`)

#### Pros:
- App will not be bottlenecked by connections since each user gets their very own connection - queries will execute fast
- Robust to DB failures - if the DB server goes down and all connections drop, and then the DB server starts up again, the next user to open the app will get a valid connection without needing to restart the app.

#### Cons:
- App will occupy far more connections than it needs - can quickly deplete the 100 max connections. Other apps accessing this DB will also be affected.
- (related) if the number of current users viewing the app is greater than the max connections, app will fail for new visitors until connections are freed.

### :x: Approach 2: Under-Resourced Approach - 1 Connection Per App
1. Dashboard app starts and grabs 1 connection.
2. All queries to the DB use that one connection.
3. When the Dashboard app is terminated (ie. the app is taken down - not the same as a 'session' ending) some cleanup code runs to close the connection.

#### Pros:
- App does not occupy many connections, the max limits are probably not an issue here.

#### Cons:
- Having just 1 connection will be a bottleneck for multiple queries from different people.
- If the connection is dropped for any reason (eg. DB needs a restart, DB server fails, network issue, etc...) the app will fail trying to use a broken connection. The app will need to be re-started to get a new connection.

### :heavy_check_mark: Approach 3 Connection Pooling with `{pool}`
1. On App start, initialize a connection `pool` with all the usual database connection parameters (host, port, db name, user, pwd)
2. The `pool` maintains one or more db connections and opens/closes connections scaling to the demand.
3. To make a query, get a connection from the `pool`, make the query with the connection, then return the connection to the `pool` so it can be used for other queries or closed if not needed.
4. On app termination, close the `pool` so it cleans up any connections it may still hold.


#### Pros:
- Number of DB connections are appropriately scaled to demand.
- If a connection drops, a new one is created without needing to restart the app - robust to network errors or DB Server issues

#### Cons:
- A bit more complicated to implement (luckily `{pool}` does this for us)


## R Examples using `{pool}`

Install from CRAN:
``` {R}
renv::install("pool")
```

At the top level of your app, instantiate a pool, and register a callback to close the pool when the app terminates:
``` {R}
run_app <- function(...){

  ...
  # Initialize the pool
  db_pool <- pool::dbPool(
    # DB driver: postgres here, could also use RMySQL
    drv = RPostgres::Postgres(),
    # All the usual DB init suspects...
    dbname = dbname,
    host = host,
    port = port,
    user = user,
    password = password
  )

  # Make sure the pool is closed when the app stops
  shiny::onStop(function(){
    pool::poolClose(db_pool)
  })

  # Use the db_pool object anywhere in your app...
}

```

Making queries:
```
full_table <- pool::dbReadTable(db_pool, "table_name")

get_one_query <- "SELECT * FROM table WHERE id=2;"
table_row <-  pool::dbGetQuery(db_pool, get_one_query)

update_query <- "UPDATE table SET some_column=245 WHERE id=2 RETURNING *;"
updated_entry <- pool::dbGetQuery(db_pool, update_query)
```

For Most cases `dbReadTable` and `dbGetQuery` are sufficient. In these functions, under the hood `{pool}` gets a connection (or creates one if necessary), makes the query, and then returns the connection to the pool.

### Advanced Use

In some cases we need to access an actual DB connection object - for instance, when we're using another R package that requires a `connection` and not a `pool`.

`con <- pool::poolCheckout(db_pool)` will provide us with a connection object from the pool, but then we need to be careful to make sure we return the connection back to the pool when we're done with it.

A good practice for doing this is to encapsulate whatever operation needs a `con` inside a function, and return the connection to the pool in the `on.exit` handler of the function.

For example:
``` R
my_func_using_a_con <- function(db_pool){
  con <-pool::poolCheckout(db_pool)
  # This function will get called regardless of how my_func_using_a_con exits
  # eg. if an error occurs in my_func_using_a_con, the con will still be
  #     returned to the pool
  on.exit(function(){
    pool::poolReturn(con)
  })

  # ... do stuff with con ...
}
```

#### A real use case for this?

`glue::glue_sql` is a super handy function for adding some safety to SQL strings, from the docs:

> SQL databases often have custom quotation syntax for identifiers and strings which make writing SQL queries error prone and cumbersome to do. glue_sql() and glue_data_sql() are analogs to glue() and glue_data() which handle the SQL quoting.

`glue::glue_sql` expects a `.con` parameter corresponding to a DB connection - passing it a `pool` object will fail though.

So we can wrap `glue_sql` in another utility function that will use `pool` instead:
``` {R}
pool_glue_sql <- function(db_pool, query, values) {
    # Retrieve a con and register to return it after this function exits.
    con <- pool::poolCheckout(db_pool)
    on.exit({
        pool::poolReturn(con)
    })

    # call glue_sql with .con=con
    # (dw about the do.call bit if this is unfamiliar - sufficient for this tip
    #  just to understand that we're just calling glue_sql with con here)
    do.call(glue::glue_sql, c(list(query, .con = con), values))
}
```

#### Too much boilerplate?

We can (and should) define a more abstract utility function that takes a db_pool, and will safely perform some function with a connection retrieved from the pool:

``` R
with_con_from_pool <- function(db_pool, func){
  # Retrieve a con and register to return it after this function exits.
  con <- pool::poolCheckout(db_pool)
  on.exit({
      pool::poolReturn(con)
  })
  func(con)
}

output_of_glue_sql <- with_con_from_pool(db_pool, function(con){
  glue::glue_sql(.con=con, ...other_args_here...)
})
```
