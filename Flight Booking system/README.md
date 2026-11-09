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

---
## Design Decisions

| Decision / Pattern | Used In Flight Booking For | Why This Pattern / Choice? | SOLID / Design Principle |
| :--- | :--- | :--- | :--- |
| **Strategy** | Pricing - Base Fare, Dynamic Pricing, Surge, Promo Code, Student / Senior Discount | Each pricing logic is different but same interface `calculateFare()`. Add new offer without changing old code | **OCP** - Open for Extension, Closed for Modification |
| **Strategy** | Flight Search - Cheapest, Fastest, Shortest Layover, Earliest Departure | Search algorithm changes by user filter, switch at runtime | **OCP + SRP** |
| **Factory** | Creating Booking - `BookingFactory.create(type)` - One-Way, Round-Trip, Multi-City | Client doesn't know which Booking class to create, Factory decides by type | **SRP + OCP** - Creation logic in one place |
| **Factory** | Payment - UPI, Card, NetBanking, Wallet, EMI - `PaymentFactory.create(method)` | Add new payment method without changing checkout code | **OCP** |
| **Singleton** | DB Connection, Redis Client, Config Manager, Logger, Seat Lock Manager | Only one connection pool / config should exist for whole app | **SRP** - One instance controls one shared resource |
| **Observer** | Booking Status - Booked, Payment Success, Payment Failed, Cancelled, Flight Delayed, Check-in Open | One booking event notifies many - User App, Email, SMS, Push, Airline System | **OCP + SRP** - Sender and receivers decoupled |
| **Builder** | Building `Booking` - Passenger List, Seat, Meal, Baggage, Insurance, Coupon | Booking has 10+ optional fields, avoid `new Booking(a,b,c,d,e,f,null)` | **SRP** - Construction separate from representation |
| **Adapter** | Airline API - IndiGo, Air India, SpiceJet + Payment Gateway - Razorpay, Stripe | Our code uses one `FlightProvider` / `PaymentService` interface, Adapter converts to vendor format | **OCP + DIP** - Depend on abstraction, not concrete API |
| **Decorator** | Booking Add-ons - Extra Baggage + Meal + Priority Boarding + Travel Insurance | Add features dynamically like `Insurance(Meal(Booking))` without changing base Booking | **OCP + SRP** - Each feature in its own class |
| **Repository** | `FlightRepository`, `BookingRepository`, `UserRepository`, `PassengerRepository` | Separate DB logic from business logic, easy to switch PostgreSQL to MongoDB | **SRP + DIP** - Business logic doesn't depend on DB |
| **Facade** | `BookingFacade.bookFlight()` | One method hides - Search, Seat Lock, Payment, Confirm PNR, Notify | **SRP** - Client sees one simple interface |
| **Command** | Book, Cancel, Reschedule, Refund, Web Check-in, Add Baggage | Each action is an object - can be queued, retried, logged, undone | **OCP + SRP** - New action = new class |
| **Saga Pattern** | Distributed Booking - Flight Hold -> Payment -> Confirm Seat -> Issue Ticket | If Payment fails, Saga compensates - Release Seat, Refund, Cancel PNR. No partial booking | **Reliability** - No distributed transaction needed |
| **Microservices** | User, Flight Search, Booking, Payment, Notification, Pricing Services | Scale Search 10x during festival sale, Booking stays stable | **SRP** - One service, one responsibility |
