# YouTube / Netflix - Video Streaming System Design

## 1. Introduction

Design a video streaming platform like YouTube or Netflix that supports:
- Uploading and storing videos
- Transcoding to multiple resolutions
- Streaming to millions of concurrent users
- Content recommendation
- User engagement tracking

## 2. Requirements

### Functional Requirements
- Upload videos in various formats
- Transcode videos to multiple resolutions (144p, 240p, 360p, 720p, 1080p, 4K)
- Stream videos with adaptive bitrate (DASH/HLS)
- Support live streaming
- Like/dislike, comments, subscriptions
- Content recommendations
- User profiles and watch history
- Search functionality

### Non-Functional Requirements
- High availability (99.99%)
- Low latency streaming (< 2s start time)
- Scalability to millions of concurrent viewers
- Global content delivery
- Cost-effective storage
- DRM and content protection

## 3. Capacity Estimation

**Assumptions:**
- 5 million daily active users
- 1 billion videos in catalog
- 500,000 videos uploaded daily (avg 100MB each = 50TB/day)
- Peak: 10 million concurrent streams
- Average video: 1 hour, 1080p (~1GB)

**Storage:**
- Daily uploads: 50TB
- Yearly: ~18PB
- With 3x replication: ~54PB
- Use cold/warm/hot storage tiers

**Bandwidth:**
- Peak concurrent: 10M streams × 5 Mbps = 50 Tbps
- CDN handles most traffic

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Mobile  │  │  Web    │  │   TV    │  │   IoT   │      │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │           │           │           │              │
│       └───────────┴───────────┴───────────┘              │
└───────────────────────────┬───────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                    API Gateway Layer                       │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Load Balancer (Regional) + Rate Limiting          │  │
│  └────────────────────────────┬────────────────────────┘  │
└───────────────────────────────┼───────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────▼─────────┐ ┌────────▼─────────┐ ┌────────▼─────────┐
│  Upload Service   │ │  Stream Service  │ │   Search Service │
│                   │ │                   │ │                  │
│ • Auth            │ │ • Auth            │ │ • Auth           │
│ • Validation      │ │ • Rate limiting   │ │ • Query parsing  │
│ • Chunk upload    │ │ • License check   │ │ • Index search   │
└─────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────┐
│                    Processing Layer                       │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Message Queue (Kafka/RabbitMQ)                     │  │
│  └────────────────────────────┬────────────────────────┘  │
│                               │                          │
│  ┌─────────────┐ ┌────────────┐ ┌────────────┐         │
│  │   Transcoder │ │  Thumbnail  │ │  Metadata  │         │
│  │   Service    │ │  Generator  │ │  Extractor │         │
│  └─────────────┘ └────────────┘ └────────────┘         │
└───────────────────────────────┬───────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────┐
│                    Storage Layer                          │
│  ┌─────────────┐ ┌────────────┐ ┌────────────┐         │
│  │   Object    │ │   CDN       │ │  Database  │         │
│  │  Storage    │ │  (Edge)     │ │ (Metadata) │         │
│  │ (S3/MinIO)  │ │             │ │ PostgreSQL │         │
│  └─────────────┘ └────────────┘ └────────────┘         │
└───────────────────────────────────────────────────────────┘
```

## 5. Component Deep Dive

### 5.1 Upload Service
- Chunked uploads (resume support)
- Direct upload to object storage via pre-signed URLs
- Video validation (format, size, duration)
- Integration with transcoder via message queue

### 5.2 Transcoding Pipeline
**Why needed:** Videos uploaded in various formats/resolutions need to be converted to standard formats and multiple bitrates for adaptive streaming.

**Process:**
1. Upload complete → message to transcoder queue
2. Pull video from storage
3. Extract metadata (duration, codec, resolution)
4. Transcode to multiple resolutions/bitrates
5. Generate thumbnails (preview, poster)
6. Segment for HLS/DASH (chunks + manifest)
7. Upload processed files back to storage
8. Update database with URLs

**Tools:** FFmpeg, AWS Elemental, GCP Transcoder API

### 5.3 Streaming Service
- **Adaptive Bitrate Streaming:** HLS (HTTP Live Streaming) or DASH
- **Chunked delivery:** Videos split into 2-10 second segments
- **Manifest files:** .m3u8 (HLS) or .mpd (DASH) listing available renditions
- **Client-side adaptation:** Player switches quality based on network

**Flow:**
1. Client requests video
2. Auth + rate limiting
3. Return manifest file
4. Client requests chunks sequentially
5. CDN serves chunks (edge caching)

### 5.4 CDN (Content Delivery Network)
- Edge caching of video segments
- POPs (Points of Presence) globally
- Cache warm-up for popular content
- Origin fetch for cache misses
- Support for range requests (seeking)

**Providers:** CloudFront, Cloudflare, Akamai

### 5.5 Database Design

**Metadata Database (PostgreSQL/MySQL):**
```sql
-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    username VARCHAR(100),
    created_at TIMESTAMP
);

-- Videos
CREATE TABLE videos (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    title VARCHAR(500),
    description TEXT,
    original_url TEXT,
    status ENUM('uploading', 'processing', 'ready', 'failed'),
    duration INT, -- seconds
    width INT,
    height INT,
    created_at TIMESTAMP
);

-- Video Renditions (multiple resolutions)
CREATE TABLE video_renditions (
    id UUID PRIMARY KEY,
    video_id UUID REFERENCES videos(id),
    resolution VARCHAR(20), -- '1080p', '720p', etc.
    bitrate INT, -- kbps
    codec VARCHAR(50),
    url TEXT,
    filesize BIGINT
);

-- Thumbnails
CREATE TABLE thumbnails (
    id UUID PRIMARY KEY,
    video_id UUID REFERENCES videos(id),
    type VARCHAR(20), -- 'default', 'medium', 'high'
    url TEXT
);

-- Views/Engagement
CREATE TABLE views (
    id BIGSERIAL PRIMARY KEY,
    video_id UUID REFERENCES videos(id),
    user_id UUID REFERENCES users(id),
    watch_time INT, -- seconds watched
    device_type VARCHAR(50),
    ip_address INET,
    created_at TIMESTAMP
);

-- Comments
CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    video_id UUID REFERENCES videos(id),
    user_id UUID REFERENCES users(id),
    parent_id BIGINT REFERENCES comments(id), -- for replies
    content TEXT,
    created_at TIMESTAMP
);

-- Subscriptions
CREATE TABLE subscriptions (
    user_id UUID REFERENCES users(id),
    channel_id UUID REFERENCES users(id),
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, channel_id)
);
```

**Search Index (Elasticsearch/Meilisearch):**
- Video title, description, tags
- Channel name
- Transcripts (if available)
- Faceted search (category, duration, upload date)

### 5.6 Caching Strategy
- **CDN:** Video chunks (long TTL, days/weeks)
- **Application cache (Redis):**
  - Popular video metadata (1 hour)
  - User session data
  - Trending videos (5 minutes)
  - View counts (write-back to DB periodically)

## 6. Advanced Features

### 6.1 Live Streaming
- **Ingest:** RTMP/SRT protocols
- **Transcoder:** Live transcoding to multiple renditions
- **Packager:** HLS/DASH packaging
- **CDN:** Low-latency delivery (LL-HLS, LL-DASH)
- **DVR:** Time-shifted playback buffer

### 6.2 Recommendation System
**Collaborative filtering + Content-based:**
- User watch history → similar videos
- Video features (tags, category) → related content
- Trending videos (global/regional)
- Machine learning models (can be separate microservice)

**Storage:** Pre-computed recommendations in Redis sorted sets

### 6.3 DRM (Digital Rights Management)
- Widevine (Chrome), PlayReady (Edge), FairPlay (Safari)
- License server integration
- Content encryption (AES-128)
- Token-based access control

### 6.4 Analytics
- View count, watch time, drop-off points
- Geographic distribution
- Device/OS breakdown
- Real-time dashboards (Kafka → Flink/Spark → DB/Redis)

## 7. Scalability Considerations

### 7.1 Horizontal Scaling
- Stateless API servers (scale behind load balancer)
- Database read replicas for queries
- Sharding by user_id or video_id for massive scale
- Microservices decomposition (upload, stream, search, social)

### 7.2 Storage Optimization
- **Cold storage:** Glacier/Archive for old/unpopular videos
- **Hot storage:** SSD-backed for popular content
- **Deduplication:** Identify duplicate uploads (hash comparison)
- **Compression:** Video codecs (H.265/AV1) for better compression

### 7.3 Performance Optimizations
- **Pre-warming:** Push popular content to CDN edge
- **Connection pooling:** Database and external services
- **Async processing:** Upload → queue → process (no blocking)
- **Client-side buffering:** 10-30 seconds ahead

## 8. Fault Tolerance & Reliability

- **Multi-region deployment:** Active-active or active-passive
- **Data replication:** 3x across regions
- **Health checks:** Auto-recovery for failed services
- **Circuit breakers:** Prevent cascade failures
- **Graceful degradation:** Lower quality if bandwidth low
- **Retry logic:** Exponential backoff for transient failures

## 9. Monitoring & Alerting

- **Metrics:** QPS, latency, error rates, transcoding queue depth
- **Logging:** Structured logs, centralized (ELK/Splunk)
- **Tracing:** Distributed tracing (Jaeger/Zipkin)
- **Alerts:** PagerDuty/Opsgenie for critical issues
- **Dashboards:** Grafana for real-time visibility

## 10. Cost Optimization

- **Storage tiering:** Move old videos to cheaper storage
- **CDN cost:** Negotiate pricing, use multiple providers
- **Reserved instances:** For predictable workloads
- **Spot instances:** For transcoding (fault-tolerant)
- **Compression:** Reduce bandwidth costs

## 11. Trade-offs

| Decision | Option A | Option B | Trade-off |
|----------|----------|----------|-----------|
| Storage | Replicated 3x | Erasure coding | Availability vs Cost |
| Transcoding | Real-time | Pre-transcode | Latency vs Compute Cost |
| CDN | Single provider | Multi-CDN | Simplicity vs Cost/Redundancy |
| Database | SQL | NoSQL | Consistency vs Scale |
| Search | In-house | Elasticsearch Service | Control vs Convenience |

## 12. Future Enhancements

- 360° video and VR support
- Interactive videos (chapters, quizzes)
- AI-generated thumbnails and titles
- Offline downloads (mobile)
- Social features (stories, shorts like TikTok)
- Live chat during streams
- Multi-language audio/subtitles
- Cloud gaming integration

## 13. References

- [YouTube Architecture](https://www.youtube.com/watch?v=G-lGEE4M7Yo)
- [Netflix Tech Blog](https://netflixtechblog.com/)
- [HLS Specification](https://tools.ietf.org/html/rfc8216)
- [DASH Specification](https://dashif.org/)

---

**Difficulty:** Hard 🟥
**Time to design:** 45-60 minutes
**Key focus areas:** Video processing, CDN, adaptive streaming, scalability