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
