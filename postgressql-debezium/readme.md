# Tutorial
https://github.com/debezium/debezium-examples/tree/main/tutorial

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
