## Kafka / CP + Flink + Iceberg / Nessie / MinIO + Spark

This project is based on: 
- https://github.com/developer-advocacy-dremio/flink-iceberg-nessie-environment
- https://github.com/griga23/shoe-store 


**Table Of Contents:**

- [1 - Populate Kafka Topics](#1---populate-kafka-topics)
  - [Install Plugins](#install-plugins)
  - [Create Connectors](#create-connectors)
- [2 - Flink](#2---flink)
  - [Kafka Topic Tables](#kafka-topic-tables)
  - [Keyed Tables](#keyed-tables)
- [3 - Iceberg](#3---iceberg)
- [4 - Spark](#4---spark)
- [Extra Notes](#extra-notes)
- [Cleanup](#cleanup)

First start services:

```shell
docker compose up -d
```

# 1 - Populate Kafka Topics

We are running Confluent Platform with a single Kafka node acting as broker and KRaft controller (only advised for development/trial purposes). We are also deploying Schema Registry and Kafka Connect for populating our topics with sample data in Avro format. And the Control Center for monitoring our Confluent Platform deployment.

We will create topics in Kafka and start populating with sample data from Kafka Connect connectors.

Check Kafka Connect has started:

```shell
docker compose logs -f connect
```

You should see on logs (close before the http requests log lines starting):

```
Finished starting connectors and tasks
```

## Install Plugins

If you already have the plugin downloaded locally on folder plugins you can skip this step.

Then let's install `kafka-connect-datagen` connector plugin for populating our topics with sample data:

```shell
docker compose exec connect bash
```

And then inside the container bash:

```shell
confluent-hub install confluentinc/kafka-connect-datagen:latest
```

And we exit the container bash and restart connect:

```shell
docker compose restart connect
```

Again we monitor the Kafka Connect logs and confirm it has started.

## Create Connectors

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
curl -i -X PUT -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/my-datagen-source2/config -d '{
    "name" : "my-datagen-source2",
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic" : "customers",
    "output.data.format" : "AVRO",
    "quickstart" : "SHOE_CUSTOMERS",
    "tasks.max" : "1"
}'
curl -i -X PUT -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/my-datagen-source3/config -d '{
    "name" : "my-datagen-source3",
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic" : "orders",
    "output.data.format" : "AVRO",
    "quickstart" : "SHOE_ORDERS",
    "tasks.max" : "1"
}'
```

Open Control Center http://localhost:9021 and check cluster is healthy including Kafka Connect connectors are running and our topics created and being populated.

# 2 - Flink 

We will start consuming our Kafka topics with Flink SQL and create new tables with primary keys and the final enriched one (mapped to new Kafka topics).

Start Flink SQL Client:

```shell
docker compose run sql-client
```

## Kafka Topic Tables

Let's create our first table to read from one of our Kafka topics:

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
  'properties.group.id' = 'demo1',
  'scan.startup.mode' = 'earliest-offset',
  'value.format' = 'avro-confluent',
  'value.avro-confluent.schema-registry.url' = 'http://schema-registry:8081',
  'value.fields-include' = 'EXCEPT_KEY'
);
```

After we can query:

```sql
select * from products;
```

We can also create the tables for the other two topics:

```sql
CREATE TABLE customers (
  `id` STRING,
  `first_name` STRING,
  `last_name` STRING,
  `email` STRING,
  `phone` STRING,
  `street_address` STRING,
  `state` STRING,
  `zip_code` STRING,
  `country` STRING,
  `country_code` STRING,
  `$rowtime` TIMESTAMP(3) METADATA FROM 'timestamp'
) WITH (
  'connector' = 'kafka',
  'topic' = 'customers',
  'properties.bootstrap.servers' = 'broker:9092',
  'properties.group.id' = 'demo2',
  'scan.startup.mode' = 'earliest-offset',
  'value.format' = 'avro-confluent',
  'value.avro-confluent.schema-registry.url' = 'http://schema-registry:8081',
  'value.fields-include' = 'EXCEPT_KEY'
);
```

```sql
CREATE TABLE orders (
  `order_id` BIGINT,
  `product_id` STRING,
  `customer_id` STRING,
  `$rowtime` TIMESTAMP(3) METADATA FROM 'timestamp'
) WITH (
  'connector' = 'kafka',
  'topic' = 'orders',
  'properties.bootstrap.servers' = 'broker:9092',
  'properties.group.id' = 'demo3',
  'scan.startup.mode' = 'earliest-offset',
  'value.format' = 'avro-confluent',
  'value.avro-confluent.schema-registry.url' = 'http://schema-registry:8081',
  'value.fields-include' = 'EXCEPT_KEY'
);
```

## Keyed Tables

Let's create new tables with primary keys so we get the final photo in it of each product and customer:

```sql
CREATE TABLE products_keyed(
  product_id STRING,
  brand STRING,
  model STRING,
  sale_price BIGINT,
  rating DOUBLE,
  PRIMARY KEY (product_id) NOT ENFORCED
  ) WITH (
  'connector' = 'upsert-kafka',
  'topic' = 'products_keyed',
  'properties.bootstrap.servers' = 'broker:9092',
  'value.format' = 'avro-confluent',
  'key.format' = 'raw',
  'value.avro-confluent.schema-registry.url' = 'http://schema-registry:8081',
  'value.fields-include' = 'EXCEPT_KEY'
);
```

```sql
INSERT INTO products_keyed
  SELECT id, brand, `name`, sale_price, rating 
    FROM products;
```

We can query our new table:

```sql
select * from products_keyed;
```

At the same time you can check in Control Center the new topic products_keyed has been created and is being populated from Flink.

Note: 
- The connector on the configuration of the new tables is not `kafka` anymore but we use in its place `upsert-kafka`. So that is not append only.
- At the same time on control center you can see the schema has been automatically generated and does not include product_id for the value which is stored as key (as per our configuration of the table). This comes from our Flink table configuration `EXCEPT_KEY` for `value.fields-include`.
- Finally in Flink dashboard http://localhost:8081/ (served by the Flink Job Manager service) we can see from the 5 available task slots 1 is occupied running the job of insert into the products_keyed table. You can navigate and check details on the execution graph built by Flink for the job. As well as the details on the execution stack, checkpoints, any exceptions, task managers, etc. You can also monitor the state of each task manager (in our case is only one), the logs, thread dump, etc.

Let's create now our other keyed table for customers in Flink SQL:

```sql
CREATE TABLE customers_keyed(
  customer_id STRING,
  first_name STRING,
  last_name STRING,
  email STRING,
  PRIMARY KEY (customer_id) NOT ENFORCED
  ) WITH (
  'connector' = 'upsert-kafka',
  'topic' = 'customers_keyed',
  'properties.bootstrap.servers' = 'broker:9092',
  'value.format' = 'avro-confluent',
  'key.format' = 'raw',
  'value.avro-confluent.schema-registry.url' = 'http://schema-registry:8081',
  'value.fields-include' = 'EXCEPT_KEY'
);
```

```sql
INSERT INTO customers_keyed
  SELECT id, first_name, last_name, email
    FROM customers;
```

Now we have two running jobs (and only 3 available task slots on our task manager) and a new topic again will show up in Kafka. 

# 3 - Iceberg

Let's create a new Iceberg catalog (based on Nessie implementation using S3/MinIO storage):

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

Now let's create our enrichment table:

```sql
CREATE TABLE orders (
  order_id BIGINT,
  first_name STRING,
  last_name STRING,
  email STRING,
  brand STRING,
  model STRING,
  sale_price BIGINT,
  rating DOUBLE
);
```

Let's insert values in it coming from our other tables coming from kafka:

```sql
INSERT INTO orders(
  order_id,
  first_name,
  last_name,
  email,
  brand,
  model,
  sale_price,
  rating)
SELECT
  so.order_id,
  sc.first_name,
  sc.last_name,
  sc.email,
  sp.brand,
  sp.model,
  sp.sale_price,
  sp.rating
FROM 
  default_catalog.default_database.orders so
  INNER JOIN default_catalog.default_database.customers_keyed sc 
    ON so.customer_id = sc.customer_id
  INNER JOIN default_catalog.default_database.products_keyed sp
    ON so.product_id = sp.product_id;
```

And we query (you may need to wait a bit for data to start being peristed):

```sql
select * from orders;
```

Note:
- If you check on Control Center you should be able to see (after some time) the consumer groups `demo1`, `demo2` and `demo3` corresponding to the reading of our 3 Flink tables.
- Now we have another job running considerably more complex than the first two ones due to the two joins.

There is no new topic in Kafka. But if we go to MinIO http://localhost:9001 (user/password admin/password) we should see inside `warehouse / test / orders_... / data` many parquet files being generated supporting the Iceberg storage. 

You can download one of them and open it with [Tad](https://www.tadviewer.com/). 

You can also navigate to Nessie http://localhost:19120/ and see the Iceberg entry for `test/orders`. Check the commit history.

You can also install the nessie command line tool:

```shell
pip install pynessie
```

And run for example:

```shell
nessie config -l
```

```shell
nessie remote show
```

```shell
nessie content list
```

```shell
nessie content view -r main test.orders
```

```shell
nessie log
```


# 4 - Spark 

As an example of consuming our Iceberg tables from another tool we use Spark. And access it from the Jupyter instance leveraging pyspark. For opening the Jupyter notebook server let's get the link with the token. For that we check the container log in another shell:

```shell
docker compose logs spark-iceberg|grep '127\.0\.0\.1\:8888.*token'
```

And open the link displayed on your browser should be of the format:  http://127.0.0.1:8888/lab?token=8d518deb8849abf4524c7e04f588a2dbe6664d354d4cb733

Navigate to the Jupyter notebook you will find on the folder `work` to test accessing the iceberg tables from Spark.

The notebook already displays past results for each of the commands but you will execute them yourself. You will be querying your Iceberg orders table from the pyspark notebook.

Note:
- The first command may take a while the first time you execute it cause it will in fact not only start Spark this time but download it with all the dependencies. 
- You can check meanwhile the download and startup of Spark by checking the logs of the container `spark-iceberg`.
- You can see how the count of rows increase in the table with the last command if you re-execute it (after waiting some time for checkpoints in Flink to happen).
- The Spark UI while executing the notebook will be available at http://localhost:4040/ 


# Extra Notes

- Each Flink SQL query will generate a job associated with the query execution in Flink and while is executing.
- You can scale the number of task managers by setting in docker compose file the `scale` to a number bigger than 1.
- The topics in Kafka corresponding to new tables in Flink only get automatically created when you insert a registry in the table from Flink.
- The consumer groups only appear in Control Center when there is a query executing. Typically when there is an `insert into select` that starts using the corresponding table on the select part. Until then the table is just an endpoint available to be used from Flink.
- When you start Flink it has a default catalog named `default_catalog` with a default database `default_database`. You can check those running `show catalogs;` and `show databases;`. This default catalog is of type `GenericInMemoryCatalog` and is only available in-memory for the lifetime of the session.
- The iceberg catalog will also be disposed once the session is closed but if you recreate it in another session any previous databases created on it on previous sessions will be available since they are persisted in the underlying MinIO S3 storage.
- The kafka backed queries are streaming queries and will continuously run until you cancel them. The iceberg backed queries are not, and will display the current rows in the table.
- The `insert into` iceberg backed tables will run for a long time without commiting inserts unless you set (as we did here) on the `compose.yaml` for the service the Flink property `execution.checkpointing.interval: 10000`. The commits to the storage only happen with checkpoints.  
- If you execute the pyspark notebook more than once in Jupyter it may be that the new Spark instance started finds the default port 4040 occupied and will try to occupy 4041, etc. In this case the corresponding Spark dashboard wont be available to access from host since the only port exposed in the docker compose file is the 4040.
- The Iceberg catalog in Flink will access as the endpoint for storage as per configuration the host `warehouse.storage` thats why we configured the corresponding network alias on the compose file.
- If you make changes on the `flink/Dockerfile` (adding jars etc) you will need to execute the rebuild and restart:

```shell
docker compose build flink-jobmanager
docker compose build flink-taskmanager
docker compose build sql-client
docker compose up -d
```
- In case you want to add libraries to the `flink/Dockerfile` pay attention to the version of Flink being used (1.16.1 here) and all the dependencies and subdependencies of your exact library version.

# Cleanup

```shell
docker compose down -v
rm -fr data
rm -fr flink_data
```