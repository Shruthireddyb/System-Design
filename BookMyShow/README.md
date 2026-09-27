# BookMyShow - System Design LLD 
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

### 4. LLD - Class Design

#### Core Entities
```
City -> Theatre -> Screen -> Seat
Movie -> Show -> ShowSeat (Join table - Most Important)
User -> Booking -> Payment
