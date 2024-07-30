# Tutorial
This tutorial is based off of https://github.com/debezium/debezium-examples/tree/main/tutorial

# Get a ngrok auth token
Sign up for an ngrok account at https://dashboard.ngrok.com/signup and get your auth-token from https://dashboard.ngrok.com/get-started/your-authtoken.  Write it in the .env file

# Start connector
## Register postgres
`curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-postgres.json`
## Clickhouse Sink
`curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @kafka_connect_clickhouse.json`

# Delete connector
curl -i -X DELETE -H "Accept:application/json" localhost:8083/connectors/clickhouse-connect

# Debugging
https://github.com/debezium/debezium-examples/tree/main/tutorial#debugging

# Resources
How to use with ngrok https://rmoff.net/2023/11/01/using-apache-kafka-with-ngrok/
Setting auth in Confluent Schema Registry https://github.com/Dabz/kafka-security-playbook/tree/master/schema-registry/with-basic-auth
Having Kafka Connect to Confluent Schema Registry using auth https://docs.confluent.io/platform/current/schema-registry/connect.html
