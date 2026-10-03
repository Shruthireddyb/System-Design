# Notification System 

> Scalable Multi-Channel Notification System handling 100M+ notifications/day via Email, SMS, Push, WhatsApp, In-App. Built with Kafka, Redis, Template Strategy, Retry Queue, Rate Limiting.

### Architecture: `Client -> API -> Kafka -> Workers -> Providers (SendGrid, Twilio, FCM) -> User`

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

#### C. User Stories
```
- As BookMyShow, after booking, send SMS + Email + Push: "Booking Confirmed"
- As Feed System, when someone likes my post, send In-App + Push notification
- As User, I want to mute promotional notifications at night (10PM-8AM DND)
- As Admin, I want to see how many OTP SMS failed yesterday
- As System, if SendGrid fails, retry 3 times with exponential backoff
```

### 2. Numbers 

```
100M notifs/day = 1157 req/sec avg
Storage: 100M * 500 bytes (log) = 50GB/day -> 1.5TB/month -> Need archiving to S3 after 30 days
Kafka: 100M messages/day ~ 1.2K/sec - Single Kafka cluster handles easily
Peak: Diwali offer - 10M users * 1 promo = 10M in 1 hour = 2777/sec
```

