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
