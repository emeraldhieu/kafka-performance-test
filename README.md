# Kafka Performance Test

## Quickstart

### 1) Start Kafka brokers

At the project directory, run
```sh
docker compose up -d
```

The stack spins up a quorum of 3 Kafka brokers using [KRaft](https://developer.confluent.io/learn/kraft/). It's worth to check my [Raft Consensus's revised docs](https://github.com/emeraldhieu/raft-consensus).

### 2) Tail-log brokers

Run the below commands in different tabs.
```sh
docker logs -f kafka-performance-test-kafka0-1
docker logs -f kafka-performance-test-kafka1-1
docker logs -f kafka-performance-test-kafka2-1
```

### 2) Create a topic

Log into a Kafka container
```sh
docker exec -it kafka-performance-test-kafka0-1 bash
```

Create a replicated topic
```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --create \
--bootstrap-server kafka0:9092 \
--topic products \
--partitions 6 \
--replication-factor 3
```

Describe the topic
```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --describe \
--bootstrap-server kafka0:9092 \
--topic products
```

The response looks like
```sh
Topic: products	TopicId: TFEOIlc9SKGNdn2GmFYn0g	PartitionCount: 6	ReplicationFactor: 3	Configs: segment.bytes=1073741824
	Topic: products	Partition: 0	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
	Topic: products	Partition: 1	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
	Topic: products	Partition: 2	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1
	Topic: products	Partition: 3	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2
	Topic: products	Partition: 4	Leader: 0	Replicas: 0,2,1	Isr: 0,2,1
	Topic: products	Partition: 5	Leader: 2	Replicas: 2,1,0	Isr: 2,1,0
```

### 3) Performance-test producers

Produce messages (non-TLS; plaintext)
```sh
/opt/bitnami/kafka/bin/kafka-producer-perf-test.sh \
--topic products \
--throughput -1 \
--num-records 100000 \
--record-size 1024 \
--producer-props acks=all bootstrap.servers=kafka0:9092
```

Response
```sh
40651 records sent, 8107.5 records/sec (7.92 MB/sec), 920.8 ms avg latency, 1913.0 ms max latency.
20040 records sent, 3812.8 records/sec (3.72 MB/sec), 3237.1 ms avg latency, 5782.0 ms max latency.
35550 records sent, 7110.0 records/sec (6.94 MB/sec), 6461.5 ms avg latency, 8640.0 ms max latency.
100000 records sent, 6237.914042 records/sec (6.09 MB/sec), 3480.68 ms avg latency, 8640.00 ms max latency, 2802 ms 50th, 7787 ms 95th, 8368 ms 99th, 8608 ms 99.9th.
```

### 3) Performance-test consumers

Consume messages
```sh
/opt/bitnami/kafka/bin/kafka-consumer-perf-test.sh \
--bootstrap-server kafka0:9092 \
--topic products \
--group mygroup \
--messages 100000 \
--fetch-size 1024
```

Response
```sh
start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec, rebalance.time.ms, fetch.time.ms, fetch.MB.sec, fetch.nMsg.sec
2023-06-02 08:52:22:714, 2023-06-02 08:52:29:650, 97.6563, 14.0796, 100000, 14417.5317, 3427, 3509, 27.8302, 28498.1476
```

## 4) Topic scenarios

Create a topic based on a scenario, then run producer and consumer performance tests.

#### 1) Sync, no replication

```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --create \
--bootstrap-server kafka0:9092 \
--topic test1
```

#### 2) Sync, 3x replication

```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --create \
--bootstrap-server kafka0:9092 \
--topic test2 \
--replication-factor 3
```

#### 2) Async, 3x replication

```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --create \
--bootstrap-server kafka0:9092 \
--topic test3 \
--partitions 6 \
--replication-factor 3
```

## Kafka Tools

### Kafka REST

Instead of using Kafka CLI, you can use [Kafka REST proxy](https://docs.confluent.io/platform/current/kafka-rest/index.html).

Some examples:

#### List clusters
```sh
curl -X GET -H "Accept: application/json" "http://localhost:8082/v3/clusters"
```

Response
```json
{
    "kind": "KafkaClusterList",
    "metadata":
    {
        "self": "http://localhost:8082/v3/clusters",
        "next": null
    },
    "data":
    [
        {
            "kind": "KafkaCluster",
            "brokers":
            {
                "related": "http://localhost:8082/v3/clusters/qYoMEZXcS_SKP2PzAl8-WA/brokers"
            }
        }
    ]
}
```

## Protobuf

### Create schemas

Create Protobuf schema for product key
```sh
curl -X POST -H "Content-Type:application/vnd.schemaregistry.v1+json" \
"http://localhost:8081/subjects/products-key/versions" \
--data @schemas/productKey.json
```

Create Protobuf schema for product value
```sh
curl -X POST -H "Content-Type:application/vnd.schemaregistry.v1+json" \
"http://localhost:8081/subjects/products-value/versions" \
--data @schemas/productValue.json
```

### Produce records

Produce a record with existing schemas
```sh
curl --location 'http://localhost:8082/v3/clusters/qYoMEZXcS_SKP2PzAl8-WA/topics/products/records' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data '{
    "key": {
        "schema_id": 1,
        "data": {
            "id": "2"
        }
    },
    "value": {
        "schema_id": 2,
        "data": {
            "id": "2",
            "name": "pizza",
            "price": 42
        }
    }
}'
```

Produce a record with embeded schema (DOESN'T WORK NOW. NEED CHECKING.)
```sh
curl --location 'http://localhost:8082/v3/clusters/qYoMEZXcS_SKP2PzAl8-WA/topics/products/records' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data '{
    "key": {
        "type": "AVRO",
        "schema": "{\"type\":\"string\"}",
        "data": "1"
    },
    "value": {
        "type": "PROTOBUF",
        "schema": "syntax=\"proto3\"; message Product { string id = 1; string name = 2; double price = 3; }",
        "data": {
            "id": "1",
            "name": "pizza",
            "price": 42
        }
    }
}'
```

### Consume records

Create a consumer
```sh
curl --location 'http://localhost:8082/consumers/awesome-group' \
--header 'Content-Type: application/vnd.kafka.protobuf.v2+json' \
--data '{
  "name": "awesome-consumer",
  "format": "protobuf",
  "auto.offset.reset": "earliest",
  "auto.commit.enable": "false"
}'
```

Subscribe a consumer to a topic

```sh
curl --location 'http://localhost:8082/consumers/awesome-group/instances/awesome-consumer/subscription' \
--header 'Content-Type: application/vnd.kafka.protobuf.v2+json' \
--data '{
  "topics": [
    "products"
  ]
}'
```

Consume records. Remember every time the consumer consumes records, its offset moves.
```sh
curl --location 'http://localhost:8082/consumers/awesome-group/instances/awesome-consumer/records' \
--header 'Accept: application/vnd.kafka.protobuf.v2+json'
```

## Useful commands:

Produce messages
```sh
/opt/bitnami/kafka/bin/kafka-console-producer.sh \
--bootstrap-server kafka0:9092 \
--topic products
```

Consume messages
```sh
/opt/bitnami/kafka/bin/kafka-console-consumer.sh \
--bootstrap-server kafka0:9092 \
--topic products \
--from-beginning
```

Purge a topic. After done, reset `retention.ms` to the default value 604800000.
```sh
/opt/bitnami/kafka/bin/kafka-configs.sh \
--bootstrap-server kafka0:9092 \
--entity-type topics \
--alter \
--entity-name products \
--add-config retention.ms=1000
```

Delete a topic. (Mind stoping the consumers or the topic will be recreated)
```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --delete \
--bootstrap-server kafka0:9092 \
--topic products
```

Produce a key-value message
```sh
echo "product42:{\"food\": \"pizza\"}" | \
/opt/bitnami/kafka/bin/kafka-console-producer.sh \
--bootstrap-server kafka0:9092 \
--topic products \
--property "parse.key=true" \
--property "key.separator=:"
```