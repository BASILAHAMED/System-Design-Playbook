# Twitter / X - Social Media System Design

## 1. Introduction

Design a microblogging platform like Twitter/X that allows users to:
- Post short messages (tweets) with media
- Follow other users
- Like, retweet, reply
- Real-time feed/timeline
- Search
- Notifications

## 2. Requirements

### Functional Requirements
- Post tweets (text, images, videos, polls)
- Follow/unfollow users
- View timeline (home timeline, user timeline)
- Like, retweet, reply to tweets
- Direct messaging
- Search tweets and users
- Notifications (likes, retweets, mentions, DMs)
- User profiles
- Trending topics
- Hashtags and mentions
- Media upload and storage
- Account settings

### Non-Functional Requirements
- High write throughput (many tweets per second)
- Low latency timeline generation (< 200ms)
- High availability (99.99%)
- Scalability to hundreds of millions of users
- Real-time feed updates
- Content moderation
- Global CDN for media

## 3. Capacity Estimation

**Assumptions:**
- 500 million total users
- 100 million daily active users (DAU)
- 50 million users posting tweets daily
- 100 million tweets per day (~1,157 tweets/sec)
- 1 billion timeline requests per day (~11,574/sec)
- Average user follows 200 people
- Average tweet: 140 characters (~200 bytes)
- Media: 10% of tweets have images, 1% have videos

**Storage:**
- Tweets: 100M/day × 200 bytes = 20GB/day → 7.3TB/year
- Media: 10M images/day × 1MB = 10TB/day (heavily compressed)
- Timeline data: 100M users × 200 follows × 100 latest tweets ≈ 2TB
- 5-year storage: ~200TB (tweets + metadata)

**Bandwidth:**
- Timeline requests: 1B/day × 50KB (avg page) = 50TB/day
- Media delivery: major bandwidth consumer (CDN)

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Mobile  │  │  Web    │  │  Tablet │  │   API   │      │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │           │           │           │              │
│       └───────────┴───────────┴───────────┘              │
└───────────────────────────┬───────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                    API Gateway Layer                       │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Load Balancer + Auth (OAuth 2.0)                  │  │
│  └────────────────────────────┬────────────────────────┘  │
└───────────────────────────────┼───────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────▼─────────┐ ┌────────▼─────────┐ ┌────────▼─────────┐
│   Tweet Service   │ │  User Service    │ │  Timeline       │
│                   │ │                  │ │  Service        │
│ • Post tweet      │ │ • Profile        │ │ • Fan-out       │
│ • Delete tweet    │ │ • Follow/unfollow│ │ • Pull          │
│ • Media upload    │ │ • Search users   │ │ • Cache         │
└─────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────┐
│               Processing & Storage Layer                  │
│  ┌─────────────┐ ┌────────────┐ ┌────────────┐         │
│  │   Message   │ │   Search   │ │   Media    │         │
│  │    Queue    │ │  Service   │ │  Storage   │         │
│  │  (Kafka)    │ │(Elastic/   │ │  (S3/CDN)  │         │
│  │             │ │  Meilisearch)│ │            │         │
│  └─────────────┘ └────────────┘ └────────────┘         │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │           Main Database (PostgreSQL)               │ │
│  │  • Users, tweets, follows, likes, retweets        │ │
│  └─────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
```

## 5. Component Deep Dive

### 5.1 Tweet Service
**Operations:**
- POST /tweets - Create new tweet
- DELETE /tweets/:id - Delete tweet
- GET /tweets/:id - Get tweet details

**Data Model:**
```sql
CREATE TABLE tweets (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    content TEXT NOT NULL,
    media_urls TEXT[], -- array of URLs
    parent_tweet_id BIGINT REFERENCES tweets(id), -- for replies
    quoted_tweet_id BIGINT REFERENCES tweets(id), -- for quote tweets
    like_count INT DEFAULT 0,
    retweet_count INT DEFAULT 0,
    reply_count INT DEFAULT 0,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Likes
CREATE TABLE likes (
    user_id UUID REFERENCES users(id),
    tweet_id BIGINT REFERENCES tweets(id),
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, tweet_id)
);

-- Retweets
CREATE TABLE retweets (
    user_id UUID REFERENCES users(id),
    tweet_id BIGINT REFERENCES tweets(id),
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, tweet_id)
);

-- Follows
CREATE TABLE follows (
    follower_id UUID REFERENCES users(id),
    followee_id UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);
```

### 5.2 Timeline Service

**Two approaches:**

#### A) Fan-out on Write (Push Model)
When user posts a tweet:
1. Write tweet to DB
2. Get all followers of the user (from cache or DB)
3. For each follower, insert tweet ID into their timeline cache (Redis sorted set)
4. Timeline = cached tweets + recent DB lookup

**Pros:**
- Fast timeline reads (O(1) from cache)
- Good for users with many followers (celebrities)

**Cons:**
- Write amplification: celebrity with 10M followers = 10M inserts per tweet
- Storage waste: duplicate tweet data in many timelines

#### B) Pull Model (Fan-in on Read)
When user requests timeline:
1. Get list of users they follow (from cache)
2. For each followed user, get their latest N tweets (from cache)
3. Merge and sort by timestamp
4. Return top N

**Pros:**
- Write is cheap (only one insert)
- No duplication

**Cons:**
- Read is expensive (fetch from many users)
- Poor for users following many people

#### C) Hybrid Approach (Recommended)
- **Normal users (< 10K followers):** Fan-out on write
- **Celebrities (> 10K followers):** Pull model for their tweets
- Timeline = (fan-out tweets from normal users) UNION (celebrity tweets pulled on read)
- Use Redis sorted sets: `ZADD timeline:{user_id} {timestamp} {tweet_id}`

**Implementation:**
```python
# On tweet creation
tweet_id = save_tweet(user_id, content)
followers = get_followers(user_id)  # from cache

if len(followers) < CELEBRITY_THRESHOLD:
    # Fan-out to all followers
    for follower in followers:
        redis.zadd(f"timeline:{follower}", {tweet_id: timestamp})
else:
    # Mark as celebrity tweet, will be pulled on timeline read
    redis.sadd("celebrity:users", user_id)
    redis.zadd(f"tweets:user:{user_id}", {tweet_id: timestamp})
```

**Read timeline:**
```python
def get_timeline(user_id, limit=50):
    # Get fan-out tweets (from followers with < threshold)
    timeline = redis.zrevrange(f"timeline:{user_id}", 0, limit-1)
    
    # Pull celebrity tweets
    celebrity_users = redis.smembers("celebrity:users")
    for celeb in celebrity_users[:10]:  # limit to top 10 celebs followed
        celeb_tweets = redis.zrevrange(f"tweets:user:{celeb}", 0, 20)
        timeline.extend(celeb_tweets)
    
    # Sort by timestamp, deduplicate, return top N
    return merge_and_sort(timeline)[:limit]
```

### 5.3 Media Storage
- **Upload:** Direct to S3 via pre-signed URLs (avoid proxy through app server)
- **Processing:** Generate thumbnails, compress images, transcode videos
- **CDN:** CloudFront/Cloudflare for global delivery
- **Storage classes:** S3 Standard for hot, S3 IA for older media

**Flow:**
1. Client requests upload URL from API
2. API generates pre-signed PUT URL (expires in 1 hour)
3. Client uploads directly to S3
4. S3 triggers Lambda/worker to process media
5. Processing stores processed files back to S3
6. Update tweet with media URLs

### 5.4 Search Service
**Elasticsearch/Meilisearch cluster:**
- Index: tweets (id, user_id, content, hashtags, mentions, media_type, created_at)
- Index: users (id, username, display_name, bio)
- Real-time indexing via Kafka (tweet created → index update)

**Features:**
- Full-text search
- Hashtag search
- Mention search
- User search (autocomplete)
- Recent tweets filter
- Filter by language, media type

### 5.5 Notification Service
**Types:**
- Push: likes, retweets, mentions, new followers
- Email: daily digest, security alerts
- SMS: verification, password reset

**Implementation:**
- Event-driven: tweet liked → push notification to tweet author
- Queue: Kafka topic for notifications
- Worker: consume events, determine recipients, send via FCM/APNs
- Store notifications in DB for in-app feed

**Notification table:**
```sql
CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    type VARCHAR(50), -- 'like', 'retweet', 'follow', 'mention'
    actor_id UUID REFERENCES users(id), -- who triggered it
    tweet_id BIGINT REFERENCES tweets(id), -- optional
    is_read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

## 6. Scalability Considerations

### 6.1 Database Scaling
- **Sharding:** Shard by user_id (range or hash)
- **Read replicas:** For timeline reads, profile views, search queries
- **Caching:** Redis for timelines, user profiles, tweet counts
- **Hotspots:** Celebrity tweets (many reads) → cache aggressively

### 6.2 Timeline Scale
- With hybrid approach, most reads served from Redis
- Redis cluster with sharding by user_id
- Memory estimates: 100M users × 800 bytes/timeline entry × 100 entries = 80GB
- Use Redis compression, eviction policy (LRU)

### 6.3 Write Throughput
- Tweet writes: 1,157/sec average, peak maybe 10x = 12K/sec
- Database can handle with proper indexing and partitioning
- Message queue decouples timeline fan-out processing

### 6.4 Media Scale
- 10M images/day = 10TB/day ingress
- CDN handles egress (major cost)
- Use image optimization (WebP, progressive JPEG)
- Video: transcode to multiple resolutions (HLS/DASH)

## 7. Advanced Features

### 7.1 Trending Topics
- **Algorithm:** Count tweet frequency per hashtag in last 24h
- **Location-specific:** Separate trends per country/city
- **Storage:** Redis sorted sets: `trending:global`, `trending:us`, etc.
- **Update:** Every 5 minutes via batch job
- **Display:** Top 10-20 hashtags

### 7.2 Content Moderation
- **Automated:** ML models to detect hate speech, spam, NSFW
- **Manual:** Human review queue for borderline cases
- **User reports:** Flagging system
- **Actions:** Remove tweet, shadow-ban, suspend account

### 7.3 Direct Messaging (DMs)
- 1:1 and group chats
- End-to-end encryption (optional)
- Real-time delivery via WebSocket
- Media attachments
- Read receipts
- Typing indicators

**Storage:** Separate table or separate database for privacy

### 7.4 Analytics
- User engagement (likes, retweets, replies)
- Tweet reach (impressions)
- Follower growth
- Active users (DAU, MAU)
- Real-time dashboards (Kafka → Flink → DB)

## 8. Fault Tolerance & Reliability

- **Multi-region:** Active-active for timeline service (eventual consistency OK)
- **Database:** Primary-replica with automatic failover
- **Redis:** Redis Cluster with replication
- **Message queue:** Kafka with replication factor 3
- **Graceful degradation:** If timeline service slow, fall back to recent tweets from DB

## 9. Monitoring & Alerting

- **Metrics:** Tweet rate, timeline latency, active users, notification delivery rate
- **Alerts:** High error rate on tweet creation, timeline cache miss rate > threshold
- **Dashboards:** Real-time tweet volume heatmap, top trending topics

## 10. Security Considerations

- **Rate limiting:** Per user/IP to prevent spam
- **Content validation:** Sanitize HTML, prevent XSS
- **Authentication:** OAuth 2.0, JWT tokens
- **Authorization:** Users can only delete their own tweets
- **Privacy:** Protected accounts, DM privacy
- **Abuse detection:** Suspicious activity (mass following, spam)

## 11. Trade-offs

| Decision | Fan-out on Write | Pull Model | Trade-off |
|----------|------------------|------------|-----------|
| Read latency | Low (cached) | High (compute) | Speed vs Storage |
| Write cost | High (amplification) | Low | Consistency vs Complexity |
| Celebrity tweets | Poor (no fan-out) | Good | Special case handling |
| Storage | High (duplication) | Low | Cost vs Performance |

**Hybrid approach balances both.**

## 12. Future Enhancements

- Spaces (audio chat rooms)
- Communities (topic-based groups)
- Super follows (paid content)
- Articles (long-form posts)
- NFT integration
- AI-powered content recommendations
- Edit tweet feature
- Conversation threading (nested replies)
- Better analytics for creators
- Shopping integration

## 13. References

- [Twitter Engineering Blog](https://blog.twitter.com/engineering)
- [The Architecture of Twitter’s Home Timeline](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/)
- [Redis Sorted Sets for Timelines](https://redis.io/topics/data-types)
- [Designing a Scalable Timeline](https://www.youtube.com/watch?v=KIL3bFLfG8U)

---

**Difficulty:** Hard 🔴
**Time to design:** 45-60 minutes
**Key focus areas:** Fan-out vs pull, timeline generation, real-time, scalability, hybrid approach