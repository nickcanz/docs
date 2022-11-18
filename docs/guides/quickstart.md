# Quickstart with ReadySet

This page shows you how to test ReadySet on your local machine using Docker.

You'll start a Postgres database, load sample data into it, connect ReadySet, cache some queries, and test how fast ReadySet returns results compared to Postgres. You'll then write to the database and see how ReadySet keeps your cache up-to-date automatically, with no changes to your application code.

## Before you begin

Make sure you have [Docker](https://docs.docker.com/engine/install/) installed and running.

## Step 1. Start the database

ReadySet sits between your database and application, so in this step, you'll start up a local instance of Postgres.

1. Create a new container and start Postgres inside it:

    ``` sh
    docker run -d \
    --name=postgres \
    --publish=5432:5432 \
    -e POSTGRES_PASSWORD=readyset \
    -e POSTGRES_DB=imdb \
    postgres:14 \
    -c wal_level=logical
    ```

2. Take a moment to understand the flags you used:

    <style>
      table thead tr th:first-child {
        width: 170px;
      }
    </style>

    Flag | Details
    -----|--------
    `-d` | Runs the container in the background so you can continue the next steps in the same terminal.
    `--name` | Assigns a name to the container so you can easily reference it later in Docker commands.
    `--publish` | Maps the Postgres port from the container to the host so you can access the database from outside of Docker.
    `-e` | Sets environment variables to create a password for the default `postgres` user and to create a database. You'll use these details when connecting to Postgres.
    `-c` |  Turns on Postgres [logical replication](https://www.postgresql.org/docs/current/logical-replication.html). Once ReadySet has taken an initial snapshot of the database, it uses the logical replication stream to keep its snapshot and caches up-to-date as the database changes. You'll see this in action in [Step 6](#step-6-cause-a-cache-refresh).   

## Step 2. Load sample data

In this step, you'll load two sample tables from the [IMDb dataset](https://www.imdb.com/interfaces/) so you have some data to query.

1. Create a second container for downloading sample data and running the `psql` client:

    ``` sh
    docker run -dit \
    --name=psql \
    postgres:14 \
    bash
    ```

2. Get into the container and install some dependencies for downloading the sample data:

    ``` sh
    docker exec -it psql bash
    ```

    ``` sh
    apt-get update
    ```

    ``` sh
    apt-get -y install curl unzip
    ```

3. Download CSV files containing data for these tables from the [IMDb dataset](https://www.imdb.com/interfaces/):

    ``` sh
    curl -O https://raw.githubusercontent.com/readysettech/docs/main/docs/assets/quickstart_sample_data.zip \
    && unzip quickstart_sample_data.zip
    ```

4. Use the `psql` client to create the schema for 2 tables, `title_basics` and `title_ratings`:

    ``` sh
    PGPASSWORD=readyset psql \
    --host=host.docker.internal \
    --port=5432 \
    --username=postgres \
    --dbname=imdb \
    -c "CREATE TABLE title_basics (
          tconst TEXT PRIMARY KEY,
          titletype TEXT,
          primarytitle TEXT,
          originaltitle TEXT,
          isadult BOOLEAN,
          startyear INT,
          endyear INT,
          runtimeminutes INT,
          genres TEXT
        );"
    ```

    ``` sh
    PGPASSWORD=readyset psql \
    --host=host.docker.internal \
    --port=5432 \
    --username=postgres \
    --dbname=imdb \
    -c "CREATE TABLE title_ratings (
          tconst TEXT PRIMARY KEY,
          averagerating NUMERIC,
          numvotes INT
        );"
    ```

5. Load the data into each table:

    ``` sh
    PGPASSWORD=readyset psql \
    --host=host.docker.internal \
    --port=5432 \
    --username=postgres \
    --dbname=imdb \
    -c "\copy title_basics
        from 'title_basics.tsv'
        with DELIMITER E'\t'"
    ```

    ``` sh
    PGPASSWORD=readyset psql \
    --host=host.docker.internal \
    --port=5432 \
    --username=postgres \
    --dbname=imdb \
    -c "\copy title_ratings
        from 'title_ratings.tsv'
        with DELIMITER E'\t'"
    ```

    These commands will take a few minutes, as they load 5159701 rows into `title_basics` and 1246402 rows into `title_ratings`.

6. Open the `psql` shell and get a sense of the data in each table:

    ``` sh
    PGPASSWORD=readyset psql \
    --host=host.docker.internal \
    --port=5432 \
    --username=postgres \
    --dbname=imdb
    ```

    ``` sql
    SELECT * FROM title_basics WHERE tconst = 'tt0093779';
    SELECT * FROM title_ratings WHERE tconst = 'tt0093779';
    ```

    ``` {.text .no-copy}
      tconst   | titletype |    primarytitle    |   originaltitle    | isadult | startyear | endyear | runtimeminutes |          genres
    -----------+-----------+--------------------+--------------------+---------+-----------+---------+----------------+--------------------------
     tt0093779 | movie     | The Princess Bride | The Princess Bride | f       |      1987 |         |             98 | Adventure,Family,Fantasy
    (1 row)

      tconst   | averagerating | numvotes
    -----------+---------------+----------
     tt0093779 |           8.0 |   427192
    (1 row)
    ```

7. Exit the `psql` shell:

    ``` sql
    \q
    ```

8. Exit the container:

    ``` sh
    exit
    ```

## Step 3. Connect ReadySet

Now that you have a live database with sample data, you'll connect ReadySet to the database and watch it take a snapshot of your tables. This snapshot will be the basis for ReadySet to cache query results.

1. Create a third container and start ReadySet inside it, connecting ReadySet to your Postgres database via the connection string in `--upstream-db-url`:

    ``` sh
    docker run -d \
    --name=readyset \
    --publish=5433:5433 \
    305232526136.dkr.ecr.us-east-2.amazonaws.com/readyset-psql:release-653fade90acd7700552cfab368f87aab862135e0 \
    --standalone \
    --deployment='quickstart-postgres' \
    --upstream-db-url=postgresql://postgres:readyset@host.docker.internal:5432/imdb \
    --address=0.0.0.0:5433 \
    --username='postgres' \
    --password='readyset' \
    --query-caching='explicit'
    ```

2. This `docker run` command is similar to the one you earlier. However, the flags following the `readyset-psql` image are specific to ReadySet. Take a moment to understand them:

    Flag | Details
    -----|--------
    `--standalone` | <p>For [production deployments](deploy-readyset-kubernetes.md), you run the ReadySet Server and Adapter as separate processes. For local testing, however, you can run the Server and Adapter as a single process by passing the `--standalone` flag to the ReadySet Adapter command.</p>
    `--deployment` | A unique identifier for the ReadySet deployment.
    `--upstream-db-url` | <p>The URL for connecting ReadySet to Postgres. This connection URL includes the username and password for ReadySet to authenticate with as well as the database to replicate.</p><div class="admonition tip"><p class="admonition-title">Tip</p><p>By default, ReadySet replicates all tables in all schemas of the specified database. For this tutorial, that's fine. However, in future deployments, if the queries you want to cache access only a specific schema or specific tables in a schema, or if some tables can't be replicated by ReadySet because they contain [data types](../reference/sql-support/#data-types) that ReadySet does not support, you can narrow the scope of replication by passing `--replication-tables=<schema.table>,<schema.table>`.</p>
    `--address` | The IP and port that ReadySet listens on. For this tutorial, ReadySet is running locally on a different port than Postgres, so connecting `psql` to ReadySet is just a matter of changing the port from `5432` to `5433`.</p>       
    `--username`<br>`--password`| The username and password for connecting clients to ReadySet. For this tutorial, you're using the same username and password for both Postgres and ReadySet.
    `--query-caching` | <p>The query caching mode for ReadySet.</p><p>For this tutorial, you've set this to `explicit`, which means you must run a specific command to have ReadySet cache a query (covered in [Step 4](#step-4-cache-queries)). The other options are `inrequestpath` and `async`. `inrequestpath` caches [supported queries](../reference/sql-support/#query-caching) automatically but blocks queries from returning results until the cache is ready. `async` also caches supported queries automatically but proxies queries to the upstream database until the cache is ready. For most deployments, the `explicit` option is recommended, as it gives you the most flexibility and control.</p>

3. When ReadySet first connects to the database, it takes an initial snapshot of the tables in your database. This snapshot is important because ReadySet can only cache queries that access snapshotted tables.

    Watch as ReadySet completes the snapshot:

    ``` sh
    docker logs readyset | grep 'Replicating table'
    ```

    ``` {.text .no-copy}
    2022-11-11T19:37:49.467850Z  INFO Replicating table{table=`public`.`title_ratings`}: replicators::postgres_connector::snapshot: Replicating table
    2022-11-11T19:37:49.621510Z  INFO Replicating table{table=`public`.`title_ratings`}: replicators::postgres_connector::snapshot: Snapshotting started rows=1246402
    2022-11-11T19:38:10.379432Z  INFO Replicating table{table=`public`.`title_ratings`}: replicators::postgres_connector::snapshot: Snapshotting finished rows_replicated=1246402
    2022-11-11T19:38:10.380743Z  INFO Replicating table{table=`public`.`title_basics`}: replicators::postgres_connector::snapshot: Replicating table
    2022-11-11T19:38:10.806728Z  INFO Replicating table{table=`public`.`title_basics`}: replicators::postgres_connector::snapshot: Snapshotting started rows=5159701
    2022-11-11T19:38:41.816038Z  INFO Replicating table{table=`public`.`title_basics`}: replicators::postgres_connector::snapshot: Snapshotting progress rows_replicated=1085440 progress=21.04% estimate=00:01:56
    2022-11-11T19:39:12.831545Z  INFO Replicating table{table=`public`.`title_basics`}: replicators::postgres_connector::snapshot: Snapshotting progress rows_replicated=2170880 progress=42.07% estimate=00:01:25
    2022-11-11T19:39:43.869316Z  INFO Replicating table{table=`public`.`title_basics`}: replicators::postgres_connector::snapshot: Snapshotting progress rows_replicated=3259392 progress=63.17% estimate=00:00:54
    2022-11-11T19:40:14.878916Z  INFO Replicating table{table=`public`.`title_basics`}: replicators::postgres_connector::snapshot: Snapshotting progress rows_replicated=4340736 progress=84.13% estimate=00:00:23
    2022-11-11T19:40:38.566688Z  INFO Replicating table{table=`public`.`title_basics`}: replicators::postgres_connector::snapshot: Snapshotting finished rows_replicated=5159701
    ```

    This will take a few minutes. As ReadySet snapshots a table, you'll see its progress and the estimate time remaining in the log messages (e.g., `progress=84.13% estimate=00:00:23`). Don't continue to the next step until you see `Snapshotting finished` for both `title_ratings` and `title_basics`.

## Step 4. Cache queries

With snapshotting finished, ReadySet is ready for caching, so in this step, you'll run some queries, check if ReadySet supports them, and then cache them.   

1. Get back into the container with the Postgres client and connect it to ReadySet instead of the database:

    ``` sh
    docker exec -it psql bash
    ```

    ``` sh
    PGPASSWORD=readyset psql \
    --host=host.docker.internal \
    --port=5433 \
    --username=postgres \
    --dbname=imdb
    ```

2. Run a query that joins results from `title_ratings` and `title_basics` to count how many titles released in 2000 have an average rating higher than 5:

    ``` sql
    SELECT count(*) FROM title_ratings
    JOIN title_basics ON title_ratings.tconst = title_basics.tconst
    WHERE title_basics.startyear = 2000 AND title_ratings.averagerating > 5;
    ```

    ``` {.text .no-copy}
      count
     -------
      14144
     (1 row)
    ```

3. Because the query is not yet cached, ReadySet proxied it to the upstream database. Use ReadySet's custom [`SHOW PROXIED QUERIES`](../cache-queries/#identify-queries-to-cache) command to check if ReadySet can cache the query:

    ``` sql
    SHOW PROXIED QUERIES;
    ```

    ```
          query id      |                                                                                            proxied query                                                                                             | readyset supported
    --------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------
     q_df958c381703f5d4 | SELECT count(*) FROM `title_ratings` JOIN `title_basics` ON (`title_ratings`.`tconst` = `title_basics`.`tconst`) WHERE ((`title_basics`.`startyear` = $1) AND (`title_ratings`.`averagerating` > 5)) | yes
    (1 row)
    ```

    You'll see `yes` under `readyset supported`.

    !!! note

        To successfully cache the results of a query, ReadySet must support the SQL features and syntax in the query. For more details, see [SQL Support](../reference/sql-support/#query-caching).

4. Cache the query in ReadySet:

    ``` sql
    CREATE CACHE FROM -- (1)
    SELECT count(*) FROM title_ratings
    JOIN title_basics ON title_ratings.tconst = title_basics.tconst
    WHERE title_basics.startyear = 2000 AND title_ratings.averagerating > 5;
    ```

    1.   To cache a query, you can provide either the full `SELECT` (as shown here) or the query ID listed in the `SHOW PROXIED QUERIES` output.

    The `CREATE CACHE` command constructs the initial dataflow graph for the query and adds indexes to the relevant ReadySet table snapshots, as necessary. The command will return once this is complete.            

5. Run a second query, this time joining results from your two tables to get the title and average rating of the 10 top-rated movies from 1950:

    ``` sql
    SELECT title_basics.originaltitle, title_ratings.averagerating
    FROM title_basics
    JOIN title_ratings ON title_basics.tconst = title_ratings.tconst
    WHERE title_basics.startyear = 1950 AND title_basics.titletype = 'movie'
    ORDER BY title_ratings.averagerating DESC
    LIMIT 10;
    ```

    ``` {.text .no-copy}
              originaltitle              | averagerating
    -------------------------------------+---------------
    Le mariage de Mademoiselle Beulemans |           9.0
    Es kommt ein Tag                     |           8.7
    Nili                                 |           8.7
    Sudhar Prem                          |           8.7
    Pyar                                 |           8.6
    Jiruba Tetsu                         |           8.5
    Meena Bazaar                         |           8.5
    Pardes                               |           8.4
    Showkar                              |           8.4
    Siete muertes a plazo fijo           |           8.4
    (10 rows)
    ```

6. Use the [`SHOW PROXIED QUERIES`](../cache-queries/#identify-queries-to-cache) command to check if ReadySet can cache the query:

    ``` sql
    SHOW PROXIED QUERIES;
    ```

    ``` {.text .no-copy}
          query id      |                                                                                                                                             proxied query                                                                                                                                             | readyset supported
    --------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------
     q_21bee9ced453311c | SELECT `title_basics`.`originaltitle`, `title_ratings`.`averagerating` FROM `title_basics` JOIN `title_ratings` ON (`title_basics`.`tconst` = `title_ratings`.`tconst`) WHERE ((`title_basics`.`startyear` = $1) AND (`title_basics`.`titletype` = $2)) ORDER BY `title_ratings`.`averagerating` DESC | yes
    (1 row)
    ```

    You'll see `yes` under `readyset supported`.

7. Cache the query in ReadySet:

    ``` sql
    CREATE CACHE FROM
    SELECT title_basics.originaltitle, title_ratings.averagerating
    FROM title_basics
    JOIN title_ratings ON title_basics.tconst = title_ratings.tconst
    WHERE title_basics.startyear = 1950 AND title_basics.titletype = 'movie'
    ORDER BY title_ratings.averagerating DESC
    LIMIT 10;
    ```

8. Use ReadySet's custom [`SHOW CACHES`](../cache-queries/#cache-queries_1) command to verify that caches have been created for your two queries:

    ``` sql
    SHOW CACHES;
    ```

    ``` {.text .no-copy}
            name         |                                                                                                                                                                                         query                                                                                                                                                                                          | fallback behavior
    ----------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------
    `q_21bee9ced453311c` | SELECT `public`.`title_basics`.`originaltitle`, `public`.`title_ratings`.`averagerating` FROM `public`.`title_basics` JOIN `public`.`title_ratings` ON (`public`.`title_basics`.`tconst` = `public`.`title_ratings`.`tconst`) WHERE ((`public`.`title_basics`.`startyear` = $1) AND (`public`.`title_basics`.`titletype` = $2)) ORDER BY `public`.`title_ratings`.`averagerating` DESC | fallback allowed
    `q_df958c381703f5d4` | SELECT count(coalesce(`public`.`title_ratings`.`tconst`, '<anonymized>')) FROM `public`.`title_ratings` JOIN `public`.`title_basics` ON (`public`.`title_ratings`.`tconst` = `public`.`title_basics`.`tconst`) WHERE ((`public`.`title_basics`.`startyear` = $1) AND (`public`.`title_ratings`.`averagerating` > '<anonymized>'))                                                      | fallback allowed
    (2 rows)        
    ```

9. Exist the Postgres client:

    ``` sql
    \q
    ```

10. Exit the container:

    ``` sh
    exit
    ```

## Step 5. Check latencies

In this step, you'll use a simple Python application to run your queries against both the database and ReadySet so you can compare how fast results are returned.

1. If you don't already have it, install the [`pyscopg2` Postgres driver](https://www.psycopg.org/docs/install.html):

    ``` sh
    pip install psycopg2-binary
    ```

2. Download the Python app:

    ``` sh
    curl -O https://raw.githubusercontent.com/readysettech/docs/main/docs/assets/quickstart-app.py
    ```

3. Run the first `JOIN` query against the database:

    ``` sh
    python3 quickstart-app.py \
    --url="postgresql://postgres:readyset@127.0.0.1:5432/imdb?sslmode=disable" \
    --query="SELECT count(*) FROM title_ratings JOIN title_basics ON title_ratings.tconst = title_basics.tconst WHERE title_basics.startyear = 2000 AND title_ratings.averagerating > 5;"
    ```

    ``` text hl_lines="9"
    Result:
    ['count']
    ['14144']

    Query latencies (in milliseconds):
    [576.509952545166, 256.64591789245605, 248.0618953704834, 268.12100410461426, 274.6622562408447, 249.19509887695312, 242.0189380645752, 244.05193328857422, 251.70207023620605, 243.8950538635254, 242.94090270996094, 243.37410926818848, 256.84309005737305, 242.22111701965332, 243.90602111816406, 244.5220947265625, 248.14796447753906, 242.26880073547363, 243.3609962463379, 242.77997016906738]

    Median query latency (in milliseconds):
    244.28701400756836
    ```

    The Python app runs the query 20 times and prints the latency of each iteration as well as the median query latency. Note the median latency when results are returned from the database.

4. Run the same `JOIN` again, but this time against ReadySet:

    !!! tip

        Changing your connection string is the only change you make to your application. In this case, you're just changing the port from `5432` for Postgres to `5433` for ReadySet.

    ``` sh
    python3 quickstart-app.py \
    --url="postgresql://postgres:readyset@127.0.0.1:5433/imdb?sslmode=disable" \
    --query="SELECT count(*) FROM title_ratings JOIN title_basics ON title_ratings.tconst = title_basics.tconst WHERE title_basics.startyear = 2000 AND title_ratings.averagerating > 5;"
    ```

    ``` text hl_lines="9"
    Result:
    ['count(coalesce(`public`.`title_ratings`.`tconst`, 0))']
    ['14144']

    Query latencies (in milliseconds):
    [7.894039154052734, 1.7399787902832031, 1.2347698211669922, 1.0900497436523438, 1.1479854583740234, 0.9770393371582031, 0.9369850158691406, 1.046895980834961, 0.78582763671875, 1.0271072387695312, 0.9579658508300781, 1.0287761688232422, 0.9453296661376953, 1.0707378387451172, 0.8528232574462891, 1.0328292846679688, 1.0128021240234375, 0.9450912475585938, 0.9052753448486328, 1.0328292846679688]

    Median query latency (in milliseconds):
    1.0279417037963867
    ```

    As you can see, ReadySet returns results much faster. In the example here, latency went from 244ms to 1ms.

5. Now run the second `JOIN` query against the database:

    ``` sh
    python3 quickstart-app.py \
    --url="postgresql://postgres:readyset@127.0.0.1:5432/imdb?sslmode=disable" \
    --query="SELECT title_basics.originaltitle, title_ratings.averagerating FROM title_basics JOIN title_ratings ON title_basics.tconst = title_ratings.tconst WHERE title_basics.startyear = 1950 AND title_basics.titletype = 'movie' ORDER BY title_ratings.averagerating DESC LIMIT 10;"
    ```

    ``` text hl_lines="18"
    Result:
    ['originaltitle', 'averagerating']
    ['Le mariage de Mademoiselle Beulemans', '9.0']
    ['Nili', '8.7']
    ['Es kommt ein Tag', '8.7']
    ['Sudhar Prem', '8.7']
    ['Pyar', '8.6']
    ['Meena Bazaar', '8.5']
    ['Jiruba Tetsu', '8.5']
    ['Vidyasagar', '8.4']
    ['Sunset Blvd.', '8.4']
    ['Tathapi', '8.4']

    Query latencies (in milliseconds):
    [442.4419403076172, 205.0178050994873, 180.39894104003906, 180.38487434387207, 181.5941333770752, 180.16314506530762, 191.8940544128418, 182.40904808044434, 179.16297912597656, 189.73207473754883, 194.08202171325684, 197.3593235015869, 184.1599941253662, 181.69879913330078, 178.76791954040527, 178.4951686859131, 177.46591567993164, 179.26788330078125, 177.91295051574707, 178.87020111083984]

    Median query latency (in milliseconds):
    180.99653720855713
    ```

    Note the median latency when results are returned from the database.

4. Run the same `JOIN` again, but this time against ReadySet:

    ``` sh
    python3 quickstart-app.py \
    --url="postgresql://postgres:readyset@127.0.0.1:5433/imdb?sslmode=disable" \
    --query="SELECT title_basics.originaltitle, title_ratings.averagerating FROM title_basics JOIN title_ratings ON title_basics.tconst = title_ratings.tconst WHERE title_basics.startyear = 1950 AND title_basics.titletype = 'movie' ORDER BY title_ratings.averagerating DESC LIMIT 10;"
    ```

    ``` text hl_lines="18"
    Result:
    ['originaltitle', 'averagerating']
    ['Le mariage de Mademoiselle Beulemans', '9.0']
    ['Es kommt ein Tag', '8.7']
    ['Nili', '8.7']
    ['Sudhar Prem', '8.7']
    ['Pyar', '8.6']
    ['Jiruba Tetsu', '8.5']
    ['Meena Bazaar', '8.5']
    ['Pardes', '8.4']
    ['Showkar', '8.4']
    ['Siete muertes a plazo fijo', '8.4']

    Query latencies (in milliseconds):
    [10.348796844482422, 1.4729499816894531, 1.4388561248779297, 1.1410713195800781, 1.0881423950195312, 1.1289119720458984, 1.026153564453125, 0.9410381317138672, 0.9829998016357422, 1.1851787567138672, 1.5919208526611328, 1.0099411010742188, 1.068115234375, 1.1279582977294922, 1.199960708618164, 1.0521411895751953, 1.0619163513183594, 1.2021064758300781, 1.4808177947998047, 1.2123584747314453]

    Median query latency (in milliseconds):
    1.1349916458129883
    ```

    Again, ReadySet returns results much faster. In this case, latency went from 181ms to 1ms.

## Step 6. Cause a cache refresh

One of ReadySet's most important features is its ability to keep your cache up-to-date as writes are applied to the upstream database. In this step, you'll see this in action.

1. Get back into the container with the Postgres client:

    ``` sh
    docker exec -it psql bash
    ```

2. Insert new rows that will change the count returned by your first `JOIN` query:

    ``` sh
    PGPASSWORD=readyset psql \
    --host=host.docker.internal \
    --port=5432 \
    --username=postgres \
    --dbname=imdb \
    -c "INSERT INTO title_basics (tconst, titletype, primarytitle, originaltitle, isadult, startyear, runtimeminutes, genres)
          VALUES ('tt9999998', 'movie', 'The ReadySet movie', 'The ReadySet movie', false, 2000, 0, 'Adventure');
        INSERT INTO title_ratings (tconst, averagerating, numvotes)
          VALUES ('tt9999998', 10, 1000000);"
    ```

3. Exit the container:

    ``` sh
    exit
    ```

4. Run the `JOIN` against ReadySet again:

    ``` sh
    python3 quickstart-app.py \
    --url="postgresql://postgres:readyset@127.0.0.1:5433/imdb?sslmode=disable" \
    --query="SELECT count(*) FROM title_ratings JOIN title_basics ON title_ratings.tconst = title_basics.tconst WHERE title_basics.startyear = 2000 AND title_ratings.averagerating > 5;"
    ```

    ``` text hl_lines="3"
    Result:
    ['count(coalesce(`public`.`title_ratings`.`tconst`, 0))']
    ['14145']

    Query latencies (in milliseconds):
    [7.816076278686523, 1.3060569763183594, 1.0461807250976562, 1.0340213775634766, 1.085042953491211, 1.2619495391845703, 1.065969467163086, 0.965118408203125, 1.055002212524414, 1.0008811950683594, 1.0418891906738281, 1.0440349578857422, 1.0480880737304688, 0.9131431579589844, 1.1019706726074219, 1.0309219360351562, 0.8969306945800781, 0.9522438049316406, 1.146078109741211, 1.0101795196533203]

    Median query latency (in milliseconds):
    1.0451078414916992
    ```

    !!! success

        Previously, the count was 14144. Now, the count is 14145. This shows how ReadySet automatically updates your cache, using the database's replication stream, with no action needed on your part to keep the database and cache in sync.

## Step 7. Tear down

When you are done testing your local deployment, use the `docker stop` and `docker rm` commands to stop and remove the containers and volumes for ReadySet, Postgres, and the Postgres client:

``` sh
docker stop readyset postgres psql
```

``` sh
docker rm -v readyset postgres psql
```
