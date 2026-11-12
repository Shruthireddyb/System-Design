# OTT (Over-The-Top) Platform -HLD
> Netflix / Prime Video / Hotstar scale video streaming platform

## 1. Overview
OTT Platform is a microservices based video streaming system that handles 10M+ concurrent users. It handles upload, transcode, and deliver video globally with personalized recommendations.

## 2. Requirements
**Functional**
- User auth,
- Subscription,
- Content upload,
- Transcoding (144p to 4K),
- Search catalogue,
- Streaming via CDN,
- Watch history,
- Recommendations,
- Notifications,
- Analytics
  
**Non-Functional**
- 99.99% Available
- <200ms latency via CDN,
- Scalable,
- Durable S3 storage

## Design Decisions

| Decision / Pattern | Used In OTT For | Why This Pattern / Choice? | SOLID / Design Principle |
| :--- | :--- | :--- | :--- |
| **Command** | User Action - `PlayCommand`, `PauseCommand`, `AddToWatchlistCommand`, `DownloadCommand`, `SubscribeCommand` all implement `UserCommand.execute()` | Each user action is an object - can be queued, logged, undone, retried, audited | **OCP + SRP** - New action = new class, no change in Player Service |
| **Strategy** | Video Quality Strategy - Auto, 144p, 360p, 720p, 1080p, 4K | Quality changes at runtime based on network speed, device, user plan | **OCP** - Add 8K without changing Player |
| **Strategy** | Subscription Plan Strategy - Mobile, Basic, Standard, Premium | Price, screens, quality, ads change by plan | **OCP + SRP** |
| **Strategy** | Recommendation Strategy - Trending, Collaborative Filtering, Content-Based, Watch History | Recommendation logic changes per user, per section | **OCP** |
| **Strategy** | Payment Strategy - UPI, Card, NetBanking, Wallet, Google Play, App Store | Same checkout, different payment method | **OCP** |
| **Factory** | Content Factory - `ContentFactory.create(MOVIE / SERIES / LIVE / SHORTS)` | Client sends content type, Factory returns correct Content class | **SRP + OCP** - Creation logic in one place |
| **Factory** | Transcoder Factory - HLS, DASH, MP4 | Same video creates different formats per device | **OCP** |
| **Singleton** | DB Connection, Redis Client, Kafka Producer, Config Manager, CDN Client | Only one instance should control shared resource, avoid duplicate connections | **SRP** - One instance controls one shared resource |
| **Observer** | Content Events - New Release, Episode Added, Subscription Expiry, Watchlist Update | One event notifies many - Push, Email, SMS, Home Page, Recommendation | **OCP + SRP** - Sender and receivers decoupled |
| **Builder** | Building `Video` - VideoId, Title, Thumbnail, Subtitles, Audio Tracks, Qualities, Duration | Video has 20+ fields, many optional, avoid `new Video(a,b,c,d...)` | **SRP** - Construction separate from representation |
| **Adapter** | Payment Gateway - Razorpay, Stripe, Google Play, Apple Pay | Our code uses one `PaymentGateway` interface, Adapter converts to vendor format | **OCP + DIP** - Depend on abstraction, not vendor |
| **Adapter** | DRM Provider - Widevine, FairPlay, PlayReady | Same interface, different DRM per device - Android / iOS / Web | **OCP + DIP** |
| **Decorator** | Video Stream - Base Video + Subtitles + Audio Dub + Ad + Watermark | Add features dynamically like `Ad(Subtitles(BaseVideo))` without changing base | **OCP + SRP** - Each feature in its own class |
| **Repository** | `UserRepository`, `ContentRepository`, `WatchHistoryRepository`, `SubscriptionRepository` | Separate DB logic from business logic, easy to switch PostgreSQL to MongoDB | **SRP + DIP** - Business logic doesn't depend on DB |
| **Facade** | `PlaybackFacade.play()` | One method hides - Check Subscription, Check DRM, Get CDN URL, Resume From Last Position, Log Analytics | **SRP** - Client sees one simple interface |
| **State Pattern** | Video Player State - IDLE, BUFFERING, PLAYING, PAUSED, SEEKING, ENDED, ERROR | Each state has its own behaviour, no big `if-else` for player | **OCP + SRP** |
| **State Pattern** | Subscription State - TRIAL, ACTIVE, EXPIRED, CANCELLED, GRACE_PERIOD | Each state decides - Can watch? Show paywall? | **OCP + SRP** |
| **Template Method** | `BaseTranscoder.process()` - Upload -> Transcode -> Generate Thumbnails -> Store in S3 -> Publish | Common video pipeline fixed, only `transcode()` changes per format | **OCP** - Reuse common code |
| **Microservices** | User Service, Content Service, Catalog Service, Subscription Service, Payment Service, Streaming Service, Recommendation Service, Notification Service, Search Service | Scale Streaming 10x on new movie release, Recommendation stays stable | **SRP** - One service, one responsibility |
