# Feed System - System Design - HLD

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

#### C. Out of Scope 
- Direct Messaging - Separate chat system
- Full text search on posts - Separate search service (Elasticsearch)
- Video transcoding - Separate media service
- Push notifications - Separate notification service (Kafka consumer)

#### D. Assumptions & User Stories
```
Assumption:
- Avg user follows 200 users
- Celebrity user has 10M followers
- Avg user opens feed 10 times/day
- Post size ~ 1KB text + 500KB image URL

User Stories:
- As a user, I post a photo -> my followers see it in feed in 5 sec
- As a user with 10M followers, I post -> system shouldn't crash
- As a user, I scroll feed -> next 20 posts load instantly from cache
```


### 2. LLD - Class Design

```java
User { Long id; String name; boolean isCelebrity; int followersCount }
Post { Long id; Long userId; String content; String mediaUrl; PostType type; LocalDateTime createdAt }
Follow { Long followerId; Long followeeId; }
FeedItem { Long postId; Long authorId; long timestamp; } // For timeline
Like { Long userId; Long postId; }

// Services
FeedService.getFeed(userId, cursor, size)
PostService.createPost(post) -> publishes Kafka event
FanoutService.fanoutPost(post) -> core logic
RankingStrategy.rank(feedItems) -> chronological / EdgeRank
```

**Patterns Used:**
- Strategy: RankingStrategy (Time vs Relevance)
- Factory: PostFactory (Text/Image/Video)
- Observer: PostCreated -> Fanout

### 3. HLD - Architecture

```
Client -> API Gateway -> LB
           |-> User Service (MySQL)
           |-> Post Service (MySQL + S3 + Kafka Producer)
           |-> Graph Service (Follow/Unfollow - MySQL/Cassandra)
           |-> Feed Service (Redis ZSET + Cassandra) - READ PATH
           |-> Fanout Service (Kafka Consumer) - WRITE PATH
           |-> Engagement Service (Like/Comment)

```
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
