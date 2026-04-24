### Topic: Event-Driven Messaging Systems — Level 3

> **Related:** [Level 3 — Kafka](./kafka.md) · [Level 2 — Microservices](../level-2/microservices.md)

**Why it matters (Karat angle)**
Citi processes millions of transactions daily using event-driven architecture. Interviewers ask about messaging patterns, delivery guarantees, and when to use async messaging vs synchronous REST.

**Core concept**

**Event-driven vs request-response:**

| | Request-Response (REST) | Event-Driven (Messaging) |
|--|------------------------|-------------------------|
| Coupling | Tight — caller knows callee | Loose — producer doesn't know consumers |
| Latency | Synchronous — caller waits | Asynchronous — fire and forget |
| Scalability | Limited by slowest service | Independent scaling |
| Failure | Cascading | Isolated (messages buffered) |
| Use case | CRUD, queries | Notifications, event sourcing, data pipelines |

**Messaging patterns:**

| Pattern | How | Use case |
|---------|-----|---------|
| **Point-to-point** | One producer → queue → one consumer | Task distribution |
| **Pub/sub** | One producer → topic → many consumers | Event broadcasting |
| **Request-reply** | Send request message → receive reply message | Async request-response |
| **Event sourcing** | Store events (not state) → rebuild state from events | Audit-complete systems |
| **CQRS** | Separate write (command) model from read (query) model | High-read, low-write systems |

**Messaging brokers comparison:**

| | Kafka | RabbitMQ | ActiveMQ |
|--|-------|---------|----------|
| Model | Distributed log | Queue/Exchange | Queue |
| Ordering | Per partition | Per queue | Per queue |
| Retention | Configurable (days/forever) | Until consumed | Until consumed |
| Throughput | Very high (millions/sec) | High (100K/sec) | Moderate |
| Use at Citi | Event streaming, CDC | Task queues | Legacy JMS |

**Delivery guarantees:**

| Guarantee | Meaning | Implementation |
|-----------|---------|---------------|
| **At-most-once** | May lose messages | Fire and forget, no acks |
| **At-least-once** | May deliver duplicates | Ack after processing; retry on failure |
| **Exactly-once** | No loss, no duplicates | Transactional producer + idempotent consumer |

**Spring Boot + Kafka integration:**

```java
// File: topics/level-3/KafkaIntegration.java

// --- Producer ---
@Service
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public CompletableFuture<SendResult<String, OrderEvent>> publishOrderCreated(OrderEvent event) {
        return kafkaTemplate.send("order-events", event.getOrderId(), event);
    }
}

// --- Consumer ---
@Service
public class PaymentEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "payment-service")
    public void handleOrderCreated(OrderEvent event, Acknowledgment ack) {
        try {
            paymentService.processPayment(event);
            ack.acknowledge();                             // manual ack — at-least-once
        } catch (Exception e) {
            log.error("Failed to process: {}", event, e);
            // Don't ack — message will be redelivered
        }
    }
}
```

**Idempotent consumer — handling duplicates:**

```java
// File: topics/level-3/IdempotentConsumer.java

@Service
public class PaymentEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "payment-service")
    public void handle(OrderEvent event, Acknowledgment ack) {
        // Check if already processed (idempotency key)
        if (processedEventRepo.exists(event.getEventId())) {
            log.info("Duplicate event: {}", event.getEventId());
            ack.acknowledge();
            return;
        }

        paymentService.processPayment(event);
        processedEventRepo.save(event.getEventId());       // mark as processed
        ack.acknowledge();
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Event-driven systems decouple producers from consumers via a message broker (Kafka, RabbitMQ). Patterns include pub/sub (broadcast), point-to-point (task queue), and event sourcing (store events, rebuild state).
2. **Why/when:** Use messaging for asynchronous workflows (order → payment → shipping), data pipelines, and decoupling microservices. Use REST for synchronous queries and simple CRUD.
3. **Example:** Order service publishes `OrderCreated` to Kafka. Payment service and shipping service both consume it independently. If payment service is down, messages are buffered in Kafka until it recovers.
4. **Gotcha/tradeoff:** At-least-once delivery means consumers must be idempotent — check for duplicate event IDs before processing. Exactly-once is expensive (transactional producer + idempotent consumer + Kafka transactions).

**Common pitfalls**
- Non-idempotent consumers with at-least-once delivery — duplicate processing (double payments!).
- No dead-letter queue — poison messages block the consumer forever.
- Synchronous processing in consumer blocking other messages — use async processing or separate thread pools.
- Not monitoring consumer lag — consumer falls behind and data becomes stale.

**Self-check question**
Your payment consumer processes an `OrderCreated` event, charges the customer, but crashes before acknowledging the message. What happens when the consumer restarts? How do you prevent the customer from being charged twice?
