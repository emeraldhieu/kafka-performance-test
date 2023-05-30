# Kafka Performance Test

## Quickstart

### 1) Start Kafka brokers

At the project directory, run
```sh
docker compose up -d
```

The stack spins up a quorum of 3 Kafka brokers using [KRaft](https://developer.confluent.io/learn/kraft/)

If you haven't heard about Raft, it's worth to check [Raft Consensus](https://github.com/emeraldhieu/raft-consensus).

### 2) Test brokers manually

Log into a Kafka container
```sh
docker exec -it kafka-playground-kafka-0-1 bash
```

Create a replicated topic
```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --topic mytopic --partitions 3 --replication-factor 3
```

Describe the topic
```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic mytopic
```

Produce messages
```sh
/opt/bitnami/kafka/bin/kafka-console-producer.sh --bootstrap-server kafka-0:9092 --topic mytopic
```

Consume messages
```sh
/opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka-0:9092 --topic mytopic --from-beginning
```

(Both `localhost` and `kafka-0` are accepted inside the container)

### 3) Performance-test producers

Performance-test producer (non-TLS; plaintext)
```sh
/opt/bitnami/kafka/bin/kafka-producer-perf-test.sh \
--topic mytopic \
--throughput -1 \
--num-records 3000000 \
--record-size 1024 \
--producer-props acks=all bootstrap.servers=kafka-0:9092
```

Response
```sh
23400 records sent, 4679.1 records/sec (4.57 MB/sec), 6645.7 ms avg latency, 8693.0 ms max latency.
31830 records sent, 6340.6 records/sec (6.19 MB/sec), 5516.0 ms avg latency, 8937.0 ms max latency.
26190 records sent, 5238.0 records/sec (5.12 MB/sec), 5356.0 ms avg latency, 7847.0 ms max latency.
24450 records sent, 4874.4 records/sec (4.76 MB/sec), 6235.8 ms avg latency, 8756.0 ms max latency.
25245 records sent, 5017.9 records/sec (4.90 MB/sec), 6085.8 ms avg latency, 8873.0 ms max latency.
21240 records sent, 4243.8 records/sec (4.14 MB/sec), 6973.1 ms avg latency, 8885.0 ms max latency.
27300 records sent, 5458.9 records/sec (5.33 MB/sec), 6400.2 ms avg latency, 8967.0 ms max latency.
3000000 records sent, 4711.846997 records/sec (4.60 MB/sec), 6436.16 ms avg latency, 20086.00 ms max latency, 5832 ms 50th, 11316 ms 95th, 15029 ms 99th, 18156 ms 99.9th.
```

### 4) Performance test scenarios

