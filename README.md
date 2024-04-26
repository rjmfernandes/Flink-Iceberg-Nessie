## Flink-Iceberg-Nessie Setup

Based on https://github.com/developer-advocacy-dremio/flink-iceberg-nessie-environment

Start:

```shell
docker compose up -d
```

Start Flink SQL Client:

```shell
docker compose run sql-client
```

Create the Iceberg catalog (replacing the IP by the one found before):

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

Now let's create a table and insert some values in it:

```sql
CREATE TABLE products (
  `id` STRING,
  `brand` STRING,
  `name` STRING,
  `sale_price` BIGINT,
  `rating` DOUBLE
);
```

```sql
INSERT INTO products
  VALUES ('demo1', 'apple','iphone', 1000,4), ('demo2', 'apple','mac', 2000,5);
```

Check Flink dashboard at: http://localhost:8081/#/job/running

You may still have time to see the insert into iceberg catalog table job running.

Else check under Completed http://localhost:8081/#/job/completed

You can run on Flink SQL shell:

```sql
select * from products;
```

Also go to MinIO http://localhost:9001/login (user/password admin/password) 

Navigate through `warehouse / test / products_... / data` and you should see the parquet file. 

Download and open it with a tool as [Tad](https://www.tadviewer.com/).

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
```