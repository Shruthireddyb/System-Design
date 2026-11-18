# Parking Lot System - LLD

Java | LLD | Design Patterns

## Problem Statement
Design a Parking Lot System for a 3-floor mall with 500 spots. Supports CAR, BIKE, TRUCK, ELECTRIC vehicles. Multiple Entry/Exit gates. Handle concurrent entries.

## Requirements

**Functional:**
- Park vehicle based on type (Bike can't park in Truck spot)
- Generate Ticket with entry time & spot
- Calculate fee on exit based on duration
- Display Board shows free spots per floor
- Support different payment methods (Cash, Card)

**Non-Functional:**
- Thread-safe for concurrent entry
- Extensible for new vehicle types and payment methods

##  Design Decisions

| # | Design Decision | Options Considered | Chosen Approach | Reason / Trade-off |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Vehicle -> Spot Mapping** | 1. One spot type per vehicle 2. Flexible mapping | **Flexible Mapping** (Car can use Large spot) | Better utilization, less fragmentation. Large spot wasted if only large vehicles allowed |
| **2** | **Spot Allocation Strategy** | 1. Nearest First 2. Random 3. Best Fit | **Nearest to Entry + Best Fit** | Improves user experience + reduces search time. Uses `PriorityQueue` / `TreeMap` for O(log N) |
| **3** | **Floor vs Single Lot** | 1. Flat List of Spots 2. Floors containing Spots | **Floor -> Row -> Spot Hierarchy** | Scalable to multi-storey. Easy to add floor. Mirrors real world |
| **4** | **Ticket Generation** | 1. Ticket at Entry 2. Ticket at Spot | **Ticket at Entry Gate** | Simple flow, entry time = start time. Ticket holds `entryTime, vehicle, spotId` |
| **5** | **Payment Calculation** | 1. Fixed Price 2. Hourly + Vehicle Type | **Strategy Pattern for Pricing** | `HourlyPricingStrategy`, `EVSurchargeStrategy`. Easy to extend dynamic pricing |
| **6** | **Concurrency Control** | 1. Synchronized method 2. Lock per Floor 3. DB Lock | **ReentrantLock per Floor** | 1000 spots - locking whole lot is bottleneck. Floor-level lock gives throughput |
| **7** | **Data Structure for Free Spots** | 1. HashMap 2. List Scan O(N) 3. PriorityQueue / TreeMap | **Map<VehicleType, PriorityQueue<Spot>>** | O(log N) allocation, O(log N) release. HashMap = O(N) scan |
| **8** | **EV Spot Handling** | 1. Separate Lot 2. Same Lot with Flag | **Spot Type = EV_CHARGING extends Large** | EV can use large spot but not vice-versa. Single pool logic |
| **9** | **Entry/Exit Gates** | 1. Single Gate 2. Multiple Gates | **Multiple Gates + Observer** | Gates notify ParkingLot central. No tight coupling |
| **10** | **Persistence** | 1. In-Memory 2. DB | **In-Memory for LLD, Repository pattern for HLD** | LLD interview = in-memory + Interface `ParkingRepository` for future DB switch |
| **11** | **Design Patterns Used** | - | **Factory, Strategy, Singleton, Observer** | Factory: Vehicle creation, Strategy: Pricing & Allocation, Singleton: ParkingLot, Observer: Display board |
| **12** | **Payment Location** | 1. At Entry 2. At Exit 3. At Floor | **At Exit Gate (Pay and Exit)** | Real-world flow, allows hourly calc |

 "For production replace Repository with DB + Redis GEO"

## Concurrency Handling

### Problem
- 2 cars try to book same spot at same time
- 2 entries at same time -> double booking / overcount

### Solution

**1. DB Level - Pessimistic Locking**
- Use `SELECT ... FOR UPDATE` on spot row
- One transaction locks spot, other waits
- Use for: `bookSpot()`

**2. Optimistic Locking**
- Add `@Version` column in Spot table
- If version changed, throw `OptimisticLockException` -> Retry
- Use for: High read, low conflict

**3. Distributed Lock - Redis**
- `SET spot:101 NX PX 5000` - Only 1 server gets lock
- Release with Lua script - only owner can unlock
- Use for: Multi-instance deployment

**4. Atomic Counter - Entry Gate**
- `AtomicInteger` / Redis `INCR` / `DECR` for available slots
- Never do `read -> +1 -> write` in code - race condition

**5. Idempotency**
- Client sends `Idempotency-Key: UUID`
- Same key + same request = return same ticket, no duplicate booking

**6. Queue**
- Booking requests go to Kafka Queue
- Process 1 by 1 in order - no race

### What We Used
- `FOR UPDATE` + `@Version` for Spot booking
- Redis Lock for Payment + Booking
- Idempotency Key for Retry safety
