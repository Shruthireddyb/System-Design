## Parking Lot System - LLD

### Problem: Design parking lot for 500 spots, 3 floors, supports CAR, BIKE, TRUCK, ELECTRIC

### Design Decisions (Interviewers read this)
1. **Why Strategy Pattern for Spot Assignment?**
   - NearestSpotStrategy for normal users, EmergencyPrioritySpotStrategy for ambulance/police
   - Easy to add new strategy without changing EntryGate code - OCP principle

2. **Why Strategy for Payment?**
   - CardPaymentStrategy, CashPaymentStrategy, UPI later
   - PaymentService doesn't care about payment type

3. **Thread Safety:** Used ConcurrentHashMap for parkingSpots map to handle 2 cars entering same time

### Class Diagram: docs/class-diagram.jpg
### Flow: Vehicle -> EntryGate -> SpotAssignmentStrategy -> ParkingSpot -> Ticket -> ExitGate -> PaymentService

### Code Highlights
- Enums: VehicleType, SpotStatus
- Composition: ParkingLot has Floors has Spots
- Factory: TicketFactory generates ticket with entryTime