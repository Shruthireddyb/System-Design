# URL Shortener - HLD 

> Design a system like TinyURL / Bit.ly - 100M writes/month, 10B reads/month

### 1. Requirements

**Functional:**
- POST long_url -> short_url `sho.rt/abc123`
- GET short_url -> 302 redirect to long_url
- Custom alias + Expiry support
- Analytics: click count

**Non-Functional:**
- Low latency: <100ms for redirect
- High availability: 99.9%
- Read-heavy 100:1 ratio
- Non-guessable short codes

### 5. Design Decisions

| Pattern | Principle | Why Used in URL Shortener? | What Problem It Solves | Benefit |
| :--- | :--- | :--- | :--- | :--- |
| **Strategy** | **OCP** - Open/Closed | `EncodingStrategy` interface for Base62, Base58, Hashids, UUID | If-else hell when adding new encoding algo. Modifying service every time. | Add new algorithm without touching existing service code. Just add new class + inject. |
| **Factory** | **SRP** - Single Responsibility | `ShortUrlFactory` handles validation, expiry, code generation | Controller/Service doing object creation + validation + DB save = multiple reasons to change. | Creation logic isolated. Service only orchestrates. Controller stays thin. |
| **Singleton** | **Single Instance + Thread Safety** | `SnowflakeIdGenerator`, `Base62Config` as Spring Singleton Bean | Multiple generator instances = duplicate IDs. Recreating char map per request = memory waste. | One thread-safe instance across app. No duplicates. Memory efficient. |

### 6. How They Work Together

| Layer | Pattern | Responsibility |
| :--- | :--- | :--- |
| Controller | None | Only HTTP handling |
| Service | Uses Factory + Strategy | Orchestrates: factory.create() -> strategy.encode() -> repo.save() -> redis.set() |
| Factory | SRP | Creates `ShortUrl` object |
| Strategy | OCP | Decides HOW to encode |
| Generator | Singleton | Generates unique ID once |

## Concurrency Handling

## The Problem
1000 users click "Shorten" at same time for same long URL.
10000 users click same short URL at same time.
If we don't handle concurrency, we get - Duplicate short codes, Lost click count, Cache stampede, DB overload.

| Concurrency Problem | Where It Happens | Solution Used | How It Works | Why This Choice? |
| :--- | :--- | :--- | :--- | :--- |
| **1. Duplicate Short Code Generation** | Two requests generate same `abc123` at same time, both try to insert | **Unique Constraint + Retry + Pre-generated Codes** | DB column `short_code` is UNIQUE. If insert fails with duplicate key, retry with new code. Better: Pre-generate 1M codes in Redis / DB table and use `Atomic POP` | **Correctness** - DB is final truth, no two URLs get same code |
| **2. Same Long URL Shortened Twice** | User A and User B shorten same `youtube.com/...` at same time, creates 2 different short codes | **Unique Constraint on Long URL Hash + Idempotency Key** | `long_url_hash = SHA256(long_url)` is UNIQUE. First insert wins, second gets existing code. Client sends `Idempotency-Key = userId + longUrlHash` | **Deduplication** - Same URL = Same short code, saves storage |
| **3. Lost Update - Click Count** | 10000 users click `bit.ly/abc` at same time. All read count=100, all write 101. Real count should be 10100, but becomes 101 | **Atomic Counter - Redis INCR** | Don't do `read + 1 + write` in DB. Do `Redis INCR click:abc` - Redis is single-threaded, atomic. Flush to DB every 1 min / every 1000 clicks | **Performance + Correctness** - 100K INCR/sec, no lost update |
| **4. Cache Stampede / Thundering Herd** | Short code expires from cache, 10000 requests hit DB at same time for same code | **SingleFlight / Redis Lock + Cache Warming** | First request gets lock, loads from DB to cache. Others wait 50ms and read from cache. Or use `Cache-Aside + Logical Expiry` - Serve stale cache, refresh in background | **Protect DB** - DB gets 1 query, not 10000 |
| **5. Race in Custom Alias** | Two users want custom alias `my-link` at same time | **DB Unique Constraint + Redis SETNX** | `SETNX custom:my-link` in Redis - Only one wins. If fails, return "Alias already taken". DB UNIQUE as backup | **Correctness** - First come, first serve |
| **6. Counter / ID Generation Bottleneck** | Using DB Auto-Increment for ID is slow, single point of failure, not distributed | **Snowflake ID / Base62 Counter + Redis Atomic** | Use Snowflake - `timestamp + machineId + sequence` - No coordination needed. Or Redis `INCR global_counter` then encode to Base62 | **Scalability** - Generate 10K IDs/sec across 10 servers, no lock |
| **7. Concurrent Expiry / Delete** | User deletes link while 100 users are clicking it | **Soft Delete + Version Check** | Don't delete row. Set `is_active=false, deleted_at=now`. Redirect service checks `is_active`. Use `Optimistic Locking - version` column | **Consistency** - In-flight clicks finish, new clicks get 404 |
| **8. Rate Limit Race** | User sends 1000 requests in 1 sec from 10 servers, each server thinks count=10 | **Redis Token Bucket - Atomic Lua Script** | Rate limit check + decrement in ONE Lua script in Redis. `EVAL` is atomic. Use `Sliding Window` in Redis Sorted Set | **Correctness Across Servers** - Shared Redis = Single source of truth |
| **9. Analytics Write Contention** | Every click writes to `clicks` table - DB lock, slow redirect | **Async Write - Kafka + Batch Insert** | Redirect service only does `INCR` + `Kafka produce click event`. Analytics service consumes and batch inserts 1000 rows at once to ClickHouse / Cassandra | **Performance** - Redirect stays <10ms, analytics not in critical path |
| **10. Double Redirect / Duplicate Click** | User double-clicks, browser retries, counts 2 clicks for 1 user | **Deduplication - Redis Bloom Filter / Idempotency** | Use `userId + shortCode + 1-sec window` as dedup key in Redis. If exists, don't count again. Or use Kafka idempotent producer | **Accuracy** - Analytics not inflated |
| **11. Distributed Lock for Critical Section** | Only one server should generate code for custom alias / cleanup expired links | **RedLock / Redis SET NX PX** | `SET lock:alias:my-link NX PX 5000` - Who gets lock, does work. Release with Lua script checking owner token. Use for cleanup job only, not for every request | **Safety** - Avoid dead lock with TTL |
| **12. Read-Your-Write / Cache Inconsistency** | User creates short URL, immediately clicks, cache misses, gets 404 | **Write-Through Cache + Cache-Aside** | On create, write to DB AND write to Redis cache immediately. On read, read from Redis first. On delete, delete from both | **Consistency** - User sees own write instantly |
| **13. Database Connection Pool Exhaustion** | 10000 concurrent redirects open 10000 DB connections | **Connection Pool - HikariCP (20 connections) + Cache + Read Replica** | 95% redirects served from Redis, only 5% hit DB. Pool size = ` (core * 2) + disk`. Use Read Replica for redirect reads | **Scalability** - DB never overloaded |
| **14. Optimistic Locking for URL Update** | Two admins update same URL destination at same time | **Version Column** | `UPDATE urls SET dest=?, version=version+1 WHERE id=? AND version=?` If 0 rows updated, retry - someone else updated | **No Lost Update** - Last writer doesn't silently overwrite |
| **15. Load Balancer + Stateless Service** | Same user request goes to different servers | **Stateless Service + Shared Redis + Shared DB** | No in-memory counter / map in service. All shared state in Redis / DB. Any server can handle any request | **Horizontal Scaling** - Add 10 servers, no sync needed |
