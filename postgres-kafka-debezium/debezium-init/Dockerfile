FROM debian

RUN apt update && apt install -y curl

ENTRYPOINT curl -XPUT http://debezium:8083/connectors/outbox-connector/config -d @/register-postgres.json -H 'Content-Type:application/json' -H 'Accept:application/json'