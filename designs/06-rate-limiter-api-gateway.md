# Rate Limiter / API Gateway System Design

## 1. Introduction

Design a rate limiting system that can be used as an API gateway or standalone service to protect APIs and services from abuse. The system must:
- Limit requests by user, IP, or API key
- Support different rate limiting algorithms
- Be distributed and scalable
- Provide real-time monitoring and alerts
- Allow dynamic configuration

## 2. Requirements

### Functional Requirements
- Rate limit requests per client (user/IP/api-key)
- Multiple time windows: per second, minute, hour, day
- Different limits for different endpoints/users
- Allow burst traffic (token bucket)
- Return proper HTTP status codes (429 Too Many Requests)
- Expose metrics and dashboards
- Dynamic rule updates without restart
- Block/blacklist abusive clients

### Non-Functional Requirements
- Low latency (< 1ms overhead)
- High availability (99.99%)
- Scalability to millions of requests/sec
- Distributed consistency
- Real-time monitoring

## 3. Capacity Estimation

**Assumptions:**
- 1 million requests per second peak
- 10 million unique clients (users/IPs) per day
- 1KB per counter in memory
- 1 hour TTL for counters

**Memory:**
- Active counters: 10M clients × 1KB = 10GB
- With replication: 20-30GB
- Fit in Redis cluster

**Throughput:**
- 1M req/sec → need distributed rate limiter
- Each check: read+increment+expire → 3 Redis ops
- 3M ops/sec → Redis cluster with 10-20 shards

## 4. Rate Limiting Algorithms

### 4.1 Fixed Window Counter
- Counter per time window (e.g., per minute)
- Reset counter at window boundary
- Simple but allows burst at window boundary

**Implementation:**
```python
key = f"rate:{client_id}:{endpoint}:{window_start}"
count = redis.incr(key)
if count == 1:
    redis.expire(key, window_size)
if count > limit:
    return False
```

### 4.2 Sliding Window Log
- Store timestamps of each request in sorted set
- Count requests in time window by removing old entries
- Accurate but memory-intensive

**Implementation:**
```python
now = time.time()
redis.zadd(key, {request_id: now})
redis.zremrangebyscore(key, 0, now - window_size)
count = redis.zcard(key)
if count > limit:
    return False
```

### 4.3 Sliding Window Counter (Hybrid)
- Combine fixed window + previous window count
- Approximate but memory-efficient

**Algorithm:**
```
current_window = floor(now / window_size)
previous_window = current_window - 1

current_count = get(rate:{client}:{current_window})
previous_count = get(rate:{client}:{previous_window})

# Weighted average based on time overlap
if previous_count exists:
    weight = (window_size - (now % window_size)) / window_size
    estimate = current_count + previous_count * weight
else:
    estimate = current_count

if estimate > limit:
    reject
else:
    increment current_count
```

### 4.4 Token Bucket
- Tokens added at constant rate (e.g., 10 tokens/sec)
- Each request consumes 1 token
- Allows burst up to bucket capacity
- Smooths traffic

**Implementation:**
```python
key = f"bucket:{client}"
now = time.time()

# Get current tokens and last refill time
tokens, last = redis.hgetall(key) or (capacity, now)
tokens = min(capacity, tokens + (now - last) * refill_rate)
last = now

if tokens >= 1:
    tokens -= 1
    redis.hmset(key, {"tokens": tokens, "last": last})
    redis.expire(key, ttl)
    return True
else:
    return False
```

**Recommended:** Token bucket for most use cases (allows burst)

## 5. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │   API   │  │  Web    │  │ Mobile  │  │ External│      │
│  │ Gateway │  │ Server  │  │  App    │  │  APIs   │      │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │           │           │           │              │
│       └───────────┴───────────┴───────────┘              │
└───────────────────────────┬───────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                Rate Limiter Service Layer                 │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Distributed Rate Limiter (Go/Python/Java)         │  │
│  │  • Parse request (IP, user, API key)               │  │
│  │  • Apply rules from config                        │  │
│  │  • Check Redis/DB counters                        │  │
│  │  • Return allow/deny                              │  │
│  └────────────────────────────┬────────────────────────┘  │
└───────────────────────────────┼───────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────▼─────────┐ ┌────────▼─────────┐ ┌────────▼─────────┐
│   Redis Cluster   │ │  Config Service  │ │  Monitoring     │
│   (Counters)      │ │  (etcd/ZooKeeper)│ │  (Prometheus)   │
└───────────────────┘ └──────────────────┘ └─────────────────┘
```

## 6. Component Deep Dive

### 6.1 Rate Limiter Service
**Deployment:** Sidecar or middleware in API gateway (Envoy, Kong, NGINX)

**Request flow:**
1. Receive HTTP request
2. Extract identifier: `user_id` (authenticated) or `IP` (anonymous)
3. Look up rate limit rules for this endpoint/user
4. For each rule (e.g., 100 req/min, 1000 req/hour):
   - Generate key: `rate:{identifier}:{endpoint}:{window}`
   - Query Redis for current count
   - Apply algorithm (token bucket, sliding window)
   - If any limit exceeded, return 429
5. If all passed, forward request to backend
6. Increment counters in Redis (async or sync)

**Key design:**
```
rate:{scope}:{identifier}:{endpoint}:{window}
Example: rate:user:john@example.com:/api/v1/users:60
```

**Scopes:**
- Global: all endpoints
- Endpoint: specific API path
- User group: premium vs free

**Performance:**
- Keep logic in-memory (rules cache)
- Use pipelined Redis commands
- Target < 1ms latency

### 6.2 Redis Cluster Setup

**Sharding:** Use consistent hashing on rate limit key
**Replication:** 1 master + 1 replica per shard
**Persistence:** AOF every second (balance durability vs performance)
**Memory:** 30GB cluster for counters + overhead

**Redis configuration:**
```
maxmemory 30gb
maxmemory-policy allkeys-lru
save 60 1
appendonly yes
appendfsync everysec
```

### 6.3 Rule Configuration

**Config format (YAML/JSON):**
```yaml
rules:
  - name: "authenticated_api"
    scope: "endpoint"
    identifier: "user_id"
    endpoint: "/api/v1/*"
    limits:
      - window: 60    # 1 minute
        limit: 100
      - window: 3600  # 1 hour
        limit: 1000
      - window: 86400 # 1 day
        limit: 10000

  - name: "ip_global"
    scope: "global"
    identifier: "ip"
    limits:
      - window: 60
        limit: 1000
```

**Dynamic updates:**
- Watch config file or config service (etcd)
- Hot-reload without restart
- Versioned rules with rollback

### 6.4 Monitoring & Alerting

**Metrics to expose (Prometheus format):**
```
# Counter metrics
rate_limit_requests_total{endpoint="/api/v1/users",result="allowed"}
rate_limit_requests_total{endpoint="/api/v1/users",result="denied"}

# Histogram
rate_latency_seconds{endpoint="/api/v1/users"}

# Gauge
rate_active_counters{scope="user"}
rate_redis_memory_bytes
```

**Dashboards:**
- Request rate by endpoint
- Denial rate (blocked requests)
- Top blocked clients
- Redis memory usage
- Latency percentiles

**Alerts:**
- Denial rate > 10% for 5 min
- Redis memory > 80%
- Rate limiter error rate > 1%

### 6.5 Blocklisting

- Maintain set of abusive clients in Redis: `blocklist:{scope}:{identifier}`
- Check before rate limiting (fast path)
- Auto-block after N denials in short period
- Manual block/unblock via admin API
- TTL for temporary blocks

## 7. Distributed Challenges

### 7.1 Clock Skew
- Use server's local time for window boundaries
- Accept slight skew between nodes (eventual consistency)
- Or use centralized time service (NTP sync)

### 7.2 Hot Keys
- Popular user or IP may hit same Redis key frequently
- Mitigation: Add random jitter to key or use multiple sub-keys

**Example:**
```
rate:user:123:api:60 → rate:user:123:api:60:0,1,2 (shard by hash)
```

### 7.3 Redis Failover
- Use Redis Sentinel for automatic failover
- Rate limiter should handle Redis errors gracefully:
  - On Redis down: fail open (allow all) or fail closed (deny all) based on policy
  - Cache negative results locally (short TTL)

### 7.4 Scalability
- Partition rate limit keys across Redis cluster
- Stateless rate limiter services scale horizontally
- Load balancer distributes requests

## 8. Integration Patterns

### 8.1 Middleware (API Gateway)
- Envoy, Kong, NGINX rate limiting modules
- Lua scripts in Redis for atomic operations
- Minimal latency overhead

**Envoy config:**
```yaml
rate_limits:
  - actions:
      - generic_key:
          descriptor: "user"
          value: "%REQ(user-id)%"
      - generic_key:
          descriptor: "path"
          value: "%PATH%"
    descriptor_key: "ratelimit"
```

### 8.2 Standalone Service
- Dedicated microservice
- Called via gRPC/HTTP before forwarding to backend
- Centralized rule management
- More flexibility

### 8.3 Client-Side
- SDK/library embedded in application
- Local rate limiting before sending request
- Reduces server load but can be bypassed

## 9. Advanced Features

### 9.1 Adaptive Rate Limiting
- Adjust limits based on system load
- During high load, reduce limits automatically
- Priority classes: premium users get higher limits

### 9.2 Quota Management
- Time-based quotas (e.g., 10,000 requests/month)
- Reset on billing cycle
- Usage tracking and alerts

### 9.3 Geo-Based Limiting
- Different limits per country/region
- Combat regional abuse

### 9.4 Cost-Based Throttling
- Expensive endpoints (compute-heavy) have stricter limits
- Dynamic based on current resource usage

### 9.5 Burst Mode
- Allow temporary exceedance of limit for critical requests
- Track burst credits (banking analogy)

## 10. Security Considerations

- **Client identification:** Use multiple factors (IP + user + API key)
- **Spoofing prevention:** Validate JWT/user tokens
- **Bypass protection:** Ensure rate limiter can't be skipped
- **DDoS protection:** Combine with WAF
- **Encryption:** TLS for all communications

## 11. Trade-offs

| Decision | Option A | Option B | Trade-off |
|----------|----------|----------|-----------|
| Algorithm | Token bucket | Sliding window log | Simplicity vs Accuracy |
| Storage | Redis | Database | Speed vs Durability |
| Scope | Per-user | Per-IP | Granularity vs Overhead |
| Failure mode | Fail open | Fail closed | Availability vs Protection |
| Distribution | Centralized | Per-service | Consistency vs Latency |

## 12. Example Implementation (Token Bucket)

**Redis key structure:**
```
key: "rate:{scope}:{identifier}:{endpoint}"
value: {
  "tokens": float,
  "last_refill": timestamp
}
```

**Lua script for atomicity:**
```lua
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2]) -- tokens per second
local now = tonumber(ARGV[3])

local bucket = redis.call('HMGET', key, 'tokens', 'last')
local tokens = tonumber(bucket[1]) or capacity
local last = tonumber(bucket[2]) or now

tokens = math.min(capacity, tokens + (now - last) * refill_rate)
last = now

if tokens >= 1 then
  tokens = tokens - 1
  redis.call('HMSET', key, 'tokens', tokens, 'last', last)
  redis.call('EXPIRE', key, 3600)
  return 1
else
  return 0
end
```

**Usage:**
```bash
EVAL script 1 rate:user:123:/api/v1/users 100 10 1700000000
# Returns 1 if allowed, 0 if denied
```

## 13. References

- [Rate Limiting Algorithms](https://en.wikipedia.org/wiki/Rate_limiting)
- [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket)
- [Designing a Scalable Rate Limiter](https://www.youtube.com/watch?v=FU4WlwfS3G0)
- [Redis Lua Scripting](https://redis.io/docs/manual/programmability/eval/)
- [Envoy Rate Limiting](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/rate_limit_filter)

---

**Difficulty:** Medium 🟡
**Time to design:** 30-40 minutes
**Key focus areas:** Distributed systems, algorithms, Redis, scalability, low latency