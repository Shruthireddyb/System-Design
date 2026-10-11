# OOT (Over-The-Top) Platform -HLD
> Netflix / Prime Video / Hotstar scale video streaming platform

## 1. Overview
OOT Platform is a microservices based video streaming system that handles 10M+ concurrent users. It handles upload, transcode, and deliver video globally with personalized recommendations.

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

## 3. High Level Architecture
- Clients (Web, Mobile, TV) -> CDN (AWS CloudFront) -> Media Service -> S3 Media |
->  API Gateway
-> User Service -> Relational DB with Replication
-> Catalogue Service (Search) -> Elastic Search + Redis Cache
-> Video Ingestion -> S3 Raw -> Kafka -> Transcoding (FFmpeg) -> S3 Transcoded (HLS) -> CDN
-> Streaming Service -> CDN -> Client
-> Order/Subscription -> Kafka -> Fulfillment -> Relational DB
-> Notification Service -> FBC / Push, WA, SMS, Email
-> Flink -> Data Lake (S3, Iceberg) <- Analytic Service
 <- AI Service (GenAI, Rec)

## 4. Core Microservices
- User Service: Spring Boot + JWT + MySQL Primary-Replica
- Catalogue Service: Spring Boot + Elastic Search + Redis (Cache-Aside)
- Video Ingestion & Transcoding: S3 + Kafka (video-uploaded topic) + FFmpeg -> HLS .m3u8
- treaming Service: CloudFront CDN + HLS/DASH Adaptive Bitrate
- Order/Subscription + Fulfillment: Kafka async activation
- otification: Kafka Consumer + Firebase
- Data Platform: Flink for watch heartbeat -> Data Lake (S3 + Iceberg) -> Analytic & AI Recommendation


## 5. Key Design Decisions
1. Why CDN + S3? S3 durable, CDN low-latency global
2. Why Kafka? Transcoding is slow (10-20 min), async prevents timeout
3. Why Elastic? MySQL LIKE is slow, Elastic fast full-text
4. Why HLS? Adaptive bitrate based on network
