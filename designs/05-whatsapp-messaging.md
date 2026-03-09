# WhatsApp / Telegram - Messaging System Design

## 1. Introduction

Design a real-time messaging platform like WhatsApp or Telegram that supports:
- One-on-one and group messaging
- Real-time delivery
- End-to-end encryption
- Media sharing (images, videos, documents)
- Push notifications
- Presence/online status
- Message history sync

## 2. Requirements

### Functional Requirements
- Send/receive text messages in real-time
- Group chats (up to 256 participants)
- Media attachments (images, videos, documents, voice notes)
- Read receipts and delivery status
- Online/last seen status
- Push notifications for offline users
- Message history and sync across devices
- Contact discovery
- Voice/video calls (optional)
- Message reactions and replies
- Message editing and deletion
- Broadcast lists

### Non-Functional Requirements
- Real-time delivery (< 100ms for online users)
- High availability (99.99%)
- Message ordering guarantee
- End-to-end encryption (E2EE)
- Scalability to billions of users
- Offline message queuing
- Low latency globally
- Message persistence

## 3. Capacity Estimation

**Assumptions:**
- 2 billion total users
- 500 million daily active users (DAU)
- 100 billion messages per day (~1.15M/sec)
- Average message: 100 bytes (text), media ~100KB
- 10% of messages have media attachments
- Group chats: 20% of messages
- Peak: 3x average = 3.45M/sec

**Storage:**
- Text messages: 90% × 100B × 100B/day = 9TB/day
- Media: 10% × 100KB × 100B/day = 1PB/day (heavy!)
- Metadata: ~10% of text size = 0.9TB/day
- 1 year: ~330TB (text+metadata) + 365PB (media) → media dominates

**Bandwidth:**
- Inbound: 100B/day = 10TB/day (text)
- Media upload: 1PB/day
- Outbound (delivery): Same as inbound + media download

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Mobile  │  │  Web    │  │ Desktop │  │   API   │      │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │           │           │           │              │
│       └───────────┴───────────┴───────────┘              │
└───────────────────────────┬───────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                    Connection Layer                        │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  WebSocket Servers (Socket.io/WebRTC)              │  │
│  │  • Connection management                           │  │
│  │  • Heartbeat/presence                              │  │
│  │  • Message routing                                 │  │
│  └────────────────────────────┬────────────────────────┘  │
└───────────────────────────────┼───────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────▼─────────┐ ┌────────▼─────────┐ ┌────────▼─────────┐
│  Auth Service     │ │  Message Service │ │  Presence        │
│                   │ │                  │ │  Service         │
│ • JWT/OAuth       │ │ • Send message   │ │ • Status updates │
│ • Device auth     │ │ • Store message  │ │ • Last seen      │
└─────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────┐
│               Processing & Storage Layer                  │
│  ┌─────────────┐ ┌────────────┐ ┌────────────┐         │
│  │   Message   │ │   Media    │ │   Push     │         │
│  │    Queue    │ │  Storage   │ │ Notification│         │
│  │  (Kafka)    │ │  (S3/CDN)  │ │  Service   │         │
│  └─────────────┘ └────────────┘ └────────────┘         │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │           Message Database (Cassandra/              │ │
│  │           PostgreSQL with partitioning)            │ │
│  │  • Messages (by chat/user)                         │ │
│  │  • Chat metadata                                   │ │
│  └─────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
```

## 5. Component Deep Dive

### 5.1 Connection Management (WebSocket)
- Clients maintain persistent WebSocket connection to nearest server
- Connection servers are stateless; session state stored in Redis
- Heartbeat every 30 seconds to detect disconnects
- On disconnect, buffer messages for offline user

**Server responsibilities:**
- Authenticate connection (JWT token)
- Assign connection ID
- Route incoming messages to appropriate recipients
- Broadcast presence updates

**Load balancing:**
- Use consistent hashing to map user → WebSocket server
- On reconnect, user routed to same server (if possible) for in-memory state

### 5.2 Message Flow

**1:1 Messaging:**
```
Sender → WS Server → Message Queue → 
  → Persist to DB → 
  → If recipient online: route via WS → recipient
  → If offline: store in offline queue + push notification
```

**Group Messaging:**
```
Sender → WS Server → Message Queue →
  → Persist to DB (once) →
  → For each recipient:
      - If online: route via WS
      - If offline: store in offline queue + push notification
```

**Message format:**
```json
{
  "message_id": "uuid",
  "chat_id": "group_id or user_pair_hash",
  "sender_id": "user_id",
  "type": "text|image|video|document|audio",
  "content": "Hello",
  "media_url": "https://...", // if media
  "timestamp": 1234567890,
  "read_by": ["user1", "user2"], // for group read receipts
  "reactions": [{"user": "u1", "emoji": "👍"}]
}
```

### 5.3 Message Storage

**Challenge:** Message volume is enormous (100B/day). Need distributed storage.

**Approach 1: Chat-based partitioning**
- Partition messages by chat_id (1:1 chat or group_id)
- Each partition stored on multiple nodes
- Use Cassandra for write scalability and partitioning

**Cassandra schema:**
```cql
-- Messages table (wide row per chat)
CREATE TABLE messages (
    chat_id UUID,
    message_id TIMEUUID, // time-based UUID for sorting
    sender_id UUID,
    type TEXT,
    content TEXT,
    media_url TEXT,
    metadata MAP<TEXT, TEXT>,
    PRIMARY KEY (chat_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Fetch last N messages for a chat
SELECT * FROM messages WHERE chat_id = ? LIMIT 50;

-- Chat metadata
CREATE TABLE chats (
    chat_id UUID PRIMARY KEY,
    type TEXT, -- 'dm' or 'group'
    participants SET<UUID>,
    created_at TIMESTAMP,
    last_message_id TIMEUUID,
    unread_count MAP<UUID, INT> // per user
);
```

**Approach 2: User-centric partitioning**
- Partition by user_id (all messages involving user)
- Faster to fetch user's entire history
- More storage (duplicate messages for 1:1 chats)

**Hybrid:** Store by chat_id, maintain per-user pointers to latest messages for quick sync

### 5.4 Offline Message Queue

When recipient is offline:
- Store message in per-user offline queue (Redis list)
- Also persisted in main DB
- On user reconnect:
  1. Send all offline messages from Redis queue
  2. Acknowledge and remove from queue
  3. If Redis empty, fetch from DB (last N messages)

**Redis structure:**
```
key: offline:{user_id}
value: [message_id1, message_id2, ...]
```

**Size limit:** Keep only last 1000 messages in Redis, older in DB

### 5.5 Media Handling

**Upload flow:**
1. Client requests upload URL from API
2. API generates pre-signed URL for S3 (or object storage)
3. Client uploads directly to storage (bypass app servers)
4. Storage triggers processing (thumbnail generation, virus scan)
5. Processing stores processed files and sends message with media_url

**Media storage:**
- S3 with CloudFront CDN
- Path structure: `/{user_id}/{chat_id}/{timestamp}_{uuid}`
- Retention: Keep forever unless deleted by user
- Deduplication: Hash-based (same file uploaded once)

**Thumbnails:**
- Generate for images/videos (different resolutions)
- Store separately, reference in message

### 5.6 Push Notifications

For offline users:
- After message persisted, send to push notification service
- FCM (Android) / APNs (iOS) / Web Push
- Payload includes: sender, preview, chat_id, message_id
- Tapping notification opens app and fetches message

**Rate limiting:** Don't spam - aggregate notifications for group chats

### 5.7 End-to-End Encryption (E2EE)

**Protocol:** Signal Protocol (Double Ratchet)
- Each user has key pair (identity key, signed prekey, one-time prekeys)
- Session established between users via key exchange
- Messages encrypted with symmetric keys that ratchet forward
- Forward secrecy: Compromised device can't decrypt past messages
- Future secrecy: Compromised device can't decrypt future messages

**Implementation:**
- Client-side encryption/decryption only
- Server stores only encrypted blobs
- Server cannot read message content
- Key management: Local storage + backup (optional)

**Challenges:**
- Group encryption: Sender encrypts message with each recipient's key
- Multi-device sync: Keys need to be shared across devices
- Key backup/recovery: User-controlled or service-managed

**Simplified (without E2EE):**
- TLS in transit
- Server-side encryption at rest
- Acceptable for non-privacy-critical apps

### 5.8 Presence & Last Seen

**Online status:**
- WebSocket connection active → online
- Heartbeat interval: 30 seconds
- Store in Redis: `user:{id}:online = true` with TTL 60s
- When heartbeat received, refresh TTL
- When connection closes, set to false

**Last seen:**
- Update `user:{id}:last_seen` timestamp on disconnect
- Privacy settings: hide last seen from specific users

**Broadcast:**
- When user status changes, publish to presence channel
- All contacts receive update (via WebSocket or push)

## 6. Scalability Considerations

### 6.1 WebSocket Server Scaling
- Stateless servers; session in Redis
- Load balancer with sticky sessions (IP hash)
- Scale horizontally to thousands of servers
- Each server handles ~50K concurrent connections (memory bound)

### 6.2 Database Scaling
- **Cassandra:** Naturally scalable, add nodes, data rebalancing
- **Partition key:** chat_id (ensures even distribution)
- **Replication factor:** 3 for HA
- **Consistency level:** QUORUM for writes/reads

### 6.3 Media Storage Scaling
- Object storage (S3) is inherently scalable
- CDN distributes globally
- No bottleneck

### 6.4 Message Queue Scaling
- Kafka cluster with multiple partitions
- Partition by chat_id or user_id
- Consumers can be scaled horizontally

## 7. Advanced Features

### 7.1 Message Reactions & Replies
- Store reactions as separate message-like objects with reference to original
- Update read receipts for reactions
- Show in UI with counts

### 7.2 Message Editing & Deletion
- Edit: Update content in DB, mark as edited
- Delete: Soft delete (mark as deleted) for all participants
- "Delete for everyone" within time window (e.g., 1 hour)
- Sync delete to all devices

### 7.3 Voice/Video Calls
- WebRTC for peer-to-peer media
- Signaling via WebSocket
- STUN/TURN servers for NAT traversal
- Call metadata stored in DB

### 7.4 Contact Discovery
- Upload phone contacts (hashed) to server
- Server matches hashes to find registered users
- Return matched contacts
- Privacy: Use private set intersection or upload phone numbers encrypted

## 8. Fault Tolerance & Reliability

- **Multi-region:** Deploy in multiple regions, use global load balancer
- **Database replication:** Cross-region replication (eventual consistency)
- **Message durability:** Write-ahead logs, replication factor 3
- **Client-side queuing:** If offline, messages queued locally and sent on reconnect
- **Exactly-once delivery:** Idempotent message IDs, deduplication

## 9. Monitoring & Alerting

- **Metrics:** Messages/sec, online users, avg latency, queue depth
- **Alerts:** High message drop rate, WebSocket disconnects spike
- **Dashboards:** Active users, message volume heatmap, top media consumers

## 10. Security Considerations

- **E2E encryption** (if implemented)
- **TLS 1.3** for all connections
- **Authentication:** Strong password + 2FA
- **Rate limiting:** Per user/IP to prevent spam
- **Content scanning:** Media for malware, CSAM detection
- **GDPR:** Right to delete, export data
- **Device management:** List active devices, revoke sessions

## 11. Trade-offs

| Decision | Option A | Option B | Trade-off |
|----------|----------|----------|-----------|
| Storage | Per-chat partition | Per-user partition | Query patterns vs duplication |
| Encryption | E2EE | TLS only | Privacy vs features (cloud backup) |
| Offline queue | Redis + DB | DB only | Speed vs durability |
| Media | S3 | Self-hosted | Cost vs control |
| Presence | WebSocket | Polling | Real-time vs battery |

## 12. Future Enhancements

- Stories (disappearing after 24h)
- Payment integration (send money)
- In-chat bots and AI assistants
- Message scheduling
- Auto-translation
- Message pinning
- Chat folders/navigation
- Desktop app with local database
- Decentralized architecture (Matrix protocol)

## 13. References

- [WhatsApp Architecture](https://www.youtube.com/watch?v=3G5bVWg1K5w)
- [Signal Protocol](https://signal.org/docs/)
- [Designing a Scalable Chat Application](https://www.youtube.com/watch?v=7rD_KJz0Kqw)
- [Cassandra Data Modeling](https://cassandra.apache.org/doc/latest/data_modeling/)

---

**Difficulty:** Hard 🔴
**Time to design:** 45-60 minutes
**Key focus areas:** Real-time messaging, WebSocket, message persistence, offline queue, E2EE, scalability