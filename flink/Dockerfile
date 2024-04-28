# Start from the official Flink image
FROM flink:1.16.1-scala_2.12

###############################################
## Download Neccessary Jars to Flink Class Path
###############################################

## Iceberg Flink Library
RUN curl -L https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-flink-runtime-1.16/1.3.1/iceberg-flink-runtime-1.16-1.3.1.jar -o /opt/flink/lib/iceberg-flink-runtime-1.16-1.3.1.jar

## Hive Flink Library
RUN curl -L https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-hive-2.3.9_2.12/1.16.1/flink-sql-connector-hive-2.3.9_2.12-1.16.1.jar -o /opt/flink/lib/flink-sql-connector-hive-2.3.9_2.12-1.16.1.jar

## Hadoop Common Classes
RUN curl -L https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-common/2.8.3/hadoop-common-2.8.3.jar -o /opt/flink/lib/hadoop-common-2.8.3.jar

## Hadoop AWS Classes
RUN curl -L https://repo.maven.apache.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.8.3-10.0/flink-shaded-hadoop-2-uber-2.8.3-10.0.jar -o /opt/flink/lib/flink-shaded-hadoop-2-uber-2.8.3-10.0.jar 

## AWS Bundled Classes
RUN curl -L https://repo1.maven.org/maven2/software/amazon/awssdk/bundle/2.20.18/bundle-2.20.18.jar -o /opt/flink/lib/bundle-2.20.18.jar

## Icberg Nessie Library
# RUN curl -L https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-nessie/1.3.0/iceberg-nessie-1.3.0.jar -o /opt/flink/lib/iceberg-nessie-1.3.0.jar

# for cp
RUN curl -L  https://repo1.maven.org/maven2/org/apache/flink/flink-connector-kafka/1.16.1/flink-connector-kafka-1.16.1.jar -o /opt/flink/lib/flink-connector-kafka-1.16.1.jar
RUN curl -L  https://repo1.maven.org/maven2/org/apache/kafka/kafka-clients/3.2.3/kafka-clients-3.2.3.jar -o /opt/flink/lib/kafka-clients-3.2.3.jar
RUN curl -L  https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-kafka/1.16.1/flink-sql-connector-kafka-1.16.1.jar -o /opt/flink/lib/flink-sql-connector-kafka-1.16.1.jar
RUN curl -L  https://repo1.maven.org/maven2/org/apache/flink/flink-shaded-force-shading/15.0/flink-shaded-force-shading-15.0.jar -o /opt/flink/lib/flink-shaded-force-shading-15.0.jar
RUN curl -L  https://repo1.maven.org/maven2/org/apache/flink/flink-avro/1.16.1/flink-avro-1.16.1.jar -o /opt/flink/lib/flink-avro-1.16.1.jar
RUN curl -L  https://repo1.maven.org/maven2/org/apache/flink/flink-avro-confluent-registry/1.16.1/flink-avro-confluent-registry-1.16.1.jar -o /opt/flink/lib/flink-avro-confluent-registry-1.16.1.jar
RUN curl -L  https://packages.confluent.io/maven/io/confluent/kafka-schema-registry-client/6.2.2/kafka-schema-registry-client-6.2.2.jar -o /opt/flink/lib/kafka-schema-registry-client-6.2.2.jar
RUN curl -L  https://repo1.maven.org/maven2/org/apache/avro/avro/1.11.1/avro-1.11.1.jar -o /opt/flink/lib/avro-1.11.1.jar

## Install Nano to edit files
# RUN apt update && apt install -y nano

CMD ["./bin/start-cluster.sh"]
