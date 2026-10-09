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

## 3. Design Patterns Used:
- Strategy:
- PaymentObserver: Order -> Notification
- Factory: Order creation
- Singleton: Redis connection

## 4. Data Model
- User: id, email, password_hash, address[]
- Product: id, name, price, seller_id, images[], category
- Inventory: product_id [PK], quantity, reserved, version [optimistic lock]
- Cart: user_id -> Redis Hash { product_id: qty }
- Order: id, user_id, total, status, PNR-like order_number [UNIQUE]
- OrderItem: order_id, product_id, qty, price_at_purchase

## 5. Scale & Handling Failure
1. No Overselling:
   - Inventory Service: Redis SETNX lock:product:123 with 10s TTL + MySQL FOR UPDATE
   - On failure: Release lock via Lua script, Kafka retry 3 times
2. Idempotency:
   - Payment: Client sends Idempotency-Key: uuid -> Store in Redis payment:uuid -> If exists, return same result
3. Search:Product DB -> Debezium CDC -> Kafka -> Elasticsearch indexer
   - Cache top 1000 products in Redis
4. Cart:
   - Redis cluster, persistence AOF, TTL 7 days for abandoned cart -> Kafka job to send reminder
