# TinyURL / Bitly - URL Shortener System Design

## 1. Introduction

Design a URL shortening service like TinyURL or Bitly that converts long URLs into short, shareable links. The system must:
- Generate unique short codes for URLs
- Redirect users to original URLs quickly
- Handle high read/write volumes
- Support custom short URLs
- Track click analytics
- Set expiration dates

## 2. Requirements

### Functional Requirements
- Shorten a long URL to a short URL
- Redirect from short URL to original URL (301/302 redirect)
- Custom short URLs (user-defined alias)
- URL expiration (optional)
- Click tracking and analytics
- User accounts (optional, for managing links)
- API for programmatic access

### Non-Functional Requirements
- Very high read-to-write ratio (100:1 typical)
- Low latency redirects (< 100ms)
- High availability (99.99%)
- Scalability to billions of short URLs
- Persistent storage (no data loss)
- Security (prevent abuse, malicious URLs)

## 3. Capacity Estimation

**Assumptions:**
- 100 million new URLs per month (~3.3M/day, ~38/sec)
- 10:1 read:write ratio → 1 billion redirects/month (~11,500/sec)
- Average URL length: 100 characters
- Average short URL: 10 characters
- Storage: 5 years of data

**Storage:**
- URLs: 100M/month × 60 months = 6B records
- Each record: short_code (10B) + original_url (500B) + metadata (100B) ≈ 610B
- Total: 6B × 610B ≈ 3.66TB (raw)
- With indexes: ~6TB

**Memory (cache):**
- Active URLs: 1B × 610B = 610GB → too large
- Cache hot 10%: 100M × 610B ≈ 61GB (Redis cluster)

**Bandwidth:**
- Redirects: 1B/day × 1KB (HTTP headers) = 1TB/day
- Writes: 3.3M/day × 1KB = 3.3GB/day

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Mobile  │  │  Web    │  │   API   │  │  Admin  │      │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │           │           │           │              │
│       └───────────┴───────────┴───────────┘              │
└───────────────────────────┬───────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                    API Gateway Layer                       │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Load Balancer + Rate Limiting                      │  │
│  └────────────────────────────┬────────────────────────┘  │
└───────────────────────────────┼───────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────▼─────────┐ ┌────────▼─────────┐ ┌────────▼─────────┐
│  Shorten Service  │ │ Redirect Service │ │  Analytics       │
│                   │ │                  │ │  Service         │
│ • Generate code    │ │ • Lookup code    │ │ • Track clicks   │
│ • Check custom    │ │ • 301 redirect   │ │ • Generate stats │
│ • Store URL       │ │ • Update stats   │ │                  │
└─────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────┐
│               Processing & Storage Layer                  │
│  ┌─────────────┐ ┌────────────┐ ┌────────────┐         │
│  │   Cache     │ │   Hash     │ │   DB       │         │
│  │  (Redis)    │ │  Service   │ │(PostgreSQL/│         │
│  │             │ │            │ │  Cassandra)│         │
│  └─────────────┘ └────────────┘ └────────────┘         │
└───────────────────────────────────────────────────────────┘
```

## 5. Component Deep Dive

### 5.1 Short Code Generation

**Requirements:**
- Unique, non-sequential (security through obscurity)
- Fixed length (typically 6-8 characters)
- URL-safe encoding (A-Z, a-z, 0-9 → base62)
- Check for custom alias availability

**Approaches:**

#### A) Base62 Encoding of Auto-increment ID
- Use database auto-increment integer ID
- Encode to base62: `encode(id)` → short code
- Pros: Short, unique, no collisions
- Cons: Predictable sequence (can be mitigated by starting at random offset)

**Example:** ID 1000 → "g8" (base62)

#### B) Hash-based (MD5/SHA-1) with collision resolution
- Take hash of original URL (or random salt+URL)
- Take first N characters (e.g., 7 chars from base62 of hash)
- If collision, retry with different salt or increment counter
- Pros: Fixed length, no sequential pattern
- Cons: Collisions require handling, longer codes

**Example:** `base62(hash(url)[0:7])`

#### C) Random Generation
- Generate random 7-char base62 string
- Check DB for uniqueness, retry if collision
- Pros: Unpredictable
- Cons: Collision probability increases with scale

**Recommended:** Base62 of auto-increment ID with random starting offset

**Implementation:**
```python
# Encoding
def encode(base62, num):
    chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    result = []
    while num > 0:
        num, rem = divmod(num, 62)
        result.append(chars[rem])
    return ''.join(reversed(result))

# Decoding
def decode(base62, s):
    chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    num = 0
    for c in s:
        num = num * 62 + chars.index(c)
    return num
```

**Database sequence:**
```sql
CREATE SEQUENCE url_id_seq START WITH 1000000; -- random offset
```

### 5.2 Redirect Service

**Flow:**
1. User clicks `https://short.url/abc123`
2. Load balancer routes to redirect server
3. Server extracts short code `abc123`
4. Look up in cache (Redis): `GET url:abc123`
5. If cache miss, query database: `SELECT original_url FROM urls WHERE short_code = 'abc123'`
6. If found, store in cache (with TTL) and return 301/302 redirect
7. If not found, return 404
8. Increment click count (async via message queue)

**Redirect type:**
- 301 (Moved Permanently): SEO benefit, cached by browsers
- 302 (Found): Temporary, allows analytics tracking, A/B testing
- Usually 301 for public URLs, 302 for private/tracked

**Performance:**
- Cache hit rate > 99% for popular URLs
- Redis GET: ~1ms
- DB query: ~10ms (with index)
- Total latency: < 50ms typical

### 5.3 Database Design

**PostgreSQL schema:**

```sql
-- Main URL mapping
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    user_id UUID REFERENCES users(id), -- nullable for anonymous
    custom_alias BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    expires_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    click_count INT DEFAULT 0
);

-- Create index on short_code (already unique)
CREATE INDEX idx_urls_short_code ON urls(short_code);
CREATE INDEX idx_urls_user_id ON urls(user_id);
CREATE INDEX idx_urls_expires_at ON urls(expires_at) WHERE is_active = TRUE;

-- Click tracking (optional, can be aggregated)
CREATE TABLE clicks (
    id BIGSERIAL PRIMARY KEY,
    url_id BIGINT REFERENCES urls(id),
    ip_address INET,
    user_agent TEXT,
    referrer TEXT,
    country_code CHAR(2),
    clicked_at TIMESTAMP DEFAULT NOW()
);
-- Partition by month for scale
CREATE TABLE clicks_2025_03 PARTITION OF clicks 
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');
```

**Alternative for massive scale:** Use Cassandra
```
CREATE TABLE urls (
    short_code text PRIMARY KEY,
    original_url text,
    user_id uuid,
    created_at timestamp,
    expires_at timestamp,
    click_count counter
);
```

### 5.4 Caching Strategy

**Redis structure:**
```
key: url:{short_code}
value: {original_url}
TTL: 24h (for hot URLs) or no expiry (manual eviction)
```

**Cache warm-up:**
- Pre-populate cache for popular URLs (top 1M)
- On write, also write to cache (write-through)

**Cache eviction:**
- LRU policy
- Max memory: 50-100GB
- Monitor hit rate, adjust size

**Analytics cache:**
```
key: stats:{url_id}:daily:{date}
value: {clicks, unique_ips, countries...}
TTL: 30 days
```

### 5.5 Analytics Service

**Track per click:**
- Timestamp
- IP address → geolocation (country, city)
- User agent → device, browser, OS
- Referrer (where click came from)
- Cookies for unique visitor counting

**Aggregation:**
- Real-time: Increment Redis counters (per URL, per day)
- Batch: Periodically flush to DB for long-term storage
- Daily rollup: Total clicks, unique users, top countries

**API endpoints:**
- GET /analytics/{short_code}?range=day/week/month
- Return JSON: {clicks, unique_visitors, top_referrers, top_countries, timeline}

**Implementation:**
1. On redirect, log click to Kafka (async, non-blocking)
2. Consumer writes to:
   - Redis real-time counters (fast)
   - Click events table (for detailed analysis)
3. Daily batch job aggregates into summary tables

### 5.6 Custom Aliases
- User can choose their own short code (e.g., `yourbrand/promo`)
- Validate: available, meets length/character requirements, not offensive
- Mark `custom_alias = TRUE` in DB
- Prevent abuse: rate limit custom alias requests

### 5.7 Security & Abuse Prevention
- **Rate limiting:** Per IP/user on shorten endpoint (e.g., 100/hour)
- **URL validation:** Check for malicious URLs (phishing, malware)
- **Blocklist:** Disallow short codes that are offensive or reserved
- **Captcha:** For unauthenticated users after threshold
- **Spam detection:** Patterns of similar URLs, mass creation
- **Expiring URLs:** Auto-deactivate after TTL

## 6. Scalability Considerations

### 6.1 Database Scaling
- **Sharding:** Shard by short_code prefix (first 2 chars) or user_id
- **Read replicas:** For redirect queries (read-heavy)
- **Connection pooling:** PgBouncer for PostgreSQL
- **Index optimization:** Covering indexes for redirect query

### 6.2 Cache Scaling
- **Redis Cluster:** Shard by short_code hash
- **Replication:** Master-replica with failover
- **Memory:** Estimate 10% of URLs hot → scale accordingly

### 6.3 API Scaling
- **Stateless services:** Scale horizontally behind load balancer
- **Rate limiting:** Distributed (Redis) to prevent abuse
- **CDN:** For static assets (docs, landing page)

### 6.4 High Availability
- **Multi-region:** Active-active for redirect service (DNS failover)
- **Database:** Primary-replica with automatic failover
- **Cache:** Redis Sentinel or Redis Cluster
- **Queue:** Kafka with replication

## 7. Advanced Features

### 7.1 Bulk URL Shortening
- Upload CSV of URLs → return mapping
- Async processing via queue
- Notification when ready

### 7.2 QR Code Generation
- Generate QR code for short URL
- Cache QR images on CDN

### 7.3 Password Protection
- Require password to access redirected URL
- Store encrypted password hash
- Prompt on redirect landing page

### 7.4 Link Expiration
- Set expiry date/time
- After expiry, return 410 Gone or redirect to custom page
- Background job cleans up expired URLs

### 7.5 Link Editing
- Users can update target URL for existing short code
- Keep click history, update `original_url`
- Audit log of changes

### 7.6 Team/Enterprise Features
- Organization accounts
- Shared link libraries
- Role-based access (admin, member, viewer)
- Branded domains (company.short)
- SSO integration

## 8. Fault Tolerance & Reliability

- **Redundant infrastructure:** Multiple availability zones
- **Health checks:** Auto-restart failed services
- **Graceful degradation:** If cache down, fall back to DB (slower)
- **Data backup:** Daily snapshots, point-in-time recovery
- **Disaster recovery:** Cross-region replication

## 9. Monitoring & Alerting

- **Metrics:** QPS, latency (p50, p99), error rates, cache hit rate
- **Alerts:** Redirect errors > threshold, DB connection pool exhaustion
- **Dashboards:** Top shortened URLs, click trends, active users

## 10. Security Considerations

- **Input validation:** Sanitize URLs, prevent XSS in redirect pages
- **HTTPS everywhere:** TLS for all endpoints
- **Authentication:** API keys, OAuth for user accounts
- **Authorization:** Users can only modify their own URLs
- **Abuse monitoring:** Detect and block spam campaigns
- **GDPR compliance:** Allow users to delete their data

## 11. Trade-offs

| Decision | Option A | Option B | Trade-off |
|----------|----------|----------|-----------|
| Code generation | Base62 of ID | Random hash | Predictable vs collision risk |
| Redirect type | 301 Permanent | 302 Temporary | SEO vs Analytics |
| Storage | SQL | NoSQL (Cassandra) | Consistency vs Scale |
| Cache | Redis | Memcached | Features vs Simplicity |
| Analytics | Real-time | Daily batch | Freshness vs Complexity |

## 12. Future Enhancements

- AI-powered link preview generation
- Link scheduling (publish at specific time)
- Geographically targeted redirects
- Device-specific redirects (mobile vs desktop)
- Link retargeting pixels for marketing
- API rate limits per user
- Webhook notifications on click events
- Bulk import/export
- Link bundles (multiple URLs in one)

## 13. References

- [System Design: URL Shortener](https://www.youtube.com/watch?v=JQDHz72OA_0)
- [Bitly Engineering Blog](https://medium.com/@bitly)
- [TinyURL Wikipedia](https://en.wikipedia.org/wiki/TinyURL)
- [Designing a Scalable URL Shortening Service](https://www.geeksforgeeks.org/system-design-url-shortening-service/)

---

**Difficulty:** Medium 🟡
**Time to design:** 30-40 minutes
**Key focus areas:** Hash generation, caching, high read-to-write ratio, redirect optimization