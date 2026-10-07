#  Flight Booking System

A scalable, transaction-safe flight booking platform with PNR generation, seat locking, and real-time inventory management. Designed for high concurrency like IRCTC / MakeMyTrip.


---

## 📋 REQUIREMENTS

### 1. Functional Requirements

**FR1 - User Management**
- FR1.1: User registration with email verification
- FR1.2: Login with JWT authentication
- FR1.3: View booking history by user ID
- FR1.4: Password reset via email

**FR2 - Flight Search & Discovery**
- FR2.1: Search flights by Source, Destination, Date, Class, Passengers
- FR2.2: Filter by Airline, Price range, Duration, Stops [Direct/1-stop]
- FR2.3: Sort by Price, Duration, Departure time
- FR2.4: Real-time seat availability count
- FR2.5: Show aircraft type and baggage allowance

**FR3 - Booking & PNR**
- FR3.1: Interactive seat map [Business/Economy layout]
- FR3.2: Seat lock for 10 minutes after selection [prevent double booking]
- FR3.3: Add multiple passengers under single PNR
- FR3.4: PNR generation: Format `FLYYYYMMDDXXXXXX` [10-digit alphanumeric]
- FR3.5: E-ticket generation with QR code
- FR3.6: Fare calculation: Base fare + taxes + convenience fee

**FR4 - Payment**
- FR4.1: Mock payment integration [Razorpay / Stripe test mode]
- FR4.2: Payment timeout - auto release seat if not paid in 10 mins
- FR4.3: Refund initiation on cancellation

**FR5 - Cancellation & Modification**
- FR5.1: Cancel booking via PNR
- FR5.2: Refund calculation: 100% if >24h, 50% if 2-24h, 0% if <2h
- FR5.3: Email notification on cancellation

**FR6 - Admin Module**
- FR6.1: Admin login [Role-based access]
- FR6.2: CRUD operations for Flights, Airports, Aircrafts
- FR6.3: Dynamic pricing update based on demand
- FR6.4: View passenger manifest per flight
- FR6.5: Revenue dashboard and load factor report
- FR6.6: Delay / Schedule change notification to passengers

### 2. Non-Functional Requirements

**NFR1 - Performance**
- Search response < 200ms for 10k flights
- Support 500+ concurrent users
- Seat lock operation < 100ms

**NFR2 - Security**
- Password hashing with bcrypt [cost 10]
- JWT expiry 24h, refresh token 7 days
- SQL injection prevention via prepared statements
- Rate limiting: 100 requests/min per IP

**NFR3 - Reliability & Consistency**
- Zero double-booking: MySQL `SELECT... FOR UPDATE` + transactions
- ACID compliance for booking table
- PNR uniqueness enforced at DB level [UNIQUE constraint]
- 99.9% uptime target

**NFR4 - Scalability**
- Horizontal scaling ready [Stateless API]
- Database indexing on `source, destination, departure_date`
- Future: Redis caching for search results

**NFR5 - Usability**
- Mobile responsive [Tailwind CSS]
- PNR lookup without login
- Simple 3-step flow: Search -> Select -> Pay

### 3. System Constraints
- MySQL 8.0+ for transactions
- No real payment - use Stripe test keys only
- Single airport code standard: IATA [HYD, BLR, DEL]

---
