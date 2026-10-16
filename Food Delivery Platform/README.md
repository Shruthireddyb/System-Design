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


