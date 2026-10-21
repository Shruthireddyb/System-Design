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
