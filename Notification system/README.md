# Notification System 

> Scalable Multi-Channel Notification System handling 100M+ notifications/day via Email, SMS, Push, WhatsApp, In-App. Built with Kafka, Redis, Template Strategy, Retry Queue, Rate Limiting.
---

### 1. Requirements

#### A. Functional Requirements
1. **Multi-Channel:** Send notification via Email, SMS, Push, WhatsApp, In-App
2. **Multi-Type:**
   - Transactional (OTP, Booking Confirmed - high priority)
   - Promotional (Offer - low priority, can be delayed)
   - System (Feed liked, Comment - medium priority)
3. **Template Support:** Dynamic templates with variables `Hi {{name}}, your booking {{bookingId}} confirmed`
4. **User Preference:** User can opt-out: "Don't send me promotional SMS but allow OTP"
5. **Scheduling:** Send later - `Remind me tomorrow 10 AM`
6. **Tracking:** Delivery status - SENT, DELIVERED, FAILED, BOUNCED, OPENED
7. **Batch:** Send to 1M users at once (Promotional)

#### B. Non-Functional Requirements
1. **High Throughput:** 100M notifications/day ~ 1.5K/sec avg, peak 10K/sec
2. **Low Latency:** OTP should be delivered < 5 sec (p99)
3. **Durability:** No notification should be lost - Even if service crashes, OTP must go
4. **Availability:** 99.95% - Notification failure should not fail main flow (BookMyShow booking success even if SMS fails, retry later)
5. **Rate Limiting:** Don't spam user - Max 5 SMS/hour/user, Max 1 promotional/day
6. **Scalability:** Easy to add new channel (e.g., add Slack tomorrow without code change in core)

## Design Decisions

| Decision / Pattern | Used In Notification For | Why This Pattern / Choice? | SOLID / Design Principle |
| :--- | :--- | :--- | :--- |
| **Strategy** | Channel Strategy - Email, SMS, Push, WhatsApp, In-App | Same notification, different channel logic, user chooses channel at runtime | **OCP** - Add Telegram without changing Notification Service |
| **Strategy** | Template Strategy - Welcome, OTP, Order Placed, Payment Failed, Password Reset | Same send flow, different template + variables per type | **OCP + SRP** |
| **Strategy** | Priority Strategy - Critical (OTP), High (Payment), Medium (Order), Low (Promo) | Critical goes now, Low can be delayed / batched | **OCP** |
| **Factory** | Notification Factory - `NotificationFactory.create(EMAIL / SMS / PUSH)` | Client sends channel type, Factory returns correct Sender class | **SRP + OCP** - Creation logic in one place |
| **Factory** | Provider Factory - Twilio, SendGrid, Firebase, AWS SNS | Same interface, different provider per channel / region | **OCP** |
| **Observer** | Order Events - Order Placed, Shipped, Delivered | One event notifies many - Email, SMS, Push, In-App, all listen to same event | **OCP + SRP** - Order Service doesn't know who is listening |
| **Builder** | Building `Notification` - To, Subject, Body, TemplateId, Variables, Priority, RetryCount | Notification has 15+ fields, many optional, avoid `new Notification(a,b,c...)` | **SRP** - Construction separate from representation |
| **Adapter** | Provider Adapter - Twilio SMS, SendGrid Email, FCM Push, WhatsApp API | Our code uses one `NotificationSender` interface, Adapter converts to vendor format | **OCP + DIP** - Depend on abstraction, not vendor |
| **Decorator** | Notification - Base + Retry + RateLimit + Logging + Personalization | Add features dynamically like `Retry(RateLimit(Logging(BaseSender)))` without changing base | **OCP + SRP** |
| **Singleton** | Kafka Producer, Redis Client, Template Cache, Config Manager | Only one instance should control shared resource, avoid duplicate connections | **SRP** |
| **Repository** | `NotificationRepository`, `TemplateRepository`, `UserPreferenceRepository` | Separate DB logic from business logic, easy to switch PostgreSQL to MongoDB | **SRP + DIP** |
| **Facade** | `NotificationFacade.send()` | One method hides - Check Preference, Check DND, Pick Template, Pick Channel, Pick Provider, Send, Log | **SRP** - Client sees one simple interface |
| **State Pattern** | Notification State - PENDING, QUEUED, SENT, DELIVERED, FAILED, RETRYING, BOUNCED | Each state has its own behaviour, no big `if-else` for status | **OCP + SRP** |
| **Template Method** | `BaseSender.send()` - Validate -> Render Template -> Check Preference -> Send -> Log -> Update Status | Common flow fixed, only `doSend()` changes per channel | **OCP** - Reuse common code |
| **Microservices** | Notification API Service, Template Service, Preference Service, Dispatcher Service, Worker Service, Tracking Service | Scale Worker 10x during Big Billion Days, Template Service stays stable | **SRP** - One service, one responsibility |
