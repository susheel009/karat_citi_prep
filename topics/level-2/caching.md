### Topic: Caching — Level 2

> **Related:** [Level 2 — Spring Boot (caching)](./spring_boot.md) · [Level 2 — Microservices](./microservices.md)

**Why it matters (Karat angle)**
Caching turns a 50ms DB call into a 0.1ms memory lookup. Interviewers test whether you understand eviction strategies, cache-aside vs write-through, and the cache invalidation problem — not just `@Cacheable`.

**Core concept**

**Cache strategies:**

| Strategy | How it works | Consistency | Use case |
|----------|-------------|:-----------:|---------|
| **Cache-aside** (lazy) | App checks cache → miss → load from DB → put in cache | Eventually consistent | Most read-heavy apps |
| **Write-through** | Write to cache AND DB simultaneously | Strong | Financial data |
| **Write-behind** | Write to cache → async flush to DB | Weak | High write throughput |
| **Read-through** | Cache loads from DB on miss (transparent) | Eventually consistent | Cache providers like Hazelcast |

**Cache eviction policies:**

| Policy | Evicts | Best for |
|--------|--------|---------|
| **LRU** (Least Recently Used) | Oldest accessed | General purpose |
| **LFU** (Least Frequently Used) | Least popular | Long-lived popularity patterns |
| **TTL** (Time To Live) | After fixed time | Time-sensitive data (exchange rates) |
| **FIFO** (First In First Out) | Oldest inserted | Simple, predictable |

**Spring Boot caching in depth:**

```java
// File: topics/level-2/CachingDetailsDemo.java

@Configuration
@EnableCaching
public class CacheConfig {

    // --- In-memory cache (for single-instance apps) ---
    @Bean
    public CacheManager simpleCacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)                            // max entries
            .expireAfterWrite(Duration.ofMinutes(10))     // TTL
            .recordStats()                                // expose metrics
        );
        return manager;
    }
}

@Service
public class ExchangeRateService {

    // Cache-aside: check cache → miss → call method → store result
    @Cacheable(value = "exchangeRates", key = "#from + '-' + #to")
    public BigDecimal getRate(String from, String to) {
        // Only called on cache miss
        return externalRateApi.fetchRate(from, to);
    }

    // Update cache entry on write
    @CachePut(value = "exchangeRates", key = "#from + '-' + #to")
    public BigDecimal updateRate(String from, String to, BigDecimal rate) {
        rateRepo.save(new ExchangeRate(from, to, rate));
        return rate;                                       // returned value is cached
    }

    // Evict on delete
    @CacheEvict(value = "exchangeRates", key = "#from + '-' + #to")
    public void invalidateRate(String from, String to) {
        // Cache entry removed
    }

    // Conditional caching — only cache if result is non-null
    @Cacheable(value = "exchangeRates", key = "#from + '-' + #to",
               unless = "#result == null")
    public BigDecimal getRateOrNull(String from, String to) {
        return rateRepo.findRate(from, to).orElse(null);
    }
}
```

**Distributed caching with Redis:**

```java
// File: topics/level-2/RedisCacheDemo.java

// application.yml:
// spring:
//   cache:
//     type: redis
//   redis:
//     host: redis.citi.com
//     port: 6379
//     timeout: 2000

@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                ))
            .disableCachingNullValues();

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("exchangeRates",
                config.entryTtl(Duration.ofMinutes(5)))    // shorter TTL for rates
            .build();
    }
}
```

| | In-memory (Caffeine) | Distributed (Redis) |
|--|---------------------|-------------------|
| Speed | ~nanoseconds | ~1ms (network hop) |
| Scope | Single JVM | All app instances |
| Consistency | Per-instance (stale across instances) | Shared (consistent across instances) |
| Size limit | JVM heap | Redis memory (separate server) |
| Use case | Single-instance, hot data | Multi-instance, shared state |

**What to say in the interview (4-beat answer)**
1. **Definition:** Caching stores frequently accessed data closer to the application. Cache-aside (lazy loading) is most common — check cache, miss → load from DB → store in cache. Eviction via TTL, LRU, or explicit `@CacheEvict`.
2. **Why/when:** Use in-memory (Caffeine) for single-instance apps. Distributed (Redis) for multi-instance. Write-through for consistency-critical data. Cache-aside for read-heavy workloads.
3. **Example:** `@Cacheable(value = "rates", key = "#from + '-' + #to")` — first call fetches from API (50ms); subsequent calls return from cache (0.1ms). TTL of 5 minutes ensures rates don't go stale.
4. **Gotcha/tradeoff:** "There are only two hard things in CS: cache invalidation and naming things." Stale cache = serving outdated data. Always set TTL and use `@CacheEvict` on writes. Caching mutable entities causes `LazyInitializationException` — cache DTOs instead.

**Common pitfalls**
- Caching without TTL — data never expires; stale data served forever.
- Caching null values — cache fills with nulls; subsequent requests never hit the DB even when data exists.
- In-memory cache in multi-instance deployment — each instance has different cache contents; inconsistent responses.
- `@Cacheable` on a method that returns JPA entities with lazy collections — `LazyInitializationException` on cache hit.
- Not monitoring cache hit rate — a cache with 10% hit rate wastes memory.

**Self-check question**
Your service has 3 instances behind a load balancer, using in-memory Caffeine cache. Instance 1 caches account "A001" with balance $1000. A write request hits instance 2, updating the balance to $500. What happens when the next read hits instance 1?
