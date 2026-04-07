# Idempotency in Payment Systems

## Context
Idempotency is a property of a system's operation or a request that produces the same results whether it is executed once or many times. In a distributed environment, it is not uncommon for networks to become unavailable at times, or server crash happening mid request. 

In payment systems, this is not just a technical concern. Without idempotency:

- Customers can be charged multiple times.
- Refunds can be duplicated.
- Ledger entries can drift.
- Reconciliation complexity increases significantly.
- The financial and reputational impact is immediate.

---

## The problem
Consider a client submitting a payment request to API. The request reaches the server and payment is processed succesfully. But a network unavailability caused the response to never make it back to the client. The client has no idea if the payment is processed or not, and they retries. 

**Without idempotency**
```
Client                    Load Balancer              Payment Service
  |                            |                           |
  |------- POST /payments ----->|                           |
  |                            |-------- forward ---------->|
  |                            |                           | (payment processed)
  |                            |                           | (network failure: response lost)
  |<------- timeout -----------|                           |
  |                            |                           |
  |------- POST /payments ----->|  (retry)                 |
  |                            |-------- forward ---------->|
  |                            |                           | (DUPLICATE payment processed)
  |                            |<------- 200 OK -----------|
  |<------- 200 OK ------------|                           |
```

The client sees 200 response second time. The issue with duplicate payment isn't picked up until reconciliation or if customer complaints.  

---

## The solution - Idempotency Key
The standard, accepted solution is the *idempotency key*, a client-generated unique id attached to each request, that a server uses to recognize and deduplicate the repeated requests. 

**With idempotency**
```
Client                                        Payment Service
  |                                                |
  | POST /payments                                 |
  | Idempotency-Key: a3f8c2d1-...                  |
  |----------------------------------------------->|
  |                                                | Store key + lock
  |                                                | Process payment
  |                                                | Store result against key
  |                                                | (Resonse lost: network failure)
  |                                                |
  | (timeout — client unsure if succeeded)         |
  |                                                |
  | POST /payments                                 |
  | Idempotency-Key: a3f8c2d1-... (same key)       |
  |----------------------------------------------->|
  |                                                | Key found in store
  |                                                | Return cached result
  |<---------- 200 OK { payment_id: ... } ---------|
  |                                    (same response, no duplicate payment)
```

The client uses the same key on every retry for the same logical operation. The server processes it once and returns the cached response on all subsequent calls.

---

## Implementation

### Idempotency record schema

```sql
CREATE TABLE idempotency_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key             VARCHAR(255) NOT NULL,
    client_id       UUID NOT NULL,               -- scope keys per client/tenant
    request_path    VARCHAR(255) NOT NULL,        -- POST /payments
    request_hash    VARCHAR(64) NOT NULL,         -- SHA-256 of request body
    response_status INTEGER,
    response_body   JSONB,
    processing      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
 
    CONSTRAINT uq_key_client UNIQUE (key, client_id) -- only one record per idempotency key per client should exist
);
 
CREATE INDEX idx_idempotency_keys_expires ON idempotency_keys (expires_at); 
```

Key design decisions here:
- **Scoped per client**: `(key, client_id)` unique together. Two different clients can use the same key without collision.
- **Request hash**: allows to detect and reject requests where the same key is submitted with a different body (a genuine client bug or attack vector, not a retry).
- **`processing` flag**: used as a mutex to handle concurrent submissions of the same key (see below).
- **`expires_at`**: keys do not live forever. 24 hours is a common window; 7 days is reasonable for async flows.
 
---

### Repository: Pessimistic Lock
 
The `FOR UPDATE` lock is to avoid two concurrent retries to pass the existence check simultaneously and proceed to process the payment.
 
```java
public interface IdempotencyKeyRepository extends JpaRepository<IdempotencyKey, UUID> {
 
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
        SELECT i FROM IdempotencyKey i
        WHERE i.key = :key AND i.clientId = :clientId
    """)
    Optional<IdempotencyKey> findByKeyAndClientIdForUpdate(
        @Param("key") String key,
        @Param("clientId") UUID clientId
    );
}
```
 
---

### Atomic result storage
 
The most critical, and most commonly missed, requirement is that the payment and the idempotency record must be committed **in the same transaction**. If they are not:
 
```java
// DANGEROUS: two separate transactions
PaymentResponse response = paymentProcessor.charge(request); // transaction 1 commits
saveIdempotencyResult(key, response);                        // transaction 2 - JVM crashes here
// Payment exists. Idempotency key has no result. Next retry charges again.
```
 
```java
// CORRECT: single @Transactional boundary covers both writes
@Transactional
public PaymentResponse processPayment(...) {
    PaymentResponse response = paymentProcessor.charge(request); // within transaction
    updateIdempotencyKey(key, response);                         // within same transaction
    // Both committed atomically on method return, or both rolled back on exception
}
```

> This requires idempotency table and payments table to share the same datasource. If they cannot, for example, if the payment processor writes to a separate database or calls an external service, the alternate is to use the *Redis-based outbox pattern* approach.

> Few common idempotency-approaches are discussed next.

---

## Idempotency approaches

### Database-backed idempotency (recommended for most teams)
Store idempotency keys in the same relational database as your payment records. Use a single `@Transactional` method to commit both atomically. This approach makes systems simple, consistent, auditable, easy to query for debugging.
 
**Fits:** Most fintech systems, particularly where payment volume is in the thousands to low millions per day.
 
**Limitations:** `PESSIMISTIC_WRITE` lock adds latency under high contention. At very high write volumes, consider partitioning the `idempotency_keys` table by `expires_at` to keep cleanup efficient.

---

### Redis-backed idempotency with outbox pattern
Use Redis `SET NX` as the distributed lock for fast key acquisition, combined with a transactional outbox to ensure the payment event is durably committed before the Redis key is marked complete.
 
```java
@Service
@RequiredArgsConstructor
public class RedisIdempotencyService {
 
    private final StringRedisTemplate redis;
    private final ObjectMapper objectMapper;
 
    private static final Duration LOCK_TTL    = Duration.ofMinutes(5);
    private static final Duration RESULT_TTL  = Duration.ofHours(24);
 
    public Optional<PaymentResponse> acquireLock(UUID clientId, String key) {
        String lockKey   = "idem:lock:%s:%s".formatted(clientId, key);
        String resultKey = "idem:result:%s:%s".formatted(clientId, key);
 
        Boolean acquired = redis.opsForValue()
                .setIfAbsent(lockKey, "processing", LOCK_TTL);
 
        if (Boolean.FALSE.equals(acquired)) {
            String cached = redis.opsForValue().get(resultKey);
            if (cached != null) {
                return Optional.of(deserialise(cached, PaymentResponse.class));
            }
            throw new ConcurrentRequestException(
                "Request with this key is already being processed"
            );
        }
 
        return Optional.empty(); // lock acquired — caller should process
    }
 
    public void storeResult(UUID clientId, String key, PaymentResponse response) {
        String lockKey   = "idem:lock:%s:%s".formatted(clientId, key);
        String resultKey = "idem:result:%s:%s".formatted(clientId, key);
 
        redis.opsForValue().set(resultKey, serialise(response), RESULT_TTL);
        redis.delete(lockKey);
    }
}
```
 
**Fits:** High-throughput systems where database lock contention is a measurable concern.
 
**Limitations:** Redis is not durable by default. A crash between payment processing and result storage can still yield duplicates. Use Redis with AOF persistence and pair with a reconciliation backstop. Do not use this as your only durability layer.

---

### Idempotency at the PSP level
Most major PSPs support idempotency keys natively. Idempotency key is passed downstream for PSP and they guarantee deduplication on their side.
 
```java
// Stripe Java SDK example
RequestOptions options = RequestOptions.builder()
        .setIdempotencyKey(idempotencyKey)
        .build();
 
PaymentIntentCreateParams params = PaymentIntentCreateParams.builder()
        .setAmount(request.getAmountInPence())
        .setCurrency("gbp")
        .build();
 
PaymentIntent intent = PaymentIntent.create(params, options);
```
 
**Fits:** Systems where the PSP is the authoritative source of truth.
 
**Limitations:** PSP-level idempotency protects only the downstream leg, between the service and the PSP. The upstream leg, between the client and the service still remains your responsibility entirely. Both legs require independent protection.
 
---

## What I'd Recommend
 
For most teams: **database-backed idempotency, with the payment and key committed within a single `@Transactional` boundary, scoped per client, with a 24-hour expiry window.**
 
> **_NOTE:_** Do not rely solely on PSP-level idempotency. It protects only one leg of the flow. Also, do not use Redis as your only idempotency store without understanding its durability characteristics under failure and having a reconciliation backstop in place.
 
> **_NOTE:_** The `PESSIMISTIC_WRITE` lock is not optional. Without it, concurrent retries arriving simultaneously will both pass the existence check and both proceed to charge. This surfaces under real load, not in unit tests.
 
> **_NOTE:_** Make the `Idempotency-Key` header required at the API layer and reject requests without it. The most common real-world failure is clients calling `UUID.randomUUID()` on each retry, which defeats the entire mechanism. Your API should make the correct pattern the only available pattern.
 
---

## Further Reading
 
- [Stripe: Idempotent Requests](https://stripe.com/docs/api/idempotent_requests): The gold standard for communicating the idempotency contract to API consumers.
- [Brandur Leach: Implementing Stripe-like Idempotency Keys in Postgres](https://brandur.org/idempotency-keys): The most thorough technical treatment of database-backed idempotency available publicly.
- [Baeldung: Idempotent REST API in Spring](https://www.baeldung.com/rest-api-idempotence): A Java/Spring-focused walkthrough, useful for onboarding engineers to the concept.
- [Designing Data-Intensive Applications: Chapter 9](https://dataintensive.net/): Kleppmann's treatment of exactly-once semantics and what distributed systems can and cannot guarantee. The theoretical foundation.
 
---
