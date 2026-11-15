# Rate Limiter 

> Distributed Rate Limiter using Token Bucket, Fixed Window, Sliding Window Log & Sliding Window Counter. Handles 1M req/sec with Redis + Lua Script. Used to protect Feed, URL Shortener, BookMyShow APIs.


### 1. Requirements

#### A. Functional Requirements
1. **Limit Requests:** Allow max N requests per time window per user/IP/API
   - Ex: 100 requests / minute / user
   - Ex: 10 requests / second / IP
2. **Multiple Rules:** Support different limits for different APIs
   - `/api/feed`: 100 req/min per user
   - `/api/posts`: 10 req/min per user
   - `/api/login`: 5 req/min per IP (brute force protection)
3. **Response Headers:** Return `X-RateLimit-Remaining`, `X-RateLimit-Retry-After`
4. **Block:** Return `429 Too Many Requests` when limit exceeded

#### B. Non-Functional Requirements
1. **Low Latency:** Rate limit check < 5ms (p99) - should not slow down API
2. **Distributed:** Works across 10+ server instances - need centralized store (Redis)
3. **High Availability:** Rate limiter failure should NOT block API traffic (fail open)
4. **High Throughput:** Support 1M req/sec check
5. **Accurate:** No race condition - 2 concurrent requests shouldn't both pass when only 1 token left

#### C. Out of Scope
- Billing based on rate limit tiers (Stripe) - separate service
- DDoS protection at L7 (Cloudflare does it)
- User authentication - assumes userId/IP comes from Gateway

#### D. User Stories
```
- As a user, if I call /api/feed 101 times in 1 min, 101st should be blocked with 429
- As an attacker, if I try 1000 logins/sec from same IP, I should be blocked after 5 tries
- As a developer, I want different limits for /api/feed and /api/posts
- As a system, rate limiter check should be atomic across 10 servers
```
## Design Decisions 

### Pattern Decisions

| Pattern | Why Used in Rate Limiter? | Example Code | Benefit |
| :--- | :--- | :--- | :--- |
| **Strategy** | Support multiple rate limiting algorithms | `RateLimitStrategy` -> `FixedWindowStrategy`, `TokenBucketStrategy`, `SlidingWindowStrategy` | Add new algorithm (e.g., Leaky Bucket) without changing filter. **OCP** |
| **Factory** | Create strategy based on config/user tier | `RateLimiterFactory.getStrategy("PREMIUM_USER")` returns TokenBucket(1000 req/sec) | Creation logic isolated. **SRP**. Handles Free vs Paid tiers. |
| **Singleton** | Redis connection + Token Bucket refill thread | `RedisTemplate` as Singleton bean, `BucketRefiller` as single scheduler | One connection pool. No race in refill. Thread-safe. |
| **Chain of Responsibility** | Multiple limiters: IP -> User -> API | Request -> IP limiter -> User limiter -> Endpoint limiter | Each limiter has single reason. Can add/remove chain easily. |


| Layer | Responsibility |
| :--- | :--- |
| Filter / Interceptor | Extracts user/IP, calls Factory |
| Factory | Creates correct Strategy based on plan (Free=100/min, Pro=1000/min) |
| Strategy | Executes `isAllowed(key)` logic using Redis |
| Singleton Redis | Stores counters atomically |

>  "Used Strategy for OCP to support multiple algorithms, Factory for SRP to handle tier-based creation, and Singleton Redis with Lua for atomic distributed limiting."

## Concurrency Handling

### 1. Problem

| Issue | What Happens |
| :--- | :--- |
| **Race Condition** | 100 requests read count = 99 at same time, all pass |
| **Root Cause** | Check-then-Act is not atomic |
| **Rule** | Never do GET then INCR in app code |

### 2. Solutions

| Method | How It Works | Use When |
| :--- | :--- | :--- |
| **Redis INCR** | `INCR` is atomic, single-threaded | Simple, Fixed Window |
| **Lua Script** | Whole script runs atomically, no interleaving | Sliding Window, Token Bucket |
| **Mutex / Lock** | `sync.Mutex` in Go, `ReentrantLock` in Java | Single server only |
| **AtomicInteger** | `AtomicLong`, `LongAdder` | Single server, high contention |

### 3. Redis Code

| Type | Command |
| :--- | :--- |
| **Fixed Window** | `INCR key` + `EXPIRE key 60` |
| **Sliding Window** | `EVAL lua 1 key now window limit` |
| **Token Bucket** | `EVAL lua 1 key now rate capacity 1` |

### 4. Lua - Sliding Window

| Step | Command |
| :--- | :--- |
| **1. Clean Old** | `ZREMRANGEBYSCORE key 0 window_start` |
| **2. Count** | `ZCARD key` |
| **3. Allow?** | If count < limit, `ZADD key now now` = Allow, Else Block |

### 5. Lua - Token Bucket

| Step | Action |
| :--- | :--- |
| **1. Refill** | `tokens = min(capacity, tokens + elapsed * rate)` |
| **2. Check** | If `tokens >= 1`, deduct and Allow |
| **3. Else** | Block, update `last_refill` only |

### 6. Distributed Issues

| Problem | Solution |
| :--- | :--- |
| **Multi Server** | Central Redis, not local memory |
| **Hot Key** | Shard key: `limit:{user}:shard:1..10` |
| **Clock Skew** | Use Redis `TIME`, not app time |
| **Redis Down** | Fail Open for Feed, Fail Closed for Payment |
| **Latency** | Use `EVALSHA` + Connection Pool |

### 7. Performance

| Tip | Why |
| :--- | :--- |
| **Use EVALSHA** | Avoid sending Lua every time |
| **Use INCR** | 10x faster than Sorted Set |
| **Avoid KEYS *** | Blocks Redis, never in hot path |
| **Set Expiry** | Prevent key leak |

### 8. Testing

| Test | Expected |
| :--- | :--- |
| **200 Parallel, Limit 100** | Exactly 100 Allowed, 100 Blocked |
| **Tool** | `k6`, `Locust`, `pytest -n 10` |
| **Fail If** | Allowed > 100 = Race Condition |

### 9. Decision

| Use Case | Pick This |
| :--- | :--- |
| **Simple API** | INCR Fixed Window |
| **Accurate, No Burst** | Lua Sliding Window |
| **Burst Allowed** | Lua Token Bucket |
| **1 Server Only** | Mutex + Atomic |
| **Global High RPS** | Sharded Counters |
