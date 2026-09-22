# Parking Lot System - Low Level Design

Java | LLD | Design Patterns

## 📌 Problem Statement
Design a Parking Lot System for a 3-floor mall with 500 spots. Supports CAR, BIKE, TRUCK, ELECTRIC vehicles. Multiple Entry/Exit gates. Handle concurrent entries.

## 🎯 Requirements

**Functional:**
- Park vehicle based on type (Bike can't park in Truck spot)
- Generate Ticket with entry time & spot
- Calculate fee on exit based on duration
- Display Board shows free spots per floor
- Support different payment methods (Cash, Card)

**Non-Functional:**
- Thread-safe for concurrent entry
- Extensible for new vehicle types and payment methods

## 🧠 Design Decisions - Why I Designed Like This

### Why Strategy Pattern for Spot Assignment?
**Problem:** Nearest spot today, but tomorrow mall wants EV priority, VIP priority.
