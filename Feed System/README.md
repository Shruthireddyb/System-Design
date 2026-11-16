# Feed System - HLD

> Scalable News Feed / Instagram Home Feed - Hybrid Fanout, Ranking, Real-time Timeline. Interview-ready with requirements, API design, DB schema, and tradeoffs.

---

### 1. Requirements - 

#### A. Functional Requirements 

**Core:**
1. **Post:** User can create post (text + image/video up to 10MB)
2. **Follow:** User can follow/unfollow other users
3. **Feed:** User can see Home Feed - posts from users they follow, sorted by time / relevance
4. **Timeline:** User can see any user's profile timeline (own posts)
5. **Engagement:** User can like, comment, share a post
6. **Pagination:** Infinite scroll - load 20 posts per page

**Extended :**
7. Stories / Reels (short video feed)
8. Hashtag & Mention in posts
9. Block / Mute user - their posts shouldn't appear

#### B. Non-Functional Requirements (Scale)

1. **Read Heavy:** 100:1 read/write ratio. 500K feed reads/sec vs 2K post writes/sec
2. **Low Latency:** Feed generation < 200ms (p95)
3. **Highly Available:** Feed should work even if like/comment fails. AP over CP
4. **Near Real-time:** New post should appear in follower's feed within 5 seconds
5. **Scalable:** Support 300M users, 10M DAU, 1M new posts/day
6. **Durable:** No post loss - MySQL as source of truth, Kafka with ack=all

## Design Decisions

###  Pattern Decisions (OCP, SRP)

| Pattern | Why Used in Feed System? | Example | Principle |
| :--- | :--- | :--- | :--- |
| **Strategy** | Different Feed Ranking Strategies | `FeedRankingStrategy` -> `ChronologicalStrategy`, `PopularityStrategy`, `MLBasedStrategy` | OCP - Add new ranking algo without changing Feed Service |
| **Factory** | Create Feed based on user type | `FeedFactory.create("CELEBRITY_FEED")` vs `NORMAL_FEED` -> decides Push vs Pull | SRP - Creation logic isolated, handles celeb special case |
| **Singleton** | Feed Cache Manager + Snowflake Generator | `FeedCacheManager`, `IdGenerator` as Singleton bean | Single instance, thread-safe, no duplicate IDs |
| **Observer** | New Post -> Notify Feed Services | `PostService` (Publisher) -> `FanoutService`, `SearchService`, `NotificationService` (Subscribers) | Decoupled, extensible |

###  Strategy Decision

| User Type | Followers | Fanout Model | How Feed Generated | Latency |
| :--- | :--- | :--- | :--- | :--- |
| **Normal User** | < 10K | Push on Write | Write post ID to all followers' Redis feed list at post time | Read O(1) - Fast |
| **Celebrity** | > 10K / 1M | Pull on Read | Don't push. At read time, merge celeb posts + normal push feed | Write O(1) - Saves write storm |
| **Hybrid Logic** | Factory Decides | `if followers > threshold -> Pull else Push` | Best of both worlds | Needs background merger |

###  Scalability Decisions

| Decision | Choice | Reason |
| :--- | :--- | :--- |
| **Feed Pagination** | Cursor-based (postId) not OFFSET | OFFSET is slow on large feeds. Cursor is O(1). |
| **Hot Feed Cache** | Redis List: `feed:{userId}` -> [postIds] LTRIM 1000 | Keep only 1000 recent. Full feed in Cassandra. |
| **Write Path** | Post -> Kafka -> Fanout Workers -> Redis | Async fanout via Kafka, no API blocking. |
| **Read Path** | App -> Redis Cache -> If miss -> Cassandra -> Merge Pull | 95% hits from Redis, 5ms response. |
| **Media** | CDN (CloudFront) for images | Reduce origin load, 10x faster. |


> "Used Hybrid Fanout (Push for normal, Pull for celeb) to handle 1M followers without write storm, Cassandra + Redis for O(1) feed read, and Strategy pattern for OCP ranking so we can add ML ranking without changing core service."

## Concurrency Handling 

This feed system handles concurrent reads, writes, likes, comments, and follows
without data loss, duplicate posts, or inconsistent counts.

### 1. Problems We Handle
- Multiple users liking / commenting on same post at same time
- Same user double-clicking Like / Follow (duplicate requests)
- Feed fan-out: One post by a user with 10K followers
- Read-while-write: User sees feed while new post is being published
- Counter drift: Like count != actual rows in DB

### 2. Idempotency - No Duplicate Actions
Every write request must include an `Idempotency-Key` (UUID).

- Like, Comment, Follow, Post creation are idempotent
- If same key is retried, server returns cached response, does not re-execute
- DB has unique constraints to enforce it at last level:
