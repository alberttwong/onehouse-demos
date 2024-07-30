# onehouse-demos

## CDC Demo
* postgressql-debezium
  * [Onehouse] postgres (with debezium configured) pushing messages into Kafka and Confluent Schema Registry to be picked up by Onehouse ingestion (Confluent Cloud Kafka with AVRO)
* mysql-debezium
  * [Onehouse] mysql (with debezium configured) pushing messages into Kafka and Confluent Schema Registry to be picked up by Onehouse ingestion (Confluent Cloud Kafka with AVRO)

## Query Demo
* trino-prestodb-spark-minio
  * [Community] Use Spark to write Hudi, have Apache xTable convert the format to Iceberg and Delta Lake, read the data using Trino
  * [Community] Use Spark to write Iceberg, have Apache xTable convert the format to Hudi and Delta Lake, read the data using Trino
  * [Community] Use Spark to write Delta Lake, have Apache xTable convert the format to Iceberg and Hudi, read the data using Trino

## Observability Demo    
  * [Onehouse LakeView] Use Spark to write Hudi upload the metadata to Onehouse Lake View
  * [Onehouse LakeView] Use Spark to write Iceberg, have Apache xTable convert to Hudi, upload the metadata to Onehouse Lake View
  * [Onehouse LakeView] Use Spark to write Delta Lake, have Apache xTable convert to Hudi, upload the metadata to Onehouse Lake View

## Visability into Kafka
* conduktor
  * Tool to see messages in Kafka topics

