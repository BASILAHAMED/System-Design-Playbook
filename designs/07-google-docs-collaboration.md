# Google Docs - Real-time Collaborative Editor System Design

## 1. Introduction

Design a real-time collaborative document editing platform like Google Docs or Figma where multiple users can:
- Edit the same document simultaneously
- See each other's changes in real-time
- Resolve conflicts automatically
- Maintain cursor positions and selections
- Support rich text, images, comments

## 2. Requirements

### Functional Requirements
- Create, edit, delete documents
- Real-time collaboration (multiple users)
- See other users' cursors/selections in real-time
- Version history and rollback
- Comments and suggestions
- Rich text formatting
- Images and media embedding
- Access control (view, edit, comment)
- Offline editing with sync
- Export to various formats (PDF, DOCX)

### Non-Functional Requirements
- Real-time sync (< 100ms latency)
- Conflict resolution (eventual consistency)
- High availability (99.99%)
- Scalability to millions of documents
- Offline support
- Low bandwidth usage
- Operational transformation or CRDTs for consistency

## 3. Capacity Estimation

**Assumptions:**
- 100 million total documents
- 10 million daily active users
- 1 million concurrent editors (peak)
- Average document: 10KB
- Average edit operation: 100 bytes
- 10 edits per second per active document = 10M ops/sec peak

**Storage:**
- Documents: 100M × 10KB = 1TB
- Version history: 10× overhead = 10TB
- Operations log: 10M ops/sec × 100B × 3600 × 12 = 43TB/day
- Compressed and pruned: keep 30 days = ~1.3PB (raw) → need compaction

**Bandwidth:**
- Real-time ops: 10M ops/sec × 100B = 1GB/sec = 86TB/day
- Initial document load: 1M users × 10KB = 10GB/sec (peak)

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Web     │  │ Mobile  │  │ Desktop │  │  CLI    │      │
│  │ Browser │  │  App    │  │  App    │  │  Tool   │      │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │           │           │           │              │
│       └───────────┴───────────┴───────────┘              │
└───────────────────────────┬───────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                    Sync Gateway Layer                      │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  WebSocket Servers (per-document rooms)             │  │
│  │  • Connection management                            │  │
│  │  • Broadcast operations                             │  │
│  │  • Presence (cursors, selections)                   │  │
│  └────────────────────────────┬────────────────────────┘  │
└───────────────────────────────┼───────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────▼─────────┐ ┌────────▼─────────┐ ┌────────▼─────────┐
│  Operation        │ │  Document Cache  │ │  Presence        │
│  Service          │ │  (Redis)         │ │  Service         │
│                   │ │                  │ │                  │
│ • Transform ops   │ │ • Hot documents  │ │ • Cursor pos     │
│ • Resolve conflicts│ │ • Version cache  │ │ • User list      │
└─────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────┐
│               Processing & Storage Layer                  │
│  ┌─────────────┐ ┌────────────┐ ┌────────────┐         │
│  │   Operation │ │   Version   │ │   Document  │         │
│  │    Log      │ │   Store     │ │   Store     │         │
│  │ (Kafka)     │ │(PostgreSQL/ │ │ (S3/Blob)  │         │
│  │             │ │  Cassandra) │ │            │         │
│  └─────────────┘ └────────────┘ └────────────┘         │
└───────────────────────────────────────────────────────────┘
```

## 5. Component Deep Dive

### 5.1 Client-Server Communication

**WebSocket per document:**
- When user opens document, connect to room `doc:{document_id}`
- Send operations (insert, delete, format) as JSON messages
- Receive operations from other users
- Apply operations locally using OT/CRDT
- Send heartbeats for presence

**Message format:**
```json
{
  "type": "operation",
  "client_id": "uuid",
  "version": 42,
  "operation": {
    "type": "insert",
    "position": 150,
    "text": "Hello"
  }
}
```

**Presence:**
```json
{
  "type": "cursor",
  "user_id": "user123",
  "name": "Alice",
  "position": 150,
  "selection": [145, 160],
  "color": "#ff0000"
}
```

### 5.2 Operational Transformation (OT) vs CRDTs

**OT (Operational Transformation):**
- Transform operations against concurrent edits
- Requires central server to serialize and transform
- Complex but bandwidth-efficient
- Used by Google Docs

**CRDTs (Conflict-free Replicated Data Types):**
- Each operation is commutative
- No central coordination needed
- Simpler but higher metadata overhead
- Used by Figma, Notion

**Recommendation:** OT for text editing (smaller ops), CRDT for complex objects

**OT Example:**
```
User A: Insert "X" at pos 0 → "Xabc"
User B: Insert "Y" at pos 0 → "Yabc"
Transformation: B's op transforms against A's → Insert "Y" at pos 1
Result: "YXabc"
```

### 5.3 Operation Service

**Responsibilities:**
- Receive operations from clients
- Validate and transform against current version
- Broadcast to other clients in same document
- Persist to operation log
- Manage version numbers

**Flow:**
1. Client sends op with base version (v)
2. Server fetches all ops after v from operation log
3. Transform incoming op against those concurrent ops
4. Increment version, assign new version number
5. Broadcast transformed op to all clients (including sender for ack)
6. Append to operation log (Kafka)
7. Periodically compact to snapshot

**Conflict resolution:**
- Last-write-wins for formatting ops (timestamp)
- Transform for text ops
- User identity for cursor positions

### 5.4 Document Storage

**Two approaches:**

#### A) Operation Log + Snapshots
- Store all operations (append-only)
- Periodically create full document snapshot
- To load: get latest snapshot + apply ops after
- Pros: Full history, easy rollback
- Cons: Applying many ops can be slow

#### B) Versioned Document Store
- Each version stored as complete document
- Optimistic concurrency with version numbers
- Merge conflicts using OT/CRDT
- Pros: Fast reads
- Cons: Storage overhead

**Hybrid:** Store snapshots every 100 ops, plus full operation log

**Storage schema (PostgreSQL):**
```sql
-- Documents
CREATE TABLE documents (
    id UUID PRIMARY KEY,
    title VARCHAR(500),
    owner_id UUID REFERENCES users(id),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    current_version INT DEFAULT 0
);

-- Snapshots
CREATE TABLE snapshots (
    doc_id UUID REFERENCES documents(id),
    version INT,
    content JSONB, -- full document state
    PRIMARY KEY (doc_id, version)
);

-- Operations
CREATE TABLE operations (
    id BIGSERIAL PRIMARY KEY,
    doc_id UUID REFERENCES documents(id),
    version INT,
    client_id UUID,
    operation JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_operations_doc_version ON operations(doc_id, version);
```

**Alternative:** Use Cassandra for ops log (time-series), PostgreSQL for snapshots

### 5.5 Caching

**Redis usage:**
- Hot document snapshots: `doc:{id}:snapshot`
- Active document room members: `doc:{id}:users`
- User presence: `user:{id}:cursor`
- Operation cache: recent ops per document (to reduce DB fetches)

**Cache invalidation:**
- On new op: update snapshot cache asynchronously
- TTL: 1 hour for snapshots
- Room members: real-time via pub/sub

### 5.6 Offline Support

**Client-side:**
- Queue operations while offline
- Assign temporary client-side version numbers
- On reconnect: send queued ops with base version
- Server transforms and returns new versions
- Client re-applies any remote ops that happened during offline

**Conflict resolution:**
- If ops can't be transformed cleanly, merge at character level
- Last-write-wins for formatting (by timestamp)
- Show conflict markers for manual resolution (like Git)

## 6. Scalability Considerations

### 6.1 WebSocket Scaling
- Each document room is independent
- Use consistent hashing to map doc_id → WebSocket server
- Server only maintains state for active documents
- Scale horizontally; each server handles 10K-50K connections

### 6.2 Operation Log Scale
- 10M ops/sec peak
- Kafka cluster with 100+ partitions
- Partition by doc_id
- Consumers: operation service, analytics, backup

### 6.3 Document Store Scale
- Shard documents by doc_id hash
- Read replicas for snapshot queries
- Hot documents cached in Redis
- Cold documents in S3 with lifecycle to Glacier

### 6.4 Real-time Broadcast
- Use Redis Pub/Sub for intra-server broadcast
- When server receives op, publish to `doc:{id}:ops`
- All servers subscribed to relevant channels
- Avoids fan-out through central server

## 7. Advanced Features

### 7.1 Version History
- Store periodic snapshots + all ops
- Allow jump to any version
- Diff between versions
- Named versions (bookmarks)
- Restore to previous version

### 7.2 Comments & Suggestions
- Anchored to document position (character offset)
- Resolved comments hidden but retained
- Suggestions: proposed edits that can be accepted/rejected
- Store as separate entity linked to document version

### 7.3 Access Control
- Document-level permissions: view, edit, comment, admin
- Shareable links with different permissions
- Expiring links
- Domain-based sharing (for teams)
- Real-time permission updates

### 7.4 Real-time Presence
- Show avatars of active users
- Cursor positions and selections
- Color-coded per user
- Update frequency: 2-5 seconds (throttled)
- Presence history (who edited what)

### 7.5 Rich Text & Formatting
- Use a data model like Quill Delta or ProseMirror
- Operations on text runs with formatting attributes
- Merge formatting conflicts (last-write-wins with timestamp)
- Undo/redo stack per client

## 8. Fault Tolerance & Reliability

- **Multi-region:** Active-active for WebSocket layer (clients connect to nearest)
- **Operation log:** Kafka replication factor 3
- **Document store:** Cross-region replication (async)
- **Redis:** Redis Cluster with replicas
- **Client-side:** Local storage of unsynced ops
- **Recovery:** Rebuild state from operation log if needed

## 9. Monitoring & Alerting

- **Metrics:** Concurrent users, ops/sec, latency (p50, p99), sync lag
- **Alerts:** High operation failure rate, WebSocket disconnect spikes
- **Dashboards:** Active documents heatmap, top editors, conflict rate

## 10. Security Considerations

- **Authentication:** OAuth 2.0, JWT
- **Authorization:** Document ACLs checked on every operation
- **Encryption:** TLS for transit, at-rest encryption for documents
- **Input validation:** Sanitize rich text to prevent XSS
- **Audit log:** Record all operations for compliance
- **GDPR:** Export/delete user data

## 11. Trade-offs

| Decision | OT | CRDT | Trade-off |
|----------|----|------|-----------|
| Consistency | Strong (server-coordinated) | Eventual | Complexity vs Decentralization |
| Bandwidth | Low | High (metadata) | Efficiency vs Simplicity |
| Conflict resolution | Centralized | Automatic | Control vs User experience |
| Offline support | Complex | Natural | | 

**OT chosen for text** due to efficiency.

## 12. Future Enhancements

- AI-assisted writing (grammar, style)
- Voice typing and transcription
- Real-time translation
- Document comparison/merge tool
- Plugins and extensions
- Mobile app with offline-first
- Integration with cloud storage (Google Drive, OneDrive)
- Electronic signature integration
- Real-time co-presentation mode

## 13. References

- [Google Docs Architecture](https://www.youtube.com/watch?v=8DvyyWbYrjk)
- [Operational Transformation](https://en.wikipedia.org/wiki/Operational_transformation)
- [CRDTs](https://crdt.tech/)
- [Figma's Real-time Architecture](https://www.figma.com/blog/how-figma-uses-crdts/)
- [Designing Real-time Collaborative Editors](https://www.youtube.com/watch?v=6vmRwC5Rk0E)

---

**Difficulty:** Hard 🔴
**Time to design:** 45-60 minutes
**Key focus areas:** Real-time sync, OT/CRDT, conflict resolution, WebSocket, scalability