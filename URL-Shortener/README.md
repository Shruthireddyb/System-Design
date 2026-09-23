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
