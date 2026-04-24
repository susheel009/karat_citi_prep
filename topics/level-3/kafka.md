### Topic: Kafka (Scalability, Fault Tolerance, Monitoring) — Level 3

> **Related:** [Level 3 — Event-Driven Messaging Systems](./event_driven_messaging_systems.md) · [Level 2 — Microservices](../level-2/microservices.md)

**Why it matters (Karat angle)**
Citi uses Kafka for real-time transaction processing, CDC (Change Data Capture), and event sourcing. Interviewers ask about partitioning strategy, consumer groups, rebalancing, and exactly-once semantics — not just "we use Kafka."

**Core concept**

**Kafka architecture:**
```
Producer → Topic (partitioned) → Consumer Group
                                    ├── Consumer 1 → Partition 0, 1
                                    ├── Consumer 2 → Partition 2, 3
                                    └── Consumer 3 → Partition 4
```

| Concept | What it is |
|---------|-----------|
| **Topic** | Named stream of records (like a table) |
| **Partition** | Ordered, immutable log within a topic (unit of parallelism) |
| **Offset** | Position of a record within a partition |
| **Consumer Group** | Group of consumers that share partition assignments |
| **Replication Factor** | Number of copies of each partition (fault tolerance) |
| **ISR** (In-Sync Replicas) | Replicas that are caught up with the leader |

**Scalability — partitioning strategy:**

| Strategy | Key | Ordering guarantee |
|----------|-----|:------------------:|
| Round-robin | No key | None |
| Key-based | `accountId` | Per key (same partition) |
| Custom | `region + accountId` | Per custom key |

```java
// File: topics/level-3/KafkaPartitionDemo.java

// Key-based: same account always goes to the same partition → ordered per account
kafkaTemplate.send("transactions", transaction.getAccountId(), transaction);

// Custom partitioner
public class RegionPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                          Object value, byte[] valueBytes, Cluster cluster) {
        String accountId = (String) key;
        String region = accountId.substring(0, 2);         // first 2 chars = region
        int numPartitions = cluster.partitionCountForTopic(topic);
        return Math.abs(region.hashCode()) % numPartitions;
    }
}
```

**Fault tolerance:**

```yaml
# Replication factor = 3: each partition has 1 leader + 2 followers
# min.insync.replicas = 2: at least 2 replicas must acknowledge a write
# acks = all: producer waits for all ISR replicas to acknowledge

# Producer config
spring:
  kafka:
    producer:
      acks: all                                            # wait for all ISR replicas
      retries: 3
      properties:
        enable.idempotence: true                           # prevent duplicate writes on retry
        max.in.flight.requests.per.connection: 5
```

**Consumer configuration:**

```java
// File: topics/level-3/KafkaConsumerConfig.java

@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setConcurrency(3);                         // 3 consumer threads
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate),  // send to DLT after retries
            new FixedBackOff(1000, 3)                          // 3 retries, 1s apart
        ));
        return factory;
    }
}
```

**Monitoring Kafka:**

| Metric | What to watch | Alert threshold |
|--------|-------------|:--------------:|
| **Consumer lag** | Offset behind the latest | > 1000 records |
| **Under-replicated partitions** | Replicas not in sync | > 0 |
| **Request latency** | Producer/consumer latency | p99 > 500ms |
| **Broker disk usage** | Log retention filling disk | > 80% |
| **ISR shrink rate** | Replicas falling behind | Increasing |

```java
// File: topics/level-3/KafkaMonitoringDemo.java

// Expose consumer lag via Micrometer
@Bean
public MicrometerConsumerListener<String, String> consumerListener(MeterRegistry registry) {
    return new MicrometerConsumerListener<>(registry);
}

// Check lag programmatically
@Scheduled(fixedRate = 60_000)
public void checkConsumerLag() {
    try (AdminClient admin = AdminClient.create(kafkaProps)) {
        Map<TopicPartition, OffsetAndMetadata> committed = admin
            .listConsumerGroupOffsets("payment-service")
            .partitionsToOffsetAndMetadata().get();
        Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> endOffsets = admin
            .listOffsets(committed.keySet().stream()
                .collect(Collectors.toMap(tp -> tp, tp -> OffsetSpec.latest())))
            .all().get();

        committed.forEach((tp, offset) -> {
            long lag = endOffsets.get(tp).offset() - offset.offset();
            if (lag > 1000) log.warn("HIGH LAG: {} lag={}", tp, lag);
        });
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Kafka is a distributed log. Topics are partitioned for parallelism. Consumer groups distribute partitions across consumers. Replication factor and ISR provide fault tolerance.
2. **Why/when:** Use Kafka for high-throughput event streaming (millions/sec), CDC, and decoupled microservices. Key-based partitioning ensures per-key ordering (all events for an account on the same partition).
3. **Example:** `acks=all` + `min.insync.replicas=2` + replication factor 3 = no data loss even if one broker dies. Consumer group with 3 consumers and 6 partitions = 2 partitions per consumer.
4. **Gotcha/tradeoff:** More partitions = more parallelism but more rebalancing overhead. Consumer group rebalancing pauses consumption — use `CooperativeStickyAssignor` to minimise disruption. Consumer lag must be monitored — unbounded lag means the consumer can't keep up.

**Common pitfalls**
- More consumers than partitions — excess consumers sit idle.
- `acks=0` or `acks=1` in financial systems — data loss on broker failure.
- No dead-letter topic (DLT) — poison messages block the consumer forever.
- Not monitoring consumer lag — consumer falls hours behind, data goes stale.
- Processing order assumption across partitions — Kafka guarantees order WITHIN a partition, not across.

**Self-check question**
Your topic has 6 partitions and 3 consumers in a group. One consumer crashes. What happens to its partitions? How long does rebalancing take? What does the consumer experience during rebalancing?
