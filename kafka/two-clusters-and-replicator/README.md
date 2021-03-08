# Create, verify, and write to topic on Source A Broker 1
```
docker run \
  --net=host \
  --rm confluentinc/cp-kafka:latest \
  kafka-topics --create --topic foo --partitions 3 --replication-factor 2 --if-not-exists --zookeeper localhost:22181

docker run \
  --net=host \
  --rm confluentinc/cp-kafka:latest \
  kafka-topics --describe --topic foo --zookeeper localhost:22181

docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:latest \
  bash -c "seq 1000 | kafka-console-producer --request-required-acks 1 --broker-list localhost:9092 --topic foo && echo 'Produced 1000 messages.'"
```
# POST replicator to Connect

```
docker exec -it documents_connect-host-1_1 bash

curl -X POST \
     -H "Content-Type: application/json" \
     --data '{
        "name": "replicator-src-a-foo",
        "config": {
          "connector.class":"io.confluent.connect.replicator.ReplicatorSourceConnector",
          "key.converter": "io.confluent.connect.replicator.util.ByteArrayConverter",
          "value.converter": "io.confluent.connect.replicator.util.ByteArrayConverter",
          "src.zookeeper.connect": "localhost:22181",
          "src.kafka.bootstrap.servers": "localhost:9092",
          "dest.zookeeper.connect": "localhost:42181",
          "topic.whitelist": "foo",
          "topic.rename.format": "${topic}.replica"}}'  \
     http://localhost:38082/connectors
```


# Read from Destination replicated topic
```
docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:latest \
  kafka-console-consumer --bootstrap-server localhost:9052 --topic foo.replica --new-consumer --from-beginning --max-messages 1000
```
# Verify that it's actually a replica
```
docker run \
  --net=host \
  --rm confluentinc/cp-kafka:latest \
  kafka-topics --describe --topic foo.replica --zookeeper localhost:42181
```