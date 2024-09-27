# Conduit + Trino

![trino-2](https://github.com/user-attachments/assets/a1030810-70eb-4a59-9fc9-4a8f7ceea62a)


Sensors generate data from different sites, delivering it to a centralized stream provider like Kafka.
Each message is encoded using Avro and their schema is available through the schema registry.

Trino discovers the topic based on the schemas available in the registry. Each message inside the topic
can be decoded using the available Avro schema. Topics data is represented as a table.

Alternatively the queried data can be saved in another store like Clickhouse.

## Setup

You need docker-compose and conduit v0.11 to run this.

```
# docker compose up -d
```

Start Conduit when all the docker compose services are running. Conduit will run in the foreground.
The generator plugin will emit about 2000 messages into two kafka topics: sensors1, sensors2

```
# cd conduit/ && conduit
```

## Usage

For this you will need to install Trino locally, by running `brew install trino`.

Start the CLI:
```
trino --server localhost:10080 --output-format-interactive VERTICAL
```

### Show available catalogs and schemas

Display the available catalogs

```
trino> show catalogs;
-[ RECORD 1 ]-------
Catalog | clickhouse
-[ RECORD 2 ]-------
Catalog | kafka
-[ RECORD 3 ]-------
Catalog | system
```

.. and available tables.

```
trino> show tables in kafka.sensors;
-[ RECORD 1 ]---
Table | sensors1
-[ RECORD 2 ]---
Table | sensors2
```

### Setup clickhouse

This table will store all the sensors which have failed data, by site ID.

```
trino> CREATE TABLE clickhouse.sensors.failed_sensors (id BIGINT, sensor_name VARCHAR, site INT);
```

### Do something

Lets get all failed sensors and put them in a table in clickhouse

```
trino> INSERT INTO clickhouse.sensors.failed_sensors
    ->  WITH failed_sensors AS (
    ->      SELECT id, sensor_name, 1 as site FROM kafka.sensors.sensors1 WHERE active=false UNION
    ->      SELECT id, sensor_name, 2 as site FROM kafka.sensors.sensors2 WHERE active=false
    ->  ) SELECT failed_sensors.* FROM failed_sensors;
INSERT: 111 rows
```

.. and run some queries, number of failed per site.. 

```
trino> SELECT site, count(site) AS count  FROM clickhouse.sensors.failed_sensors GROUP BY site;
-[ RECORD 1 ]
site  | 2
count | 55
-[ RECORD 2 ]
site  | 1
count | 56
```
