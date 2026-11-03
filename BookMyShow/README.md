# BookMyShow - LLD 
### 1. Overview
A distributed ticket booking system that handles:
- City-wise movie & theatre listing
- Real-time seat selection with 10-min hold
- No double booking under high concurrency
- Payment & notification flow

Built to demonstrate LLD, HLD, and concurrency patterns asked in Flipkart, Swiggy, PhonePe interviews.

### 2. Key Features
- [x] Search movies by city
- [x] List shows by movie & theatre
- [x] Get seat map with status (AVAILABLE / BLOCKED / BOOKED)
- [x] Block seats for 10 mins (Redis TTL)
- [x] Confirm booking after payment
- [x] Auto-release blocked seats on timeout (Kafka Delayed Queue)
- [x] Optimistic locking to prevent double booking

### 3. Requirements 
Functional:
- Search movies by city
- List theatres + showtimes for a movie
- Select seats, hold for 10 mins (payment timeout)
- Book + paymentCancel (if allowed)

Non-Functional:
- Concurrency: 2 users should NOT book same seat
- Low latency seat map
- No overbooking
- High read, low write

### Design Decisions

| Decision Area | Decision Taken | Alternatives Considered | Why This Choice? | Trade-off |
| :--- | :--- | :--- | :--- | :--- |
| **Architecture** | Microservices (User, Movie, Theatre, Booking, Payment) | Monolith | Independent scaling, team autonomy. Booking service needs 10x scale during sale | Complexity of inter-service communication |
| **Database per Service** | MySQL for Booking, MongoDB for Movie Catalog | Single shared DB | Booking needs ACID, Movie needs flexible schema | Data consistency needs Saga pattern |
| **Concurrency Handling** | Pessimistic Locking + Redis Distributed Lock | Optimistic Locking | Prevent double booking for same seat - Money critical, can't allow conflict retry | Slight throughput hit, but 100% correctness |
| **Seat Hold Mechanism** | Redis with TTL (10 min hold) | DB status column only | Fast in-memory check, auto-expiry if user doesn't pay | Need to sync Redis -> DB after payment |
| **API Gateway** | Spring Cloud Gateway | No Gateway / Zuul | Rate limiting, Auth, Routing in one place | Single point of failure - mitigate with replica |
| **Communication** | Sync: REST + Feign, Async: Kafka for Booking -> Payment -> Notification | Only REST | Payment notification shouldn't block booking | Eventual consistency |
| **Caching** | Redis for Movies, Theatres, ShowTimes | No Cache / EhCache | Read heavy - Movie list read 1000x more than write | Cache invalidation complexity |
| **Search** | Elasticsearch for Movie/Theatre search | DB LIKE query | Fast fuzzy search, filters (language, genre) | Extra infra cost |

### Database Schema Decisions

| Table | Key Decision | Reason |
| :--- | :--- | :--- |
| **Show_Seat** | Separate table for each show, not just Seat | Same physical seat can be BOOKED in Show1 and AVAILABLE in Show2 |
| **Booking Status** | PENDING -> CONFIRMED -> CANCELLED (State Machine) | Track payment failure, auto-release seat if PENDING > 10 min |
| **Seat Price** | Price in Show_Seat, not Seat | Same seat price differs for Morning vs Night show, Weekend |
| **Indexes** | Composite Index on (show_id, status) | Query `findAvailableSeatsByShowId` is 95% of traffic |
| **Idempotency Key** | booking table has idempotency_key | User double-clicks PAY - prevent double booking |
