# Food Delivery Platform 

> A scalable, microservices-based food delivery system like Swiggy / Zomato / DoorDash

## 1. Overview

This project is a complete system design for a high-scale food delivery platform handling 1M+ orders per day. It covers architecture, data flow, APIs, and scalability challenges.

**Key Features:**
- User Management & Authentication
- Restaurant Listing & Menu Management
- Real-time Order Tracking
- Payment Integration
- Delivery Partner Assignment & Tracking
- Ratings & Reviews
- Notifications & Search

## 2. Requirements

### Functional Requirements
- Users can browse restaurants, view menus, place orders
- Restaurants can manage menu, accept/reject orders
- Delivery partner can accept deliveries and update status
- Real-time order status: Placed -> Confirmed -> Preparing -> Out for Delivery -> Delivered
- Payment gateway integration (UPI, Cards, COD)
- Search and filters by cuisine, price, ratings

### Non-Functional Requirements
- **Availability:** 99.9% uptime
- **Latency:** < 200ms for search, < 500ms for order placement
- **Scalability:** Handle 10k concurrent users, 1M orders/day
- **Consistency:** Strong consistency for orders & payments, eventual for ratings
- **Security:** JWT auth, data encryption

## Design Decisions

| Decision / Pattern | Used In Food Delivery For | Why This Pattern / Choice? | SOLID / Design Principle |
| :--- | :--- | :--- | :--- |
| **Strategy** | Pricing - Normal, Surge, Discount, Coupon, Festive Offer | Each pricing logic is different but same interface `calculatePrice()`. Add new offer without changing old code | **OCP** - Open for Extension, Closed for Modification |
| **Strategy** | Delivery Assignment - Nearest Driver, Fastest Driver, Least Busy Driver | Assignment algorithm changes by city / time, switch at runtime | **OCP + SRP** |
| **Factory** | Creating Order - `OrderFactory.create(type)` - Food, Grocery, Parcel | Client doesn't know which Order class to create, Factory decides by type | **SRP + OCP** - Creation logic in one place |
| **Factory** | Payment - UPI, Card, COD, Wallet - `PaymentFactory.create(method)` | Add new payment method without changing checkout code | **OCP** |
| **Singleton** | DB Connection, Redis Client, Config Manager, Logger | Only one connection pool / config should exist for whole app | **SRP** - One instance controls one shared resource |
| **Observer** | Order Tracking - Order Placed, Preparing, Picked Up, Delivered | One order event notifies many - User App, Driver App, Restaurant Dashboard, Push Service | **OCP + SRP** - Sender and receivers decoupled |
| **Builder** | Building `Order` - Items, Address, Coupon, Instructions, Tip | Order has 10+ optional fields, avoid `new Order(a,b,c,d,e,f,null)` | **SRP** - Construction separate from representation |
| **Adapter** | Payment Gateway - Razorpay, Stripe, Paytm + Maps - Google Maps, OSM | Our code uses one `PaymentService` / `MapService` interface, Adapter converts to vendor format | **OCP + DIP** - Depend on abstraction, not concrete API |
| **Decorator** | Order Features - Discount + Free Delivery + Priority Delivery | Add features dynamically like `Discounted(Priority(Order))` without changing base Order | **OCP + SRP** - Each feature in its own class |
| **Repository** | `RestaurantRepository`, `OrderRepository`, `DriverRepository` | Separate DB logic from business logic, easy to switch PostgreSQL to MongoDB | **SRP + DIP** - Business logic doesn't depend on DB |
| **Facade** | `OrderFacade.placeOrder()` | One method hides - Validate, Check Stock, Payment, Assign Driver, Notify | **SRP** - Client sees one simple interface |
| **Command** | Place, Cancel, Refund, Reorder, Rate Order | Each action is an object - can be queued, retried, logged, undone | **OCP + SRP** - New action = new class |
| **Microservices** | User, Restaurant, Order, Payment, Delivery, Notification Services | Scale Order and Delivery independently during lunch / dinner peak | **SRP** - One service, one responsibility |
