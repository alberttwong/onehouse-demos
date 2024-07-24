# About
You can use this docker compose to setup a OLTP CDC scenario into Onehouse. 

## Choose your OLTP
Debezium provides example images for many databases like mysql, postgres, mongodb (full list at https://github.com/debezium/debezium-examples/tree/main/tutorial).  In this example, it's using the postgres image. 

## Get a ngrok auth token
Sign up for an ngrok account at https://dashboard.ngrok.com/signup and get your auth-token from https://dashboard.ngrok.com/get-started/your-authtoken.  Write it in the .env file

## Start connector
### Register postgres
`curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-postgres.json`

## Get the Kafka broker address to put into Onehouse
Execute `curl -s http://localhost:4040/api/tunnels/command_line` to get the URI of the kafka broker address.  Take that address and put it as a Onehouse Source - Kafka with PLAINTEXT security.

## Debugging
https://github.com/debezium/debezium-examples/tree/main/tutorial#debugging

## Resources
* Getting Apache Kafka to work with ngrok https://rmoff.net/2023/11/01/using-apache-kafka-with-ngrok/
* This tutorial is based off of https://github.com/debezium/debezium-examples/tree/main/tutorial

# Other uses
Instead of sinking the data into Onehouse, you can sink the data into a OLAP database 

## Example sink to Clickhouse

### Clickhouse Sink
`curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @kafka_connect_clickhouse.json`

## Delete connector
`curl -i -X DELETE -H "Accept:application/json" localhost:8083/connectors/clickhouse-connect`
