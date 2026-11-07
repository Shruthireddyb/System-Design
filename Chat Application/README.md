# Chat Application

> Scalable 1-1 & Group Chat System like WhatsApp, Telegram, Slack - Handles 10M concurrent connections, 1B messages/day

## 1. Overview

A real-time messaging platform supporting 1-1 chat, group chat, presence (online/offline), typing indicators, read receipts, and media sharing. Core challenge is real-time delivery, message ordering, and offline handling at scale.

**Key Features:**
- 1-1 and Group Chat (256 members)
- Online Presence, Last Seen, Typing Indicator
- Message Status: Sent -> Delivered -> Read (Double Tick)
- Offline Message Storage & Push Notifications
- Media Sharing (S3)
- End-to-End Encryption (Optional)

## 2. Requirements

### Functional
- Users can send/receive text, image, video, files
- Real-time delivery < 100ms
- Group chat creation, add/remove members
- Message history with pagination
- Users see online/offline status

### Non-Functional
- Latency: < 100ms for message delivery
- Scalability: 10M concurrent WebSocket connections
- Availability: 99.99% - messages should never be lost
- Ordering: Messages must be in order per conversation
- Storage: Chat history for years

## 3. Design Pattern Decisions

| Pattern | Used In Chat App For | Why This Pattern? | SOLID Principle | Problem Without This Pattern |
| :--- | :--- | :--- | :--- | :--- |
| **Strategy** | Message Sending - Text, Image, Video, File, Location | Each message type has different send / compress / validate logic, but same interface. Add new type without changing old code | **OCP** - Open for Extension, Closed for Modification | One big `if-else` in `sendMessage()` - every new type breaks old code |
| **Factory** | Creating Message Objects - `MessageFactory.create(type)` | Client doesn't know which class to create, Factory decides based on type | **SRP + OCP** - Creation logic in one place | Client code does `new TextMessage()`, `new ImageMessage()` everywhere - tight coupling |
| **Singleton** | DB Connection, Redis Client, WebSocket Manager, Logger | Only one connection pool / one socket manager should exist for whole app | **SRP** - One instance controls one shared resource | Multiple DB connections = connection leak, memory waste, inconsistent state |
| **Observer** | New Message, Typing Indicator, Online/Offline, Read Receipts | One event notifies many listeners - UI, Push Service, DB, Analytics - without sender knowing them | **OCP + SRP** - Sender and receivers decoupled | Sender has to manually call UI, Push, DB every time - tight coupling |
| **Builder** | Building `Message` / `User` Object - Optional fields like replyTo, mediaUrl, reactions | Message has 10+ optional fields, avoid telescoping constructor | **SRP** - Construction separate from representation | `new Message(a,b,c,d,e,f,g,null,null)` - unreadable, error-prone |
| **Adapter** | FCM, APNS, Twilio, S3, Third-Party APIs | Our code talks to one `NotificationService` interface, Adapter converts to FCM / APNS format | **OCP + DIP** - Depend on abstraction, not concrete API | Changing from FCM to APNS means changing code everywhere |
| **Decorator** | Message Features - Encryption, Compression, Logging, Spam Check | Add features dynamically like `Encrypted(Compressed(Logged(Message)))` without changing base message | **OCP + SRP** - Each feature in its own class | One `Message` class does encryption + compression + logging - huge god class |
| **Repository** | `UserRepository`, `MessageRepository`, `ChatRepository` | Separate DB logic from business logic, easy to switch PostgreSQL to Cassandra | **SRP + DIP** - Business logic doesn't depend on DB | SQL queries inside Controller / Service - untestable, hard to change DB |
| **Facade** | `ChatFacade.sendMessage()` | One simple method hides - Validate, Save, Publish, Notify, Update Cache | **SRP** - Client sees one interface | Client has to call 5 services in correct order every time |
| **Command** | Send, Edit, Delete, Forward, React to Message | Each action is an object - can be queued, retried, undone, logged | **OCP + SRP** - New action = new class | One service with 20 methods, no retry / queue support |
