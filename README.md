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
docker logs -f kafka-performance-test-kafka-0-1
docker logs -f kafka-performance-test-kafka-1-1
docker logs -f kafka-performance-test-kafka-2-1
```

### 2) Create a topic

Log into a Kafka container
```sh
docker exec -it kafka-performance-test-kafka-0-1 bash
```

Create a replicated topic
```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--topic mytopic \
--partitions 6 \
--replication-factor 3
```

Describe the topic
```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --describe \
--bootstrap-server localhost:9092 \
--topic mytopic
```

The response looks like
```sh
Topic: mytopic	TopicId: TFEOIlc9SKGNdn2GmFYn0g	PartitionCount: 6	ReplicationFactor: 3	Configs: segment.bytes=1073741824
	Topic: mytopic	Partition: 0	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
	Topic: mytopic	Partition: 1	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
	Topic: mytopic	Partition: 2	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1
	Topic: mytopic	Partition: 3	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2
	Topic: mytopic	Partition: 4	Leader: 0	Replicas: 0,2,1	Isr: 0,2,1
	Topic: mytopic	Partition: 5	Leader: 2	Replicas: 2,1,0	Isr: 2,1,0
```

### 3) Performance-test producers

Produce messages (non-TLS; plaintext)
```sh
/opt/bitnami/kafka/bin/kafka-producer-perf-test.sh \
--topic mytopic \
--throughput -1 \
--num-records 100000 \
--record-size 1024 \
--producer-props acks=all bootstrap.servers=kafka-0:9092
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
--bootstrap-server kafka-0:9092 \
--topic mytopic \
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
--bootstrap-server localhost:9092 \
--topic test1
```

#### 2) Sync, 3x replication

```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--topic test2 \
--replication-factor 3
```

#### 2) Async, 3x replication

```sh
/opt/bitnami/kafka/bin/kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--topic test3 \
--partitions 6 \
--replication-factor 3
```

## Useful commands:

Produce messages
```sh
/opt/bitnami/kafka/bin/kafka-console-producer.sh \
--bootstrap-server kafka-0:9092 \
--topic mytopic
```

Consume messages
```sh
/opt/bitnami/kafka/bin/kafka-console-consumer.sh \
--bootstrap-server kafka-0:9092 \
--topic mytopic \
--from-beginning
```

Purge a topic. After done, reset `retention.ms` to the default value 604800000.
```sh
/opt/bitnami/kafka/bin/kafka-configs.sh \
  --bootstrap-server kafka-0:9092 \
  --entity-type topics \
  --alter \
  --entity-name mytopic \
  --add-config retention.ms=1000
```