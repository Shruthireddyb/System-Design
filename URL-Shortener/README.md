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

### 2. Capacity Estimation

- **Write QPS:** 100M/month = 40 req/s
- **Read QPS:** 10B/month = 4000 req/s
- **Storage:** 100M * 500 bytes = 50GB/month ~ 600GB/year
- **Cache:** 20% hot URLs = 800 QPS, ~10GB Redis needed
- **Bandwidth:** Read 4K * 500 bytes = 2MB/s

### 3. API Design

```http
POST /api/v1/shorten
Body: { "long_url": "https://example.com/very/long/url", "custom_alias": "my-link", "expiry_days": 7 }
Response: 201 { "short_url": "https://sho.rt/abc123", "long_url": "...", "expiry_at": "..." }

GET /{short_code} -> 302 Redirect
Location: https://example.com/very/long/url

GET /api/v1/stats/{short_code} -> { "clicks": 120, "created_at": "..." }
```
### 4. Database Design

**MySQL - Main Mapping Table**
```sql
CREATE TABLE url_mapping (
  id BIGINT PRIMARY KEY, -- Snowflake ID
  short_code VARCHAR(10) UNIQUE INDEXED,
  long_url TEXT NOT NULL,
  user_id BIGINT,
  created_at TIMESTAMP,
  expiry_at TIMESTAMP,
  click_count BIGINT DEFAULT 0
) PARTITION BY HASH(id);
```
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
