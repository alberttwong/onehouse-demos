# onehouse-demos

## CDC Demo
* postgressql-debezium
  * postgres (with debezium configured) pushing messages into Kafka

## Query Demo
* trino-prestodb-spark-minio
  * Use Trino or PrestoDB to read Onehouse (iceberg, hudi, delta) and write data in OneHouse.  Also contains Apache xTable to translate open table format.
  * Use Spark to write Iceberg, Hudi and/or Delta Lake and upload the metadata to Onehouse Lake View

## Visability into Kafka
* conduktor
  * Tool to see messages in Kafka topics

