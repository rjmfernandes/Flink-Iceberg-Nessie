## Flink-Iceberg-Nessie Setup

Based on https://github.com/developer-advocacy-dremio/flink-iceberg-nessie-environment

Start:

```shell
docker compose up -d
```

# Setup CP

Let's install some connectors:

```shell
docker compose exec connect bash
```

And then:

```shell
confluent-hub install confluentinc/kafka-connect-datagen:latest
```

And restart connect:

```shell
docker compose restart connect
```

After we can create the connectors:

```shell
curl -i -X PUT -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/my-datagen-source1/config -d '{
    "name" : "my-datagen-source1",
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic" : "products",
    "output.data.format" : "AVRO",
    "quickstart" : "SHOES",
    "tasks.max" : "1"
}'
```

Open http://localhost:9021 and check cluster is healthy including Kafka Connect.

# Flink SQL

Start Flink SQL Client:

```shell
docker compose run sql-client
```

And execute:

```sql
CREATE TABLE products (
  `id` STRING,
  `brand` STRING,
  `name` STRING,
  `sale_price` BIGINT,
  `rating` DOUBLE,
  `$rowtime` TIMESTAMP(3) METADATA FROM 'timestamp'
) WITH (
  'connector' = 'kafka',
  'topic' = 'products',
  'properties.bootstrap.servers' = 'broker:9092',
  'properties.group.id' = 'demo3',
  'scan.startup.mode' = 'earliest-offset',
  'value.format' = 'avro-confluent',
  'value.avro-confluent.schema-registry.url' = 'http://schema-registry:8081',
  'value.fields-include' = 'EXCEPT_KEY'
);
```

After run:

```sql
select * from products;
```

Create the Iceberg catalog:

```sql
CREATE CATALOG iceberg WITH ('type'='iceberg',
'catalog-impl'='org.apache.iceberg.nessie.NessieCatalog',
'io-impl'='org.apache.iceberg.aws.s3.S3FileIO',
'uri'='http://catalog:19120/api/v1',
'authentication.type'='none',
'ref'='main',
'client.assume-role.region'='us-east-1',
'warehouse' = 's3://warehouse',
's3.endpoint'='http://storage:9000');
```

Let's use this catalog:

```sql
use catalog iceberg;
```

Create the database and use it:

```sql
create database test;
```

```sql
use test;
```

Now let's create a table:

```sql
CREATE TABLE products (
  `id` STRING,
  `brand` STRING,
  `name` STRING,
  `sale_price` BIGINT,
  `rating` DOUBLE
);
```

Let's insert values in it coming from our other table coming from kafka:

```sql
INSERT INTO products 
SELECT id, brand, `name`, sale_price, rating  FROM default_catalog.default_database.products;
```

And we query:

```sql
select * from products;
```

Check Flink dashboard at: http://localhost:8081/#/job/running

You will see the insert into iceberg catalog table job running.

Also go to MinIO http://localhost:9001/login (user/password admin/password) 

Navigate through `warehouse / test / products_... / data` and you should see the parquet files. 

Download one and open it with a tool as [Tad](https://www.tadviewer.com/).

You can also navigate with iceberg-nessie http://localhost:19120/ 

Check:
- https://projectnessie.org/
- https://iceberg.apache.org/docs/1.5.0/nessie/

## Spark Iceberg

As a final extra you can also execute:

```shell
docker compose logs spark-iceberg|grep '127\.0\.0\.1\:8888.*token'
```

And open the link displayed on your browser should be of the format:  http://127.0.0.1:8888/lab?token=8d518deb8849abf4524c7e04f588a2dbe6664d354d4cb733

Navigate to the Jupyter notebook you will find on the folder `work` to test accessing the iceberg tables from Spark.

The Spark UI while executing the notebook will be available at http://localhost:4040/ 

## Cleanup

```shell
docker compose down -v
rm -fr data
rm -fr flink_data
```