# About
You can use this docker compose to setup a OLTP CDC scenario into Onehouse. 

## Choose your OLTP
Debezium provides example images for many databases like mysql, postgres, mongodb (full list at https://github.com/debezium/debezium-examples/tree/main/tutorial).  In this example, it's using the mysql image. 

## Get a ngrok auth token
Sign up for an ngrok account at https://dashboard.ngrok.com/signup and get your auth-token from https://dashboard.ngrok.com/get-started/your-authtoken.  Write it in the docker-compose file

# Start connector
## Register mysql
`curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-mysql-avro.json`

## Get the Kafka broker address to put into Onehouse
Execute `curl -s http://localhost:4040/api/tunnels/kafka` to get the URI of the kafka broker address.  Take that address and put it as a Onehouse Source - Kafka with PLAINTEXT security.

You can also see the ngrok URI through the ngrok logs or the kafka logs within their container. 

## Debugging
https://github.com/debezium/debezium-examples/tree/main/tutorial#debugging

## Resources
* Getting Apache Kafka to work with ngrok https://rmoff.net/2023/11/01/using-apache-kafka-with-ngrok/
* This tutorial is based off of https://github.com/debezium/debezium-examples/tree/main/tutorial
* Setting auth in Confluent Schema Registry https://github.com/Dabz/kafka-security-playbook/tree/master/schema-registry/with-basic-auth
* Having Kafka Connect to Confluent Schema Registry using auth https://docs.confluent.io/platform/current/schema-registry/connect.html
