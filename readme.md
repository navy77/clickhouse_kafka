
## note
connect http:8123 , tcp:9000

## command
docker compose up -d\
clickhouse-client \
show databases; \
create database demo_db;

## replication

CREATE DATABASE my_db ON CLUSTER replicated_cluster;

SHOW DATABASES;

### create table
CREATE TABLE my_db.rep_table ON CLUSTER replicated_cluster (\
 user_id UInt32,\
 message String,\
 timestamp DateTime,\
 metric Float32
) 
ENGINE = ReplicatedMergeTree('/clickhouse/tables/rep_table/{shard}', '{replica}')\
PRIMARY KEY (user_id, timestamp);

### create distributed table

CREATE TABLE my_db.distributed_table ON CLUSTER replicated_cluster\
 AS my_db.rep_table \
 ENGINE = Distributed (replicated_cluster, my_db, rep_table, rand());

 ### test insert
 INSERT INTO my_db.distributed_table (user_id, message, timestamp, metric) VALUES\
 (101, 'Hello, Clickhouse!', now(), -1.0),\
 (102, 'Insert a lot of rows per batch', yesterday(), 1.41421),\
 (102, 'Sort your data based on your commonly-used queries', today(), 2.718),\
 (101, 'Granules are the smallest chunks of data read', now() + 5, 3.14159);

 ### show data
 SELECT * FROM my_db.distributed_table;

 ## kafka integrate

 kafka --> pull kafka table --> pull MV --> pull storage table
### create cluster
CREATE DATABASE iot_db ON CLUSTER replicated_cluster;
### consume kafka data 
CREATE TABLE  iot_db.datamicdemo1_raw ON CLUSTER replicated_cluster (\
    topic String,\
    lot String,\
    data1 UInt32,\
    data2 UInt32,\
    data3 UInt32,\
    data4 UInt32,\
    data5 UInt32,\
    data6 UInt32,\
    data7 UInt32,\
    data8 UInt32,\
    data9 UInt32,\
    data10 UInt32\
)\
ENGINE = Kafka\
SETTINGS\
    kafka_broker_list = 'localhost:29092,localhost:39092,localhost:49092',\
    kafka_topic_list = 'k_datamicdemo1',\
    kafka_group_name = 'ck_group1',\
    kafka_format = 'JSONEachRow',\
    kafka_num_consumers = 1;

### consume kafka data 

CREATE TABLE  iot_db.datamicdemo1 ON CLUSTER replicated_cluster(\
    event_time DateTime('Asia/Bangkok') DEFAULT now(),\
    topic String,\
    lot String,\
    data1 UInt32,\
    data2 UInt32,\
    data3 UInt32,\
    data4 UInt32,\
    data5 UInt32,\
    data6 UInt32,\
    data7 UInt32,\
    data8 UInt32,\
    data9 UInt32,\
    data10 UInt32\
)\
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/iot_db/datamicdemo1',/
    '{replica}'/                                        
)
ORDER BY event_time;

### Materialized View
CREATE MATERIALIZED VIEW iot_db.mv_kafka_ingest ON CLUSTER replicated_cluster\
TO iot_db.datamicdemo1\
AS\
SELECT * FROM datamicdemo1_raw;

### show data
SELECT * FROM iot_db.datamicdemo1;
