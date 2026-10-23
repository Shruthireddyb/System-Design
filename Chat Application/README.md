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

## 3. Database Design

### MySQL 
```sql
users: user_id, name, phone, last_seen
groups: group_id, name, created_by, member_count
group_members: group_id, user_id, joined_at, role
