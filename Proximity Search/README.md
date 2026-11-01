# Proximity Search 

> Scalable Nearby Search System like Google Maps, Uber, Swiggy, Tinder - Find nearby places, drivers, restaurants in < 100ms

## 1. Overview

Proximity Search (Geo Search) is core of every location-based app. Problem: Given user lat/lng, find all POIs (restaurants, cabs, ATMs) within X km radius, sorted by distance.

**Challenge:** Cannot do `SELECT * WHERE distance < 5km` - it will scan full DB and fail at scale (10M+ POIs).

This design handles **100M POIs, 50k QPS, <100ms latency**.

## 2. Requirements

### Functional
- Add / Update / Delete business location (restaurant, driver, user)
- Search nearby: `GET /nearby?lat=12.97&lng=77.59&radius=3km&category=food`
- Return results sorted by distance, with pagination
- Real-time location update for moving entities (delivery partners, cabs)

### Non-Functional
- Latency: p95 < 100ms
- Scalability: 100M locations, 50k searches/sec
- Availability: 99.95%
- Consistency: Eventual for search (Redis), Strong for DB
- Accuracy: Search should be precise, not approximate

## 3. High-Level Architecture
- [ Mobile App ]
      
- [ API Gateway ]
      
- [ Proximity Service - Spring Boot ]
      
- [ Redis Cluster ] (GEO index for real-time search)
-  [ MySQL/PostGIS + Elasticsearch ]
  (Persistent store + complex filter by rating, cuisine)
      
- [ Kafka ] <- Location updates from Delivery Partners (via MQTT)
      
- [ Location Worker ] -> Updates Redis GEO + DB

## Design Decisions 

###  Pattern Decisions

| Pattern | Why Used in Proximity Search? | Example | Principle |
| :--- | :--- | :--- | :--- |
| **Strategy** | Different search strategies by distance | `ProximityStrategy` -> `Nearby1kmStrategy`, `Nearby5kmStrategy`, `CityLevelStrategy` | OCP - Add 10km strategy without changing service |
| **Factory** | Create index based on use-case | `GeoIndexFactory.getIndex("LIVE_TRACKING")` returns Redis, `HISTORY` returns Cassandra | SRP - Creation isolated |
| **Singleton** | Redis GEO Client + Geohash Encoder | `GeoRedisClient` as Singleton | One connection pool, thread-safe |
| **Observer** | Location update -> Notify search service | Device publishes location -> Kafka -> `ProximityService`, `AlertService`, `AnalyticsService` | Decoupled |


###  Scalability Decisions

| Decision | Choice | Reason |
| :--- | :--- | :--- |
| **Sharding** | By City / Geohash prefix | Query for Bangalore never hits Delhi shard. |
| **Write Path** | Device -> MQTT -> Kafka -> Consumer -> Redis GEOADD | Async, handles burst. If Redis down, Kafka retains. |
| **Read Path** | API -> Redis GEORADIUS -> If >50 results, filter Haversine | 95% requests served in <10ms from Redis. |
| **Precision** | Geohash length 6 (1.2km x 0.6km) for index, 7 (150m) for alert | 6 is good for nearby, 7 for geofence alert. |
| **Cold Start** | Cache warming from Cassandra | On Redis crash, reload last known location. |


> "Used Redis GEO with Geohash (6-char) for O(log N) proximity search, sharded by city to avoid full scan, Kafka for async 5-sec updates, and Strategy pattern to support 1km/5km/city search without changing core service. Two-step filtering - Geohash neighbors first, then Haversine for accuracy."

