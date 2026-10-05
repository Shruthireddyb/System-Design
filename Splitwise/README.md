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
