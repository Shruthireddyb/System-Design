# Splitwise - LLD

> A scalable, consistent expense-splitting system to manage group expenses and settle debts with minimum transactions.

## 1. Problem Statement
Users go on trips, share flats, eat together. One person pays, others need to pay back. Tracking who owes whom in groups is complex and leads to confusion.

We need a system like Splitwise to manage groups, add expenses, track balances, and settle debts efficiently.

## 2. Requirements

### Functional Requirements
- Users can register / login
- Users can create groups (e.g., Trip to Goa, Flat 302)
- Users can add friends and add them to groups
- Users can add an expense in a group with amount, who paid, how to split (EQUAL, EXACT, PERCENTAGE)
- System should maintain balances: who owes whom
- Users can settle up: record a payment from A to B
- Simplify debts: minimize number of transactions to settle the group
- Dashboard: total you owe, total you are owed

### Non-Functional Requirements
- Strong Consistency for balances
- High Availability
- Low latency for balance calculations
- Scalable for millions of users and groups

### Out of Scope
- Actual payment gateway integration (we only record settlement)
- Chat / Push Notifications in v1

### 3. Pattern Selection Table

| # | Pattern | Where Used In Splitwise | Problem It Solves | Why This Pattern |
| :--- | :--- | :--- | :--- | :--- |
| P-01 | Strategy | Split Calculation: Equal / Exact / Percentage | 3 Different Split Logics, If-Else Hell | Add New Split Type Without Changing Existing Code |
| P-02 | Observer | Expense Added -> Notify Members, Update Audit, Update Balance | One Event, Many Actions | Expense Service Doesn't Know About Email, SMS, Audit |
| P-03 | Factory | Creation Of `Expense`, `Settlement` Objects | Complex Object Creation With Validation | Centralize Creation Logic |
| P-04 | Builder | `Expense` And `User` Creation With Many Optional Fields | Constructor With 10 Params | Readable, Immutable, Optional Fields |
| P-05 | Singleton | `JwtService`, `RedisConfig`, `KafkaTemplate` | Only One Instance Needed | Spring Beans Are Singleton By Default |
| P-06 | Repository | `UserRepository`, `ExpenseRepository`, `GroupRepository` | Decouple DB Logic From Business Logic | Change DB Without Changing Service |
| P-07 | DTO | `RegisterRequest`, `ExpenseRequest`, `BalanceResponse` | Don't Expose Entity To Client | Hide Password, Internal Fields |
| P-08 | Adapter | `KafkaProducer`, `EmailService` Adapts To External Service | External Service API Changes | Our Code Doesn't Break If Email Provider Changes |
| P-09 | Decorator | Logging + Validation + Caching On Service Layer | Add Cross-Cutting Concerns Without Changing Core Logic | `@Cacheable`, `@Transactional` Are Decorators |
| P-10 | Chain Of Responsibility | Validation Chain: DTO -> Business -> Auth -> Idempotency | Multiple Validations In Order | Fail Fast, Clear Order |
| P-11 | Command | `SettleUpCommand`, `AddExpenseCommand` | Encapsulate Request As Object For Undo / Retry | Useful For Kafka Retry, Audit |
| P-12 | Template Method | `BaseService` With `Validate -> Process -> Publish Event` | Same Flow For All Services | Avoid Duplication |

## Concurrency Handling

### 1. Problem
- Two users add / settle same expense at same time
- Risk of double counting, wrong balance, duplicate settlement

### 2. Balance Update Locking
- Lock on `group_id` or `user_pair` before updating balance
- Use `SELECT ... FOR UPDATE` on balance row
- Only one transaction updates balance at a time

### 3. Optimistic Locking
- Use `version` column on `expense` and `balance` tables
- Reject update if version changed
- Client retries with latest balance

### 4. Atomic Transaction
- Expense create + Balance update + Activity log in one transaction
- Rollback all changes on any failure
- No partial expense saved

### 5. Idempotency
- Each expense request has unique `idempotency_key`
- Retry with same key returns same expense
- Prevents duplicate expense on network retry

### 6. Distributed Lock
- Use Redis `SETNX` / Redlock on `group:{group_id}` or `pair:{user1}_{user2}`
- Prevents race across multiple servers

### 7. Settle Up Safety
- Settle Up uses transaction + lock on both users balances
- Check balance again before settling
- Prevents over-settlement and negative balance

### 8. Event Ordering
- Use queue for expense updates in same group
- Process expenses sequentially per group
- Final balance always consistent

### 9. Cache Handling
- Balance cached in Redis
- Cache updated only after DB commit
- Cache invalidated on expense edit / delete
  
