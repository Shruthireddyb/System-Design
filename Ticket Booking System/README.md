#  Ticket Booking System 

> A high-concurrency, distributed ticket booking platform handling 10K+ concurrent users with zero double-booking. Inspired by BookMyShow & IRCTC architecture.

##  1. Requirements

### 1.1 Stakeholders
- **End Users:** Public booking tickets
- **Admin:** Event organizers (Movie, Train, Concert)
- **External:** Payment Gateway, Email/SMS Service

### 1.2 Functional Requirements(FR)

**FR-1: User Management**
- User can Register/Login via Email/Phone + OTP
- User can view booking history
- Guest booking allowed

**FR-2: Event Management (Admin)**
- Admin can create Event with city, date, venue, total seats
- Admin can define Seat Layout (A1, A2..), Seat Type (Premium/Gold/Silver)
- Admin can set Price per seat type
- Admin can Cancel/Reschedule event

**FR-3: Search & Discovery**
- Search by City, Date, Event Name
- Filter by Price, Seat Type
- Show real-time available seats count
- Latency < 300ms

**FR-4: Seat Selection **
- Real-time seat map: Available (Green), Locked (Yellow), Booked (Red)
- User can select 1 to 6 seats per transaction
- System LOCKS seats for 10 minutes on selection (payment window)
- Auto-release if not paid in 10 min
- Locked seats visible to all users in real-time via WebSocket

**FR-5: Booking & Payment**
- Payment only for locked seats
- **Must prevent double booking** - 2 users cannot book same seat
- Mock Payment Gateway (Razorpay/Stripe)
- Idempotency Key - Prevents double charge on retry/click
- On success: Generate Booking ID / PNR
- On failure: Release lock instantly

**FR-6: Cancellation & Refund**
- Cancel before cutoff time (2 hours before event)
- Refund logic: 100%, 50%, 0% based on time
- Cancelled seat becomes instantly available

**FR-7: Waitlist - IRCTC Model**
- If housefull, user can join waitlist (max 50)
- On cancellation, auto-upgrade first waitlisted user + notify

**FR-8: Notifications**
- Email/SMS on Confirmation, Cancellation, Waitlist Upgrade

### 1.3 Non-Functional Requirements (NFR)

**NFR-1: Concurrency**
- Handle 10,000 concurrent users booking same event (IPL Final)
- Strict Consistency: If 100 seats, max 100 bookings. No overselling.

**NFR-2: Availability - 99.9%**
- Search should work even if Payment service is down

**NFR-3: Performance**
- Search Events: < 200ms p95
- Seat Map Load: < 300ms p95
- Lock API: < 500ms p95
- Target: 5,000 TPS at peak

**NFR-4: Scalability**
- Booking Service stateless -> Run N instances behind Load Balancer
- DB sharding ready for 1M+ events

**NFR-5: Security**
- Rate Limiting: 100 req/min per user (prevent bots)
- No access to other user's bookings

**NFR-6: Durability**
- Booking data never lost - ACID DB

---
**Flow:**
1. `POST /seats/lock` -> Redis SET `lock:event:seat_id` NX with 10min TTL
2. WebSocket broadcasts `seat_locked` to all users viewing that event
3. `POST /bookings` -> Check Redis lock + DB Transaction `SELECT ... FOR UPDATE`
4. Payment success -> DB `bookings` confirmed + Redis invalidated + Kafka `booking.confirmed`
5. Payment fail/timeout -> Redis lock auto-expires

---
##  High Level Architecture (HLD)
-  React Client 
      
-  API Gateway - Nginx + Rate Limiter 
      
-  Microservices: User | Event | Booking (Core) | Payment 
      
-  PostgreSQL (ACID)  +  Redis (Locks + Cache)  +  Kafka (Events) 
      
-  WebSocket Server - Pushes real-time seat status to all clients 

---
