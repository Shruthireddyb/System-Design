# Ecommerce System Design

> Tech: Java 17, Spring Boot, Kafka, Redis, PostgreSQL, Elasticsearch, S3

## 1. Requirements

**Functional:**
- User registration/login, Browse products, Search, Add to cart, Wishlist
- Place order, Payment, Order tracking, Cancel/Return
- Seller: Add/update products, Inventory management
- Notification: Email/SMS on order update

**Non-Functional:**
- 10M users, 1M concurrent, 10K orders/sec peak (Big Billion Day)
- Search < 200ms, Checkout available 99.99%
- No overselling - Inventory consistency
- Idempotent payment


## 2. Flow for Order:
1. User -> Cart Service [Redis GET cart]
2. Checkout -> Order Service creates PENDING order
3. Publish `order-created` to Kafka
4. Inventory Service consumes -> `SELECT... FOR UPDATE` -> Deduct stock + Redis lock `SETNX inventory:prod_123`
5. Payment Service consumes -> Call payment gateway with Idempotency-Key
6. On success -> `payment-success` -> Order -> CONFIRMED -> `email-send` -> Notification Service

## Design Decisions

| Decision / Pattern | Used In E-commerce For | Why This Pattern / Choice? | SOLID / Design Principle |
| :--- | :--- | :--- | :--- |
| **Command** | Order - `PlaceOrderCommand`, `CancelOrderCommand`, `ReturnOrderCommand`, `RefundCommand` all implement `OrderCommand.execute()` | Each order action is an object - can be queued, retried, logged, undone, audited | **OCP + SRP** - New action = new class, no change in Order Service |
| **Strategy** | Payment Strategy - UPI, Card, NetBanking, COD, Wallet, EMI | Payment method changes at runtime, same checkout flow | **OCP** - Add new payment without changing Checkout |
| **Strategy** | Pricing Strategy - Regular Price, Discount, Coupon, Flash Sale, Membership Price | Price logic changes by offer, user type, festival | **OCP + SRP** |
| **Strategy** | Shipping Strategy - Standard, Express, Same Day, Store Pickup | Shipping cost / time logic separate per method | **OCP** |
| **Factory** | Payment Factory - `PaymentFactory.create(UPI / Card / COD)` | Client sends payment type, Factory returns correct Payment class | **SRP + OCP** - Creation logic in one place |
| **Factory** | Notification Factory - Email, SMS, Push, WhatsApp | Same order event creates different notification type | **OCP** |
| **Singleton** | DB Connection, Redis Client, Kafka Producer, Config Manager, Cart Manager | Only one instance should control shared resource, avoid duplicate connections | **SRP** - One instance controls one shared resource |
| **Observer** | Order Events - Order Placed, Payment Success, Order Shipped, Order Delivered | One event notifies many - Inventory, Email, SMS, Analytics, Seller Dashboard, Recommendation | **OCP + SRP** - Sender and receivers decoupled |
| **Builder** | Building `Order` - OrderId, UserId, Items, Address, Payment, Coupon, Shipping, Total | Order has 15+ fields, many optional, avoid `new Order(a,b,c,d...)` | **SRP** - Construction separate from representation |
| **Adapter** | Payment Gateway - Razorpay, Stripe, Paytm, PhonePe | Our code uses one `PaymentGateway` interface, Adapter converts to vendor format | **OCP + DIP** - Depend on abstraction, not vendor |
| **Adapter** | Shipping Provider - Delhivery, Ekart, BlueDart | Same interface, different provider APIs | **OCP + DIP** |
| **Decorator** | Product Price - Base Price + Discount + Coupon + Tax + Delivery Charge | Add features dynamically like `Tax(Discount(BasePrice))` without changing base | **OCP + SRP** - Each feature in its own class |
| **Repository** | `ProductRepository`, `OrderRepository`, `UserRepository`, `CartRepository` | Separate DB logic from business logic, easy to switch PostgreSQL to MongoDB | **SRP + DIP** - Business logic doesn't depend on DB |
| **Facade** | `CheckoutFacade.checkout()` | One method hides - Validate Cart, Check Stock, Apply Coupon, Process Payment, Create Order, Send Notification | **SRP** - Client sees one simple interface |
| **State Pattern** | Order State - PENDING, PAID, CONFIRMED, SHIPPED, OUT_FOR_DELIVERY, DELIVERED, CANCELLED, RETURNED | Each state has its own behaviour, no big `if-else` for order status | **OCP + SRP** |
| **Template Method** | `BasePayment.process()` - Validate -> Deduct -> Confirm -> Log | Common payment flow fixed, only `doPay()` changes per gateway | **OCP** - Reuse common code |
| **Microservices** | User Service, Product Service, Cart Service, Order Service, Payment Service, Inventory Service, Notification Service, Search Service | Scale Product Search 10x during Big Billion Days, Payment stays stable | **SRP** - One service, one responsibility |
