### Topic: Microservices — Level 2

> **Related:** [Level 2 — REST APIs](./rest_apis.md) · [Level 2 — Circuit Breaker](./circuit_breaker_design_pattern.md) · [Level 2 — Monitoring and Alerting](./monitoring_and_alerting.md)

**Why it matters (Karat angle)**
Citi's architecture is moving from monoliths to microservices. Interviewers ask "monolith vs microservice" to see if you understand the trade-offs — not just "microservices are better." A senior answer covers orchestration vs choreography, service discovery, and the distributed system problems you introduce.

**Core concept**

| | Monolith | Microservices |
|--|---------|--------------|
| Deployment | Single deployable | Independent services |
| Scaling | Scale everything together | Scale per service |
| Data | Shared database | Database per service |
| Communication | Method calls (in-process) | HTTP/gRPC/messaging (over network) |
| Team structure | One team, one repo | Team per service, own repo |
| Complexity | Simple at first, grows unwieldy | Complex from day 1, scales better |
| Failure | One bug can crash everything | Failure is isolated (with circuit breakers) |

**When to use microservices (and when NOT to):**
- ✅ Large teams (>20 devs) needing independent deployment
- ✅ Services with different scaling needs (auth vs reports)
- ✅ Polyglot requirements (some services in Java, some in Python)
- ❌ Small team (5 devs) — microservices add operational overhead
- ❌ Tightly coupled domains — lots of cross-service calls = distributed monolith

**Orchestration vs choreography:**

| | Orchestration | Choreography |
|--|--------------|-------------|
| Pattern | Central coordinator (saga orchestrator) | Events — each service reacts independently |
| Coupling | Services coupled to orchestrator | Services decoupled (event-driven) |
| Visibility | Central flow — easy to trace | Distributed — harder to trace |
| Example | Order service calls Payment, then Shipping | Order publishes `OrderCreated`; Payment and Shipping subscribe |
| Tools | Camunda, Temporal, custom saga | Kafka, RabbitMQ, Spring Events |

```java
// File: topics/level-2/MicroserviceBasicDemo.java

// --- Orchestration example ---
@Service
public class OrderOrchestrator {
    private final PaymentClient payment;
    private final ShippingClient shipping;

    public OrderResult processOrder(Order order) {
        PaymentResult pay = payment.charge(order);                // step 1: synchronous call
        if (!pay.isSuccess()) return OrderResult.failed("Payment declined");
        ShippingResult ship = shipping.schedule(order);           // step 2: synchronous call
        return OrderResult.success(pay, ship);
    }
}

// --- Choreography example ---
@Service
public class OrderService {
    private final ApplicationEventPublisher events;

    public void createOrder(Order order) {
        orderRepo.save(order);
        events.publishEvent(new OrderCreatedEvent(order));        // fire and forget
    }
}

@Service
public class PaymentService {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // process payment independently
    }
}
```

**Key microservice patterns:**

| Pattern | What it solves |
|---------|---------------|
| **API Gateway** | Single entry point, routing, auth, rate limiting |
| **Service Discovery** | Services find each other (Eureka, Consul) |
| **Circuit Breaker** | Prevent cascading failures (Resilience4j) |
| **Saga** | Distributed transactions across services |
| **CQRS** | Separate read and write models |
| **Event Sourcing** | Store events, not state |
| **Sidecar** | Cross-cutting concerns (logging, auth) in a separate process |

**Spring Cloud stack:**

| Concern | Tool |
|---------|------|
| Service discovery | Spring Cloud Netflix Eureka / Consul |
| API Gateway | Spring Cloud Gateway |
| Config server | Spring Cloud Config |
| Circuit breaker | Resilience4j |
| Distributed tracing | Micrometer Tracing + Zipkin |
| Load balancing | Spring Cloud LoadBalancer |

**What to say in the interview (4-beat answer)**
1. **Definition:** Microservices decompose a monolith into independently deployable services, each with its own database, communicating via HTTP/messaging. They trade simplicity for scalability and team autonomy.
2. **Why/when:** Use when teams are large enough to own services independently, services have different scaling needs, or you need independent deployment cycles. Don't use for small teams or tightly coupled domains.
3. **Example:** An order service orchestrates payment and shipping via REST calls (orchestration), or publishes an `OrderCreated` event to Kafka and lets payment/shipping react independently (choreography).
4. **Gotcha/tradeoff:** Microservices introduce distributed system problems — network latency, partial failures, data consistency across services. Without circuit breakers, retries, and idempotency, you just have a distributed monolith that's harder to debug.

**Common pitfalls**
- "Microservices are always better" — they add significant operational complexity.
- Distributed monolith — microservices that can't be deployed independently because of tight coupling.
- Shared database across services — defeats the purpose; each service owns its data.
- Synchronous chains (A → B → C → D) — one slow service blocks everything; use async where possible.
- No circuit breaker — one failing service cascades to all callers.

**Self-check question**
You have a monolith processing orders, payments, and shipping. You split it into 3 microservices. An order requires payment first, then shipping. If payment succeeds but shipping fails, how do you handle the rollback without a shared database transaction?
