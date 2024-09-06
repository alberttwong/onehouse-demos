## A Demo using Docker containers

Let's use a real world example to see how Hudi works end to end. For this purpose, a self contained
data infrastructure is brought up in a local Docker cluster within your computer. 

The steps have been tested on a Mac laptop

### Prerequisites

  * Docker Setup
    * Virtual disk limit: 200 GB
    * Memory limit: 8 GB
    * File Sharing: VirtioFS
    * [If using Mac ARM], Use Rosetta for x86_64/amd64 emulation on Apple Silicon: CHECKED
  * ngrok
    * Sign up for an account with ngrok, it will be used so that you can access your kafka cluster on the internet.  You will need an ngrok authtoken in the following steps.
  * Internet connectivity
    * Maven repositories like https://mvnrepository.com/
    * Docker Hub
    * Others
    
Also, this has not been tested on some environments like Docker on Windows.


## Setting up Docker Cluster

### Bringing up Demo Cluster

This should pull the Docker images from Docker hub and setup the Docker cluster.

```
export NGROK_AUTHTOKEN=XXXXXX
docker compose up
```

At this point, the Docker cluster will be up and running. The demo cluster brings up the following services

   * Min.IO for Object Store
   * Spark Master and Worker
   * Hive Services (Metastore along with PostgresDB)
   * Apache Kafka with ngrok enabled
   * Containers for Presto setup (Presto coordinator and worker)
   * Containers for Trino setup (Trino coordinator and worker)

```output
albert@Alberts-MBP ~ % docker ps
CONTAINER ID   IMAGE                                    COMMAND                   CREATED          STATUS                            PORTS                                                                    NAMES
52a488224df2   quay.io/debezium/kafka:2.7.0.Final       "/bin/sh -c 'echo \"W…"   10 seconds ago   Up 9 seconds                      9092/tcp, 0.0.0.0:29092->29092/tcp                                       trino-prestodb-spark-minio-kafka-1
cc4e39344d20   starburstdata/hive:3.1.3-e.10            "/bin/sh -c \"/opt/bi…"   10 seconds ago   Up 9 seconds (health: starting)   0.0.0.0:9083->9083/tcp                                                   trino-prestodb-spark-minio-hive-metastore-1
2226b7caf902   minio/mc                                 "/bin/sh -c ' until …"    10 seconds ago   Up 9 seconds                                                                                               trino-prestodb-spark-minio-mc-1
d52de8d6044d   minio/minio                              "/usr/bin/docker-ent…"    10 seconds ago   Up 9 seconds                      0.0.0.0:9000-9001->9000-9001/tcp                                         trino-prestodb-spark-minio-minio-1
f3d2c4c32ab8   quay.io/debezium/zookeeper:2.7.0.Final   "/docker-entrypoint.…"    10 seconds ago   Up 9 seconds                      0.0.0.0:2181->2181/tcp, 0.0.0.0:2888->2888/tcp, 0.0.0.0:3888->3888/tcp   trino-prestodb-spark-minio-zookeeper-1
d0e5b1a7387e   trinodb/trino:418                        "/usr/lib/trino/bin/…"    10 seconds ago   Up 9 seconds (health: starting)   0.0.0.0:8080->8080/tcp                                                   trino
c8d99b87cc07   almondsh/almond:latest                   "tini -g -- start-no…"    10 seconds ago   Up 9 seconds (healthy)            0.0.0.0:8888->8888/tcp                                                   jupyter
b695f12f9d68   ngrok/ngrok:latest                       "/nix/store/n98vsmwd…"    10 seconds ago   Up 9 seconds                      0.0.0.0:4040->4040/tcp                                                   trino-prestodb-spark-minio-ngrok-1
0062dc427617   postgres:11                              "docker-entrypoint.s…"    10 seconds ago   Up 9 seconds                      5432/tcp                                                                 trino-prestodb-spark-minio-metastore_db-1
5bd39a488508   prestodb/presto:0.283                    "/opt/entrypoint.sh"      10 seconds ago   Up 9 seconds                      0.0.0.0:8082->8082/tcp                                                   presto
```

## Demo

Stock Tracker data will be used to showcase different Hudi query types and the effects of Compaction.

Take a look at the directory `demo/data`. There are 2 batches of stock data - each at 1 minute granularity.
The first batch contains stocker tracker data for some stock symbols during the first hour of trading window
(9:30 a.m to 10:30 a.m). The second batch contains tracker data for next 30 mins (10:30 - 11 a.m). Hudi will
be used to ingest these batches to a table which will contain the latest stock tracker data at hour level granularity.
The batches are windowed intentionally so that the second batch contains updates to some of the rows in the first batch.

### Step 1 : Publish the first batch to Kafka

Upload the first batch to Kafka topic 'stock ticks' 
```java
docker exec -it spark /bin/bash
cat /opt/demo/data/batch_1.json | kafkacat -b kafka:9092 -t stock_ticks -P
```

To check if the new topic shows up, use
```java
kafkacat -b kafka -L -J | jq .
{
  "originating_broker": {
    "id": -1,
    "name": "kafka:9092/bootstrap"
  },
  "query": {
    "topic": "*"
  },
  "controllerid": 1,
  "brokers": [
    {
      "id": 1,
      "name": "6.tcp.us-cal-1.ngrok.io:17553"
    }
  ],
  "topics": [
    {
      "topic": "stock_ticks",
      "partitions": [
        {
          "partition": 0,
          "leader": 1,
          "replicas": [
            {
              "id": 1
            }
          ],
          "isrs": [
            {
              "id": 1
            }
          ]
        }
      ]
    }
  ]
}
```

### Step 2: Incrementally ingest data from Kafka topic

Hudi comes with a tool named Hudi Streamer. This tool can connect to variety of data sources (including Kafka) to
pull changes and apply to Hudi table using upsert/insert primitives. Here, we will use the tool to download
json data from kafka topic and ingest to both COW and MOR tables we initialized in the previous step. This tool
automatically initializes the tables in the file-system if they do not exist yet.

```java
docker exec -it spark /bin/bash

# Run the following spark-submit command to execute the Hudi Streamer and ingest to stock_ticks_cow table in S3
spark-submit \
  --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
  --class org.apache.hudi.utilities.streamer.HoodieStreamer org.apache.hudi_hudi-utilities-slim-bundle_2.12-0.15.0.jar \
  --table-type COPY_ON_WRITE \
  --source-class org.apache.hudi.utilities.sources.JsonKafkaSource \
  --source-ordering-field ts  \
  --target-base-path s3a://warehouse/stock_ticks_cow \
  --target-table stock_ticks_cow \
  --props file:///opt/demo/config/kafka-source.properties \
  --schemaprovider-class org.apache.hudi.utilities.schema.FilebasedSchemaProvider

# Run the following spark-submit command to execute the Hudi Streamer and ingest to stock_ticks_mor table in S3
  spark-submit \
  --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
  --class org.apache.hudi.utilities.streamer.HoodieStreamer org.apache.hudi_hudi-utilities-slim-bundle_2.12-0.15.0.jar \
  --table-type MERGE_ON_READ \
  --source-class org.apache.hudi.utilities.sources.JsonKafkaSource \
  --source-ordering-field ts \
  --target-base-path s3a://warehouse/stock_ticks_mor \
  --target-table stock_ticks_mor \
  --props file:///opt/demo/config/kafka-source.properties \
  --schemaprovider-class org.apache.hudi.utilities.schema.FilebasedSchemaProvider \
  --disable-compaction

# The configs contain mostly Kafa connectivity settings, the avro-schema to be used for ingesting along with key and partitioning fields.

exit
```

You can use Min.IO browser to look at the tables
`http://localhost:9001/browser/warehouse/stock_ticks_cow%2F` with username/password of admin/password.

You can explore the new partition folder created in the table along with a "commit" / "deltacommit"
file under .hoodie which signals a successful commit.

There will be a similar setup when you browse the MOR table
`http://localhost:9001/browser/warehouse/stock_ticks_mor%2F` with username/password of admin/password.


### Step 3: Sync with Hive

At this step, the tables are available in S3. We need to sync with Hive to create new Hive tables and add partitions
inorder to run Hive queries against those tables.

```java
docker exec -it openjdk8 /bin/bash

export HUDI_CLASSPATH=/opt/hudisync/*

# If needed, we need to modify the existing run_sync_tool.sh with additional classpaths `/opt/hudisync/*:`.  Save and exit.
vi /opt/hudi/hudi-sync/hudi-hive-sync/run_sync_tool.sh

# The new java launch should look like
echo "Running Command : java -cp /opt/hudisync/*:${HADOOP_HIVE_JARS}:${HADOOP_CONF_DIR}:$HUDI_HIVE_UBER_JAR org.apache.hudi.hive.HiveSyncTool $@"
java -cp /opt/hudisync/*:$HUDI_HIVE_UBER_JAR:${HADOOP_HIVE_JARS}:${HADOOP_CONF_DIR} org.apache.hudi.hive.HiveSyncTool "$@"

# This command takes in HiveServer URL and COW Hudi table location in S3 and sync the S3 state to Hive
/opt/hudi/hudi-sync/hudi-hive-sync/run_sync_tool.sh  \
--metastore-uris 'thrift://hive-metastore:9083' \
--partitioned-by dt \
--base-path 's3a://warehouse/stock_ticks_cow' \
--database default \
--table stock_ticks_cow \
--sync-mode hms \
--partition-value-extractor org.apache.hudi.hive.SlashEncodedDayPartitionValueExtractor
.....
2024-09-04 12:33:27,101 INFO  [main] hive.HiveSyncTool (HiveSyncTool.java:syncHoodieTable(297)) - Sync complete for stock_ticks_cow
.....

# Now run hive-sync for the second data-set in S3 using Merge-On-Read (MOR table type)
/opt/hudi/hudi-sync/hudi-hive-sync/run_sync_tool.sh  \
--metastore-uris 'thrift://hive-metastore:9083' \
--partitioned-by dt \
--base-path 's3a://warehouse/stock_ticks_mor' \
--database default \
--table stock_ticks_mor \
--sync-mode hms \
--partition-value-extractor org.apache.hudi.hive.SlashEncodedDayPartitionValueExtractor
.....
2024-09-04 12:34:16,413 INFO  [main] hive.HiveSyncTool (HiveSyncTool.java:syncHoodieTable(297)) - Sync complete for stock_ticks_mor
.....

exit
```
After executing the above command, you will notice

1. A hive table named `stock_ticks_cow` created which supports Snapshot and Incremental queries on Copy On Write table.
2. Two new tables `stock_ticks_mor_rt` and `stock_ticks_mor_ro` created for the Merge On Read table. The former
supports Snapshot and Incremental queries (providing near-real time data) while the later supports ReadOptimized queries.


### Step 4 (a): Run Queries with Spark-SQL

Run a \ query to find the latest timestamp ingested for stock symbol 'GOOG'. You will notice that both snapshot 
(for both COW and MOR _rt table) and read-optimized queries (for MOR _ro table) give the same value "10:29 a.m" as Hudi create a
parquet file for the first batch of data.

```java
docker exec -it spark /bin/bash

spark-sql --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
--conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
--conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
--conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' \
--conf 'spark.kryo.registrator=org.apache.spark.HoodieSparkKryoRegistrar'

# List Tables
spark-sql (default)> show tables;
stock_ticks_cow
stock_ticks_mor
stock_ticks_mor_ro
stock_ticks_mor_rt
Time taken: 1.006 seconds, Fetched 4 row(s)


# Look at partitions that were added
spark-sql (default)> show partitions stock_ticks_mor_rt;
2018/08/31
Time taken: 1.191 seconds, Fetched 1 row(s)


# COPY-ON-WRITE Queries:
=========================

spark-sql (default)> select symbol, max(ts) from stock_ticks_cow group by symbol HAVING symbol = 'GOOG';
GOOG	2018-08-31 10:29:00
Time taken: 1.701 seconds, Fetched 1 row(s)

Now, run a projection query:

spark-sql (default)> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_cow where  symbol = 'GOOG';
20240904122742622	GOOG	2018-08-31 09:59:00	6330	1230.5	1230.02
20240904122742622	GOOG	2018-08-31 10:29:00	3391	1230.1899	1230.085
Time taken: 0.149 seconds, Fetched 2 row(s)

# Merge-On-Read Queries:
==========================

Lets run similar queries against M-O-R table. Lets look at both 
ReadOptimized and Snapshot(realtime data) queries supported by M-O-R table

# Run ReadOptimized Query. Notice that the latest timestamp is 10:29
spark-sql (default)> select symbol, max(ts) from stock_ticks_mor_ro group by symbol HAVING symbol = 'GOOG';
GOOG	2018-08-31 10:29:00
Time taken: 0.484 seconds, Fetched 1 row(s)


# Run Snapshot Query. Notice that the latest timestamp is again 10:29

spark-sql (default)> select symbol, max(ts) from stock_ticks_mor_rt group by symbol HAVING symbol = 'GOOG';
GOOG	2018-08-31 10:29:00
Time taken: 0.558 seconds, Fetched 1 row(s)


# Run Read Optimized and Snapshot project queries

spark-sql (default)> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_ro where  symbol = 'GOOG';
20240904123001395	GOOG	2018-08-31 09:59:00	6330	1230.5	1230.02
20240904123001395	GOOG	2018-08-31 10:29:00	3391	1230.1899	1230.085
Time taken: 0.121 seconds, Fetched 2 row(s)

spark-sql (default)> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_rt where  symbol = 'GOOG';
20240904123001395	GOOG	2018-08-31 09:59:00	6330	1230.5	1230.02
20240904123001395	GOOG	2018-08-31 10:29:00	3391	1230.1899	1230.085
Time taken: 0.132 seconds, Fetched 2 row(s)

spark-sql (default)> exit;

exit
```

### Step 4 (b): Run Queries with Spark-Shell
Hudi support Spark as query processor just like Hive. Here are the same hive queries running in spark-shell

```java
docker exec -it spark /bin/bash

spark-shell --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
--conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
--conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
--conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' \
--conf 'spark.kryo.registrator=org.apache.spark.HoodieSparkKryoRegistrar'

...

Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 3.4.3
      /_/

Using Scala version 2.12.17 (OpenJDK 64-Bit Server VM, Java 11.0.24)
Type in expressions to have them evaluated.
Type :help for more information.

scala> spark.sql("show tables").show(100, false)
+---------+------------------+-----------+
|namespace|tableName         |isTemporary|
+---------+------------------+-----------+
|default  |stock_ticks_cow   |false      |
|default  |stock_ticks_mor   |false      |
|default  |stock_ticks_mor_ro|false      |
|default  |stock_ticks_mor_rt|false      |
+---------+------------------+-----------+

# Copy-On-Write Table

## Run max timestamp query against COW table

scala> spark.sql("select symbol, max(ts) from stock_ticks_cow group by symbol HAVING symbol = 'GOOG'").show(100, false)
+------+-------------------+
|symbol|max(ts)            |
+------+-------------------+
|GOOG  |2018-08-31 10:29:00|
+------+-------------------+


## Projection Query

scala> spark.sql("select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_cow where  symbol = 'GOOG'").show(100, false)
+-------------------+------+-------------------+------+---------+--------+
|_hoodie_commit_time|symbol|ts                 |volume|open     |close   |
+-------------------+------+-------------------+------+---------+--------+
|20240904122742622  |GOOG  |2018-08-31 09:59:00|6330  |1230.5   |1230.02 |
|20240904122742622  |GOOG  |2018-08-31 10:29:00|3391  |1230.1899|1230.085|
+-------------------+------+-------------------+------+---------+--------+

# Merge-On-Read Queries:
==========================

Lets run similar queries against M-O-R table. Lets look at both
ReadOptimized and Snapshot queries supported by M-O-R table

# Run ReadOptimized Query. Notice that the latest timestamp is 10:29
scala> spark.sql("select symbol, max(ts) from stock_ticks_mor_ro group by symbol HAVING symbol = 'GOOG'").show(100, false)
+------+-------------------+
|symbol|max(ts)            |
+------+-------------------+
|GOOG  |2018-08-31 10:29:00|
+------+-------------------+


# Run Snapshot Query. Notice that the latest timestamp is again 10:29

scala> spark.sql("select symbol, max(ts) from stock_ticks_mor_rt group by symbol HAVING symbol = 'GOOG'").show(100, false)
+------+-------------------+
|symbol|max(ts)            |
+------+-------------------+
|GOOG  |2018-08-31 10:29:00|
+------+-------------------+


# Run Read Optimized and Snapshot project queries

scala> spark.sql("select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_ro where  symbol = 'GOOG'").show(100, false)
+-------------------+------+-------------------+------+---------+--------+
|_hoodie_commit_time|symbol|ts                 |volume|open     |close   |
+-------------------+------+-------------------+------+---------+--------+
|20240904123001395  |GOOG  |2018-08-31 09:59:00|6330  |1230.5   |1230.02 |
|20240904123001395  |GOOG  |2018-08-31 10:29:00|3391  |1230.1899|1230.085|
+-------------------+------+-------------------+------+---------+--------+


scala> spark.sql("select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_rt where  symbol = 'GOOG'").show(100, false)
+-------------------+------+-------------------+------+---------+--------+
|_hoodie_commit_time|symbol|ts                 |volume|open     |close   |
+-------------------+------+-------------------+------+---------+--------+
|20240904123001395  |GOOG  |2018-08-31 09:59:00|6330  |1230.5   |1230.02 |
|20240904123001395  |GOOG  |2018-08-31 10:29:00|3391  |1230.1899|1230.085|
+-------------------+------+-------------------+------+---------+--------+

scala> :quit

exit
```

### Step 4 (c): Run Trino Queries

Here are the similar queries with Trino.

```java
docker exec -it trino /bin/bash

trino

trino> show catalogs;
 Catalog
---------
 delta
 hive
 hudi
 iceberg
 system
(5 rows)

Query 20240904_124925_00000_2hhut, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.65 [0 rows, 0B] [0 rows/s, 0B/s]

trino:default> show schemas in hudi;
       Schema
--------------------
 default
 information_schema
(2 rows)

Query 20240904_125410_00005_dyubr, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.17 [2 rows, 35B] [11 rows/s, 203B/s]

trino> use hudi.default;
USE

trino:default> show tables;
       Table
--------------------
 stock_ticks_cow
 stock_ticks_mor
 stock_ticks_mor_ro
 stock_ticks_mor_rt
(4 rows)

Query 20240904_125328_00004_dyubr, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.21 [4 rows, 134B] [19 rows/s, 654B/s]



# COPY-ON-WRITE Queries:
=========================
    
trino:default> select symbol, max(ts) from stock_ticks_cow group by symbol HAVING symbol = 'GOOG';
 symbol |        _col1
--------+---------------------
 GOOG   | 2018-08-31 10:29:00
(1 row)

Query 20240904_125446_00006_dyubr, FINISHED, 1 node
Splits: 33 total, 33 done (100.00%)
2.01 [197 rows, 474KB] [98 rows/s, 236KB/s]

trino:default> select "_hoodie_commit_time", symbol, ts, volume, open, close from stock_ticks_cow where symbol = 'GOOG';
 _hoodie_commit_time | symbol |         ts          | volume |   open    |  close
---------------------+--------+---------------------+--------+-----------+----------
 20240904122742622   | GOOG   | 2018-08-31 09:59:00 |   6330 |    1230.5 |  1230.02
 20240904122742622   | GOOG   | 2018-08-31 10:29:00 |   3391 | 1230.1899 | 1230.085
(2 rows)

Query 20240904_125506_00007_dyubr, FINISHED, 1 node
Splits: 1 total, 1 done (100.00%)
1.08 [197 rows, 481KB] [182 rows/s, 447KB/s]

# Merge-On-Read Queries:
==========================

Lets run similar queries against MOR table.

# Run ReadOptimized Query. Notice that the latest timestamp is 10:29
    
trino:default> select symbol, max(ts) from stock_ticks_mor_ro group by symbol HAVING symbol = 'GOOG';
 symbol |        _col1
--------+---------------------
 GOOG   | 2018-08-31 10:29:00
(1 row)

Query 20240904_125531_00008_dyubr, FINISHED, 1 node
Splits: 33 total, 33 done (100.00%)
0.95 [197 rows, 474KB] [208 rows/s, 501KB/s]

trino:default> select "_hoodie_commit_time", symbol, ts, volume, open, close  from stock_ticks_mor_ro where  symbol = 'GOOG';
 _hoodie_commit_time | symbol |         ts          | volume |   open    |  close
---------------------+--------+---------------------+--------+-----------+----------
 20240904123001395   | GOOG   | 2018-08-31 09:59:00 |   6330 |    1230.5 |  1230.02
 20240904123001395   | GOOG   | 2018-08-31 10:29:00 |   3391 | 1230.1899 | 1230.085
(2 rows)

Query 20240904_125548_00009_dyubr, FINISHED, 1 node
Splits: 1 total, 1 done (100.00%)
0.94 [197 rows, 481KB] [209 rows/s, 512KB/s]

trino:default> exit

exit
```

### Step 5: Upload second batch to Kafka and run Hudi Streamer to ingest

Upload the second batch of data and ingest this batch using Hudi Streamer. As this batch does not bring in any new
partitions, there is no need to run hive-sync

```java
docker exec -it spark /bin/bash

cat /opt/demo/data/batch_2.json | kafkacat -b kafka:9092 -t stock_ticks -P

# Run the following spark-submit command to execute the Hudi Streamer and ingest to stock_ticks_cow table in S3
spark-submit \
  --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
  --class org.apache.hudi.utilities.streamer.HoodieStreamer org.apache.hudi_hudi-utilities-slim-bundle_2.12-0.15.0.jar \
  --table-type COPY_ON_WRITE \
  --source-class org.apache.hudi.utilities.sources.JsonKafkaSource \
  --source-ordering-field ts  \
  --target-base-path s3a://warehouse/stock_ticks_cow \
  --target-table stock_ticks_cow \
  --props file:///opt/demo/config/kafka-source.properties \
  --schemaprovider-class org.apache.hudi.utilities.schema.FilebasedSchemaProvider

# Run the following spark-submit command to execute the Hudi Streamer and ingest to stock_ticks_mor table in S3
  spark-submit \
  --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
  --class org.apache.hudi.utilities.streamer.HoodieStreamer org.apache.hudi_hudi-utilities-slim-bundle_2.12-0.15.0.jar \
  --table-type MERGE_ON_READ \
  --source-class org.apache.hudi.utilities.sources.JsonKafkaSource \
  --source-ordering-field ts \
  --target-base-path s3a://warehouse/stock_ticks_mor \
  --target-table stock_ticks_mor \
  --props file:///opt/demo/config/kafka-source.properties \
  --schemaprovider-class org.apache.hudi.utilities.schema.FilebasedSchemaProvider \
  --disable-compaction

exit
```

With Copy-On-Write table, the second ingestion by Hudi Streamer resulted in a new version of Parquet file getting created.
See `http://localhost:9001/browser/warehouse/stock_ticks_cow%2F2018%2F08%2F31%2F`

With Merge-On-Read table, the second ingestion merely appended the batch to an unmerged delta (log) file.
Take a look at the S3 filesystem to get an idea: `http://localhost:9001/browser/warehouse/stock_ticks_mor%2F2018%2F08%2F31%2F`

### Step 6 (a): Run Queries

With Copy-On-Write table, the Snapshot query immediately sees the changes as part of second batch once the batch
got committed as each ingestion creates newer versions of parquet files.

With Merge-On-Read table, the second ingestion merely appended the batch to an unmerged delta (log) file.
This is the time, when ReadOptimized and Snapshot queries will provide different results. ReadOptimized query will still
return "10:29 am" as it will only read from the Parquet file. Snapshot query will do on-the-fly merge and return
latest committed data which is "10:59 a.m".

```java
docker exec -it spark /bin/bash

spark-sql --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
--conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
--conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
--conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' \
--conf 'spark.kryo.registrator=org.apache.spark.HoodieSparkKryoRegistrar'


# Copy On Write Table:

spark-sql (default)> select symbol, max(ts) from stock_ticks_cow group by symbol HAVING symbol = 'GOOG';
GOOG	2018-08-31 10:59:00
Time taken: 3.263 seconds, Fetched 1 row(s)

spark-sql (default)> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_cow where  symbol = 'GOOG';
20240904122742622	GOOG	2018-08-31 09:59:00	6330	1230.5	1230.02
20240904130113388	GOOG	2018-08-31 10:59:00	9021	1227.1993	1227.215
Time taken: 0.155 seconds, Fetched 2 row(s)

As you can notice, the above queries now reflect the changes that came as part of ingesting second batch.


# Merge On Read Table:

# Read Optimized Query
spark-sql (default)> select symbol, max(ts) from stock_ticks_mor_ro group by symbol HAVING symbol = 'GOOG';
GOOG	2018-08-31 10:29:00
Time taken: 0.452 seconds, Fetched 1 row(s)

spark-sql (default)> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_ro where  symbol = 'GOOG';
20240904123001395	GOOG	2018-08-31 09:59:00	6330	1230.5	1230.02
20240904123001395	GOOG	2018-08-31 10:29:00	3391	1230.1899	1230.085
Time taken: 0.112 seconds, Fetched 2 row(s)

# Snapshot Query
spark-sql (default)> select symbol, max(ts) from stock_ticks_mor_rt group by symbol HAVING symbol = 'GOOG';
GOOG	2018-08-31 10:59:00
Time taken: 0.978 seconds, Fetched 1 row(s)

spark-sql (default)> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_rt where  symbol = 'GOOG';
20240904123001395	GOOG	2018-08-31 09:59:00	6330	1230.5	1230.02
20240904130127262	GOOG	2018-08-31 10:59:00	9021	1227.1993	1227.215
Time taken: 0.215 seconds, Fetched 2 row(s)

spark-sql (default)> exit;

exit
```

### Step 6 (b): Run Spark Shell Queries

Running the same queries in Spark-Shell:

```java
docker exec -it spark /bin/bash

spark-shell --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
--conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
--conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
--conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' \
--conf 'spark.kryo.registrator=org.apache.spark.HoodieSparkKryoRegistrar'

# Copy On Write Table:

scala> spark.sql("select symbol, max(ts) from stock_ticks_cow group by symbol HAVING symbol = 'GOOG'").show(100, false)
+------+-------------------+
|symbol|max(ts)            |
+------+-------------------+
|GOOG  |2018-08-31 10:59:00|
+------+-------------------+


scala> spark.sql("select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_cow where  symbol = 'GOOG'").show(100, false)
+-------------------+------+-------------------+------+---------+--------+
|_hoodie_commit_time|symbol|ts                 |volume|open     |close   |
+-------------------+------+-------------------+------+---------+--------+
|20240904122742622  |GOOG  |2018-08-31 09:59:00|6330  |1230.5   |1230.02 |
|20240904130113388  |GOOG  |2018-08-31 10:59:00|9021  |1227.1993|1227.215|
+-------------------+------+-------------------+------+---------+--------+

As you can notice, the above queries now reflect the changes that came as part of ingesting second batch.


# Merge On Read Table:

# Read Optimized Query
scala> spark.sql("select symbol, max(ts) from stock_ticks_mor_ro group by symbol HAVING symbol = 'GOOG'").show(100, false)
+------+-------------------+
|symbol|max(ts)            |
+------+-------------------+
|GOOG  |2018-08-31 10:29:00|
+------+-------------------+

scala> spark.sql("select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_ro where  symbol = 'GOOG'").show(100, false)
+-------------------+------+-------------------+------+---------+--------+
|_hoodie_commit_time|symbol|ts                 |volume|open     |close   |
+-------------------+------+-------------------+------+---------+--------+
|20240904123001395  |GOOG  |2018-08-31 09:59:00|6330  |1230.5   |1230.02 |
|20240904123001395  |GOOG  |2018-08-31 10:29:00|3391  |1230.1899|1230.085|
+-------------------+------+-------------------+------+---------+--------+


# Snapshot Query
scala> spark.sql("select symbol, max(ts) from stock_ticks_mor_rt group by symbol HAVING symbol = 'GOOG'").show(100, false)
org.apache.hudi.org.openjdk.jol.vm.sa.SASupportException: Sense failed., org.apache.hudi.org.openjdk.jol.vm.sa.SASupportException: Sense failed.]
+------+-------------------+
|symbol|max(ts)            |
+------+-------------------+
|GOOG  |2018-08-31 10:59:00|
+------+-------------------+

scala> spark.sql("select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_rt where  symbol = 'GOOG'").show(100, false)
+-------------------+------+-------------------+------+---------+--------+
|_hoodie_commit_time|symbol|ts                 |volume|open     |close   |
+-------------------+------+-------------------+------+---------+--------+
|20240904123001395  |GOOG  |2018-08-31 09:59:00|6330  |1230.5   |1230.02 |
|20240904130127262  |GOOG  |2018-08-31 10:59:00|9021  |1227.1993|1227.215|
+-------------------+------+-------------------+------+---------+--------+

scala> :quit

exit
```

### Step 6 (c): Run Trino Queries

Running the same queries on Trino for Read-Optimized queries.

```java
docker exec -it trino /bin/bash

trino

trino> use hudi.default;
USE
    
# Copy On Write Table:

trino:default> select symbol, max(ts) from stock_ticks_cow group by symbol HAVING symbol = 'GOOG';
 symbol |        _col1
--------+---------------------
 GOOG   | 2018-08-31 10:59:00
(1 row)

Query 20240904_132409_00014_dyubr, FINISHED, 1 node
Splits: 33 total, 33 done (100.00%)
1.14 [197 rows, 474KB] [173 rows/s, 417KB/s]

trino:default> select "_hoodie_commit_time", symbol, ts, volume, open, close  from stock_ticks_cow where  symbol = 'GOOG';
 _hoodie_commit_time | symbol |         ts          | volume |   open    |  close
---------------------+--------+---------------------+--------+-----------+----------
 20240904122742622   | GOOG   | 2018-08-31 09:59:00 |   6330 |    1230.5 |  1230.02
 20240904130113388   | GOOG   | 2018-08-31 10:59:00 |   9021 | 1227.1993 | 1227.215
(2 rows)

Query 20240904_132423_00015_dyubr, FINISHED, 1 node
Splits: 1 total, 1 done (100.00%)
0.88 [197 rows, 481KB] [223 rows/s, 546KB/s]

As you can notice, the above queries now reflect the changes that came as part of ingesting second batch.

# Merge On Read Table:
# Read Optimized Query
    
trino:default> select symbol, max(ts) from stock_ticks_mor_ro group by symbol HAVING symbol = 'GOOG';
 symbol |        _col1
--------+---------------------
 GOOG   | 2018-08-31 10:29:00
(1 row)

Query 20240904_132439_00016_dyubr, FINISHED, 1 node
Splits: 33 total, 33 done (100.00%)
0.88 [197 rows, 474KB] [223 rows/s, 538KB/s]

trino:default> select "_hoodie_commit_time", symbol, ts, volume, open, close  from stock_ticks_mor_ro where  symbol = 'GOOG';
 _hoodie_commit_time | symbol |         ts          | volume |   open    |  close
---------------------+--------+---------------------+--------+-----------+----------
 20240904123001395   | GOOG   | 2018-08-31 09:59:00 |   6330 |    1230.5 |  1230.02
 20240904123001395   | GOOG   | 2018-08-31 10:29:00 |   3391 | 1230.1899 | 1230.085
(2 rows)

Query 20240904_132451_00017_dyubr, FINISHED, 1 node
Splits: 1 total, 1 done (100.00%)
0.87 [197 rows, 481KB] [225 rows/s, 552KB/s]

trino:default> exit

exit
```

### Step 7 (a): Incremental Query for COPY-ON-WRITE Table

With 2 batches of data ingested, lets showcase the support for incremental queries in Hudi Copy-On-Write tables

Lets take the same projection query example

```java
docker exec -it spark /bin/bash

spark-sql --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
--conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
--conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
--conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' \
--conf 'spark.kryo.registrator=org.apache.spark.HoodieSparkKryoRegistrar'

spark-sql (default)> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_cow where  symbol = 'GOOG';
20240904122742622	GOOG	2018-08-31 09:59:00	6330	1230.5	1230.02
20240904130113388	GOOG	2018-08-31 10:59:00	9021	1227.1993	1227.215
Time taken: 2.913 seconds, Fetched 2 row(s)
```

As you notice from the above queries, there are 2 commits - 20240904122742622 and 20240904130113388 in timeline order.
When you follow the steps, you will be getting different timestamps for commits. Substitute them
in place of the above timestamps.

To show the effects of incremental-query, let us assume that a reader has already seen the changes as part of
ingesting first batch. Now, for the reader to see effect of the second batch, he/she has to keep the start timestamp to
the commit time of the first batch (20240904122742622) and run incremental query

Hudi incremental mode provides efficient scanning for incremental queries by filtering out files that do not have any
candidate rows using hudi-managed metadata.

```java
docker exec -it spark /bin/bash

spark-sql --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
--conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
--conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
--conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' \
--conf 'spark.kryo.registrator=org.apache.spark.HoodieSparkKryoRegistrar'


spark-sql (default)> set hoodie.stock_ticks_cow.consume.mode=INCREMENTAL;
hoodie.stock_ticks_cow.consume.mode	INCREMENTAL
Time taken: 0.042 seconds, Fetched 1 row(s)

spark-sql (default)> set hoodie.stock_ticks_cow.consume.max.commits=3;
hoodie.stock_ticks_cow.consume.max.commits	3
Time taken: 0.028 seconds, Fetched 1 row(s)

spark-sql (default)> set hoodie.stock_ticks_cow.consume.start.timestamp=20240904122742622;
hoodie.stock_ticks_cow.consume.start.timestamp	20240904122742622
Time taken: 0.029 seconds, Fetched 1 row(s)
```

With the above setting, file-ids that do not have any updates from the commit 20240904130113388 is filtered out without scanning.
Here is the incremental query :

```java
spark-sql (default)> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_cow where  symbol = 'GOOG' and `_hoodie_commit_time` > '20240904122742622';
20240904130113388	GOOG	2018-08-31 10:59:00	9021	1227.1993	1227.215
Time taken: 0.199 seconds, Fetched 1 row(s)
```

### Step 7 (b): Incremental Query with Spark Shell:

```java
docker exec -it spark /bin/bash

spark-shell --packages org.apache.hudi:hudi-utilities-slim-bundle_2.12:0.15.0,org.apache.hudi:hudi-spark3.4-bundle_2.12:0.15.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
--conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
--conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
--conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' \
--conf 'spark.kryo.registrator=org.apache.spark.HoodieSparkKryoRegistrar'


Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 3.4.3
      /_/

Using Scala version 2.12.17 (OpenJDK 64-Bit Server VM, Java 11.0.24)
Type in expressions to have them evaluated.
Type :help for more information.

scala> import org.apache.hudi.DataSourceReadOptions
import org.apache.hudi.DataSourceReadOptions

scala> val hoodieIncViewDF =  spark.read.format("org.apache.hudi").option(DataSourceReadOptions.QUERY_TYPE_OPT_KEY, DataSourceReadOptions.QUERY_TYPE_INCREMENTAL_OPT_VAL).option(DataSourceReadOptions.BEGIN_INSTANTTIME_OPT_KEY, "20240904122742622").load("s3a://warehouse/stock_ticks_cow")
24/09/04 13:34:10 WARN MetricsConfig: Cannot locate configuration: tried hadoop-metrics2-s3a-file-system.properties,hadoop-metrics2.properties
24/09/04 13:34:10 WARN DFSPropertiesConfiguration: Cannot find HUDI_CONF_DIR, please set it as the dir of hudi-defaults.conf
hoodieIncViewDF: org.apache.spark.sql.DataFrame = [_hoodie_commit_time: string, _hoodie_commit_seqno: string ... 15 more fields]

scala> hoodieIncViewDF.registerTempTable("stock_ticks_cow_incr_tmp1")
warning: one deprecation (since 2.0.0); for details, enable `:setting -deprecation' or `:replay -deprecation'

scala> spark.sql("select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_cow_incr_tmp1 where  symbol = 'GOOG'").show(100, false);
+-------------------+------+-------------------+------+---------+--------+
|_hoodie_commit_time|symbol|ts                 |volume|open     |close   |
+-------------------+------+-------------------+------+---------+--------+
|20240904130113388  |GOOG  |2018-08-31 10:59:00|9021  |1227.1993|1227.215|
+-------------------+------+-------------------+------+---------+--------+

scala> :quit

exit
```

### Step 8: Schedule and Run Compaction for Merge-On-Read table

Lets schedule and run a compaction to create a new version of columnar  file so that read-optimized readers will see fresher data.
Again, You can use Hudi CLI to manually schedule and run compaction

```java
docker exec -it openjdk8 /bin/bash

export HOODIE_ENV_fs_DOT_s3a_DOT_access_DOT_key=admin
export HOODIE_ENV_fs_DOT_s3a_DOT_secret_DOT_key=password
export HOODIE_ENV_fs_DOT_s3a_DOT_endpoint=http://minio:9000
export HOODIE_ENV_fs_DOT_s3a_DOT_aws_DOT_credentials_DOT_provider=org.apache.hadoop.fs.s3a.SimpleAWSCredentialsProvider
export CLIENT_JAR=/opt/hudicli/hadoop-aws-2.10.2.jar:/opt/hudicli/aws-java-sdk-bundle-1.11.271.jar
export SPARK_BUNDLE_JAR=/opt/hudicli/hudi-spark3.4-bundle_2.12-0.15.0.jar
export CLI_BUNDLE_JAR=/opt/hudicli/hudi-cli-bundle_2.12-0.15.0.jar
cp /opt/hudicli/hadoop-aws-2.10.2.jar /spark/jars
cp /opt/hudicli/aws-java-sdk-bundle-1.11.271.jar /spark/jars
mc alias set minio http://minio:9000 admin password
mc cp /opt/demo/config/schema.avsc minio/warehouse

root@openjdk8:/spark-3.4.3-bin-hadoop3/bin# cd /opt/hudi/packaging/hudi-cli-bundle && /opt/hudi/packaging/hudi-cli-bundle/hudi-cli-with-bundle.sh
DIR is /opt/hudi/packaging/hudi-cli-bundle
Inferring CLI_BUNDLE_JAR path assuming this script is under Hudi repo
Inferring SPARK_BUNDLE_JAR path assuming this script is under Hudi repo
CLI_BUNDLE_JAR: /opt/hudi/packaging/hudi-cli-bundle/target/hudi-cli-bundle_2.12-0.15.0.jar
SPARK_BUNDLE_JAR: /opt/hudi/packaging/hudi-cli-bundle/../hudi-spark-bundle/target/hudi-spark-bundle_2.12-0.15.0.jar
Downloading necessary auxiliary jars for Hudi CLI
--2024-09-04 18:07:34--  https://repo1.maven.org/maven2/org/glassfish/jakarta.el/3.0.3/jakarta.el-3.0.3.jar
Resolving repo1.maven.org (repo1.maven.org)... 199.232.196.209, 199.232.192.209, 2a04:4e42:4c::209, ...
Connecting to repo1.maven.org (repo1.maven.org)|199.232.196.209|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 237826 (232K) [application/java-archive]
Saving to: ‘auxlib/jakarta.el-3.0.3.jar’

jakarta.el-3.0.3.jar                         100%[===========================================================================================>] 232.25K  --.-KB/s    in 0.03s

2024-09-04 18:07:35 (7.35 MB/s) - ‘auxlib/jakarta.el-3.0.3.jar’ saved [237826/237826]

--2024-09-04 18:07:35--  https://repo1.maven.org/maven2/jakarta/el/jakarta.el-api/3.0.3/jakarta.el-api-3.0.3.jar
Resolving repo1.maven.org (repo1.maven.org)... 199.232.196.209, 199.232.192.209, 2a04:4e42:4c::209, ...
Connecting to repo1.maven.org (repo1.maven.org)|199.232.196.209|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 79816 (78K) [application/java-archive]
Saving to: ‘auxlib/jakarta.el-api-3.0.3.jar’

jakarta.el-api-3.0.3.jar                     100%[===========================================================================================>]  77.95K  --.-KB/s    in 0.02s

2024-09-04 18:07:35 (4.47 MB/s) - ‘auxlib/jakarta.el-api-3.0.3.jar’ saved [79816/79816]

Client jar location not set, please set it in conf/hudi-env.sh
Running : java -cp /opt/hudi/packaging/hudi-cli-bundle/conf:/opt/hudi/packaging/hudi-cli-bundle/auxlib/*:/spark/*:/spark/jars/*:/etc/hadoop/conf:/etc/spark/conf:/opt/hudi/packaging/hudi-cli-bundle/target/hudi-cli-bundle_2.12-0.15.0.jar:/opt/hudi/packaging/hudi-cli-bundle/../hudi-spark-bundle/target/hudi-spark-bundle_2.12-0.15.0.jar: -DSPARK_CONF_DIR=/etc/spark/conf -DHADOOP_CONF_DIR=/etc/hadoop/conf org.apache.hudi.cli.Main
Main called
===================================================================
*         ___                          ___                        *
*        /\__\          ___           /\  \           ___         *
*       / /  /         /\__\         /  \  \         /\  \        *
*      / /__/         / /  /        / /\ \  \        \ \  \       *
*     /  \  \ ___    / /  /        / /  \ \__\       /  \__\      *
*    / /\ \  /\__\  / /__/  ___   / /__/ \ |__|     / /\/__/      *
*    \/  \ \/ /  /  \ \  \ /\__\  \ \  \ / /  /  /\/ /  /         *
*         \  /  /    \ \  / /  /   \ \  / /  /   \  /__/          *
*         / /  /      \ \/ /  /     \ \/ /  /     \ \__\          *
*        / /  /        \  /  /       \  /  /       \/__/          *
*        \/__/          \/__/         \/__/    Apache Hudi CLI    *
*                                                                 *
===================================================================
733  [main] INFO  org.apache.hudi.cli.Main [] - Starting Main v0.15.0 using Java 1.8.0_422 on openjdk8 with PID 34 (/opt/hudi/packaging/hudi-cli-bundle/target/hudi-cli-bundle_2.12-0.15.0.jar started by root in /spark-3.4.3-bin-hadoop3/bin)
740  [main] INFO  org.apache.hudi.cli.Main [] - No active profile set, falling back to 1 default profile: "default"
Table command getting loaded
Sep 04, 2024 6:07:36 PM org.jline.utils.Log logr
WARNING: The Parser of class org.springframework.shell.jline.ExtendedDefaultParser does not support the CompletingParsedLine interface. Completion with escaped or quoted words won't work correctly.
1486 [main] INFO  org.apache.hudi.cli.Main [] - Started Main in 0.907 seconds (JVM running for 1.517)

hudi->connect --path s3a://warehouse/stock_ticks_mor
17945 [main] WARN  org.apache.hadoop.util.NativeCodeLoader [] - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
18394 [main] INFO  org.apache.hudi.common.table.HoodieTableMetaClient [] - Loading HoodieTableMetaClient from s3a://warehouse/stock_ticks_mor
18429 [main] INFO  org.apache.hudi.common.table.HoodieTableConfig [] - Loading table properties from s3a://warehouse/stock_ticks_mor/.hoodie/hoodie.properties
18441 [main] INFO  org.apache.hudi.common.table.HoodieTableMetaClient [] - Finished Loading Table of type MERGE_ON_READ(version=1, baseFileFormat=PARQUET) from s3a://warehouse/stock_ticks_mor
Metadata for table stock_ticks_mor loaded

hudi:stock_ticks_mor->compactions show all
42012 [main] INFO  org.apache.hudi.common.table.timeline.HoodieActiveTimeline [] - Loaded instants upto : Option{val=[20240905011929870__deltacommit__COMPLETED__20240905011934536]}
╔═════════════════════════╤═══════╤═══════════════════════════════╗
║ Compaction Instant Time │ State │ Total FileIds to be Compacted ║
╠═════════════════════════╧═══════╧═══════════════════════════════╣
║ (empty)                                                         ║
╚═════════════════════════════════════════════════════════════════╝

# Schedule a compaction. This will use Spark Launcher to schedule compaction
hoodie:stock_ticks_mor->compaction schedule --hoodieConfigs hoodie.compact.inline.max.delta.commits=1
....
Attempted to schedule compaction for 20240905045740967

# Now refresh and check again. You will see that there is a new compaction requested

hudi:stock_ticks_mor->refresh
221115 [main] INFO  org.apache.hudi.common.table.HoodieTableMetaClient [] - Loading HoodieTableMetaClient from s3a://warehouse/stock_ticks_mor
221121 [main] INFO  org.apache.hudi.common.table.HoodieTableConfig [] - Loading table properties from s3a://warehouse/stock_ticks_mor/.hoodie/hoodie.properties
221125 [main] INFO  org.apache.hudi.common.table.HoodieTableMetaClient [] - Finished Loading Table of type MERGE_ON_READ(version=1, baseFileFormat=PARQUET) from s3a://warehouse/stock_ticks_mor
Metadata for table stock_ticks_mor refreshed.

hudi:stock_ticks_mor->compactions show all
264893 [main] INFO  org.apache.hudi.common.table.timeline.HoodieActiveTimeline [] - Loaded instants upto : Option{val=[==>20240905045740967__compaction__REQUESTED__20240905045745721]}
╔═════════════════════════╤═══════════╤═══════════════════════════════╗
║ Compaction Instant Time │ State     │ Total FileIds to be Compacted ║
╠═════════════════════════╪═══════════╪═══════════════════════════════╣
║ 20240905045740967       │ REQUESTED │ 1                             ║
╚═════════════════════════╧═══════════╧═══════════════════════════════╝

# Execute the compaction. The compaction instant value passed below must be the one displayed in the above "compactions show all" query
hoodie:stock_ticks_mor->compaction run --compactionInstant  20240906213724132 --parallelism 2 --sparkMemory 1G  --schemaFilePath s3://warehouse/schema.avsc --retry 1
....
Compaction successfully completed for 20240906213724132

## Now check if compaction is completed

hudi:stock_ticks_mor->refresh
258485 [main] INFO  org.apache.hudi.common.table.HoodieTableMetaClient [] - Loading HoodieTableMetaClient from s3a://warehouse/stock_ticks_mor
258493 [main] INFO  org.apache.hudi.common.table.HoodieTableConfig [] - Loading table properties from s3a://warehouse/stock_ticks_mor/.hoodie/hoodie.properties
258497 [main] INFO  org.apache.hudi.common.table.HoodieTableMetaClient [] - Finished Loading Table of type MERGE_ON_READ(version=1, baseFileFormat=PARQUET) from s3a://warehouse/stock_ticks_mor
Metadata for table stock_ticks_mor refreshed.

hudi:stock_ticks_mor->compactions show all
274612 [main] INFO  org.apache.hudi.common.table.timeline.HoodieActiveTimeline [] - Loaded instants upto : Option{val=[20240906213724132__commit__COMPLETED__20240906215041868]}
╔═════════════════════════╤═══════════╤═══════════════════════════════╗
║ Compaction Instant Time │ State     │ Total FileIds to be Compacted ║
╠═════════════════════════╪═══════════╪═══════════════════════════════╣
║ 20240906213724132       │ COMPLETED │ 1                             ║
╚═════════════════════════╧═══════════╧═══════════════════════════════╝

```

### Step 9: Run Hive Queries including incremental queries

You will see that both ReadOptimized and Snapshot queries will show the latest committed data.
Lets also run the incremental query for MOR table.
From looking at the below query output, it will be clear that the fist commit time for the MOR table is 20180924064636
and the second commit time is 20180924070031

```java
docker exec -it adhoc-2 /bin/bash
beeline -u jdbc:hive2://hiveserver:10000 \
  --hiveconf hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat \
  --hiveconf hive.stats.autogather=false

# Read Optimized Query
0: jdbc:hive2://hiveserver:10000> select symbol, max(ts) from stock_ticks_mor_ro group by symbol HAVING symbol = 'GOOG';
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
+---------+----------------------+--+
| symbol  |         _c1          |
+---------+----------------------+--+
| GOOG    | 2018-08-31 10:59:00  |
+---------+----------------------+--+
1 row selected (1.6 seconds)

0: jdbc:hive2://hiveserver:10000> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_ro where  symbol = 'GOOG';
+----------------------+---------+----------------------+---------+------------+-----------+--+
| _hoodie_commit_time  | symbol  |          ts          | volume  |    open    |   close   |
+----------------------+---------+----------------------+---------+------------+-----------+--+
| 20180924064636       | GOOG    | 2018-08-31 09:59:00  | 6330    | 1230.5     | 1230.02   |
| 20180924070031       | GOOG    | 2018-08-31 10:59:00  | 9021    | 1227.1993  | 1227.215  |
+----------------------+---------+----------------------+---------+------------+-----------+--+

# Snapshot Query
0: jdbc:hive2://hiveserver:10000> select symbol, max(ts) from stock_ticks_mor_rt group by symbol HAVING symbol = 'GOOG';
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
+---------+----------------------+--+
| symbol  |         _c1          |
+---------+----------------------+--+
| GOOG    | 2018-08-31 10:59:00  |
+---------+----------------------+--+

0: jdbc:hive2://hiveserver:10000> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_rt where  symbol = 'GOOG';
+----------------------+---------+----------------------+---------+------------+-----------+--+
| _hoodie_commit_time  | symbol  |          ts          | volume  |    open    |   close   |
+----------------------+---------+----------------------+---------+------------+-----------+--+
| 20180924064636       | GOOG    | 2018-08-31 09:59:00  | 6330    | 1230.5     | 1230.02   |
| 20180924070031       | GOOG    | 2018-08-31 10:59:00  | 9021    | 1227.1993  | 1227.215  |
+----------------------+---------+----------------------+---------+------------+-----------+--+

# Incremental Query:

0: jdbc:hive2://hiveserver:10000> set hoodie.stock_ticks_mor.consume.mode=INCREMENTAL;
No rows affected (0.008 seconds)
# Max-Commits covers both second batch and compaction commit
0: jdbc:hive2://hiveserver:10000> set hoodie.stock_ticks_mor.consume.max.commits=3;
No rows affected (0.007 seconds)
0: jdbc:hive2://hiveserver:10000> set hoodie.stock_ticks_mor.consume.start.timestamp=20180924064636;
No rows affected (0.013 seconds)
# Query:
0: jdbc:hive2://hiveserver:10000> select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_ro where  symbol = 'GOOG' and `_hoodie_commit_time` > '20180924064636';
+----------------------+---------+----------------------+---------+------------+-----------+--+
| _hoodie_commit_time  | symbol  |          ts          | volume  |    open    |   close   |
+----------------------+---------+----------------------+---------+------------+-----------+--+
| 20180924070031       | GOOG    | 2018-08-31 10:59:00  | 9021    | 1227.1993  | 1227.215  |
+----------------------+---------+----------------------+---------+------------+-----------+--+

exit
```

### Step 10: Read Optimized and Snapshot queries for MOR with Spark-SQL after compaction

```java
docker exec -it adhoc-1 /bin/bash
$SPARK_INSTALL/bin/spark-shell \
  --jars $HUDI_SPARK_BUNDLE \
  --driver-class-path $HADOOP_CONF_DIR \
  --conf spark.sql.hive.convertMetastoreParquet=false \
  --deploy-mode client \
  --driver-memory 1G \
  --master local[2] \
  --executor-memory 3G \
  --num-executors 1

# Read Optimized Query
scala> spark.sql("select symbol, max(ts) from stock_ticks_mor_ro group by symbol HAVING symbol = 'GOOG'").show(100, false)
+---------+----------------------+
| symbol  |        max(ts)       |
+---------+----------------------+
| GOOG    | 2018-08-31 10:59:00  |
+---------+----------------------+
1 row selected (1.6 seconds)

scala> spark.sql("select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_ro where  symbol = 'GOOG'").show(100, false)
+----------------------+---------+----------------------+---------+------------+-----------+
| _hoodie_commit_time  | symbol  |          ts          | volume  |    open    |   close   |
+----------------------+---------+----------------------+---------+------------+-----------+
| 20180924064636       | GOOG    | 2018-08-31 09:59:00  | 6330    | 1230.5     | 1230.02   |
| 20180924070031       | GOOG    | 2018-08-31 10:59:00  | 9021    | 1227.1993  | 1227.215  |
+----------------------+---------+----------------------+---------+------------+-----------+

# Snapshot Query
scala> spark.sql("select symbol, max(ts) from stock_ticks_mor_rt group by symbol HAVING symbol = 'GOOG'").show(100, false)
+---------+----------------------+
| symbol  |     max(ts)          |
+---------+----------------------+
| GOOG    | 2018-08-31 10:59:00  |
+---------+----------------------+

scala> spark.sql("select `_hoodie_commit_time`, symbol, ts, volume, open, close  from stock_ticks_mor_rt where  symbol = 'GOOG'").show(100, false)
+----------------------+---------+----------------------+---------+------------+-----------+
| _hoodie_commit_time  | symbol  |          ts          | volume  |    open    |   close   |
+----------------------+---------+----------------------+---------+------------+-----------+
| 20180924064636       | GOOG    | 2018-08-31 09:59:00  | 6330    | 1230.5     | 1230.02   |
| 20180924070031       | GOOG    | 2018-08-31 10:59:00  | 9021    | 1227.1993  | 1227.215  |
+----------------------+---------+----------------------+---------+------------+-----------+
```

### Step 11:  Presto Read Optimized queries on MOR table after compaction
:::note
This section of the demo is not supported for Mac AArch64 users at this time.
:::

```java
docker exec -it presto-worker-1 presto --server presto-coordinator-1:8090
presto> use hive.default;
USE

# Read Optimized Query
resto:default> select symbol, max(ts) from stock_ticks_mor_ro group by symbol HAVING symbol = 'GOOG';
  symbol |        _col1
--------+---------------------
 GOOG   | 2018-08-31 10:59:00
(1 row)

Query 20190822_182319_00011_segyw, FINISHED, 1 node
Splits: 49 total, 49 done (100.00%)
0:01 [197 rows, 613B] [133 rows/s, 414B/s]

presto:default> select "_hoodie_commit_time", symbol, ts, volume, open, close  from stock_ticks_mor_ro where  symbol = 'GOOG';
 _hoodie_commit_time | symbol |         ts          | volume |   open    |  close
---------------------+--------+---------------------+--------+-----------+----------
 20190822180250      | GOOG   | 2018-08-31 09:59:00 |   6330 |    1230.5 |  1230.02
 20190822181944      | GOOG   | 2018-08-31 10:59:00 |   9021 | 1227.1993 | 1227.215
(2 rows)

Query 20190822_182333_00012_segyw, FINISHED, 1 node
Splits: 17 total, 17 done (100.00%)
0:02 [197 rows, 613B] [98 rows/s, 307B/s]

presto:default>
```


This brings the demo to an end.

## Additional Demos

### Apache xTable

You can easily add Apache xTable to this demo.   Just follow the steps in the Apache xTable Quickstart using this docker compose.

### Conduktor

Conduktor is a web based way for you see your kafka environment.   Just use the ngrok kafka URL to connect.   

### Debezium

You can extend this demo to a Database CDC demo by adding a database like postgresSQL and adding the Debezium Kafka Connect container images into this docker compose. 

### Onehouse.ai

You can hook up this environment to Onehouse.ai by using this demo as a kafka source and using the ngrok kafka URL.
