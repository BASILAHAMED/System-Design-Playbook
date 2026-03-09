# Distributed Key-Value Store (Dynamo/Cassandra) System Design

## 1. Introduction

Design a distributed key-value store like Amazon DynamoDB or Apache Cassandra. The system must:
- Store and retrieve data by key with low latency
- Scale horizontally to petabytes of data
- Provide high availability (no single point of failure)
- Handle node failures gracefully
- Support tunable consistency levels
- Be partition-tolerant (follows CAP theorem - AP system)

## 2. Requirements

### Functional Requirements
- Put(key, value) - store data
- Get(key) - retrieve data
- Delete(key) - remove data
- Range queries (optional)
- Secondary indexes (optional)

### Non-Functional Requirements
- Very low latency (< 10ms for reads, < 20ms for writes)
- High availability (99.99%)
- Linear scalability (add nodes → more capacity)
- Fault tolerance (survive node/rack failures)
- Consistency tunability (per operation)
- Eventually consistent by default

## 3. Capacity Estimation

**Assumptions:**
- 100 TB of data
- 1 billion keys
- Average value size: 10 KB
- 10 million operations per second (5M reads, 5M writes)
- Peak: 20M ops/sec

**Storage:**
- Raw data: 100TB
- Replication factor 3: 300TB
- Compaction overhead: +20% = 360TB total

**Throughput:**
- 20M ops/sec
- Each node can handle 10K ops/sec → need 2000 nodes

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │   App   │  │  Web    │  │ Mobile  │  │ Service │      │
│  │ Server  │  │ Server  │  │  App    │  │   Worker│      │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │           │           │           │              │
│       └───────────┴───────────┴───────────┘              │
└───────────────────────────┬───────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                    Request Router                          │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Partition/Coordinator (per partition)              │  │
│  │  • Compute partition for key (consistent hashing)  │  │
│  │  • Route request to appropriate nodes              │  │
│  │  • Handle replication and consistency              │  │
│  └────────────────────────────┬────────────────────────┘  │
└───────────────────────────────┼───────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────▼─────────┐ ┌────────▼─────────┐ ┌────────▼─────────┐
│   Node (Peer)     │ │   Node (Peer)    │ │   Node (Peer)   │
│                   │ │                  │ │                 │
│ • Storage engine  │ │ • Storage engine │ │ • Storage engine│
│ • Memtable (LSM)  │ │ • Memtable (LSM) │ │ • Memtable (LSM)│
│ • SSTables       │ │ • SSTables      │ │ • SSTables     │
│ • Hinted handoff │ │ • Hinted handoff│ │ • Hinted handoff│
│ • Compaction      │ │ • Compaction     │ │ • Compaction    │
└───────────────────┘ └──────────────────┘ └─────────────────┘
```

## 5. Core Concepts

### 5.1 Partitioning (Sharding)

**Consistent Hashing:**
- Map keys to positions on a ring (hash(key) mod 2^160)
- Each node owns one or more ranges on the ring
- Adding/removing nodes affects minimal data movement

**Virtual Nodes:**
- Each physical node owns multiple virtual nodes (tokens)
- Helps with load balancing (some nodes may have more data)
- Recommended: 100-1000 virtual nodes per physical node

**Partition key calculation:**
```
partition_key = hash(key) mod num_virtual_nodes
```

### 5.2 Replication

**Replication Factor (N):** Typically 3
- Each piece of data stored on N nodes
- Coordinator node (handling request) writes to all N replicas

**Replication strategies:**
- **SimpleStrategy:** Place replicas on next N nodes in ring (Cassandra)
- **NetworkTopologyStrategy:** Place replicas in different racks/datacenters

**Write quorum (W):** Minimum replicas that must acknowledge write
**Read quorum (R):** Minimum replicas that must respond to read
**Consistency:** R + W > N ensures strong consistency

**Example:** N=3, W=2, R=2 → always read latest write (quorum)

### 5.3 Data Model

**Column-family (wide-column) store:**
```
Row Key: user123
Columns:
  name: "John Doe"
  email: "john@example.com"
  created_at: "2024-01-01"
  last_login: "2024-12-01"
```

**Dynamic columns:** Each row can have different columns
**Sorted by column name** for efficient range queries

**Cassandra CQL:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name TEXT,
    email TEXT,
    created_at TIMESTAMP
);
```

### 5.4 Storage Engine (LSM Tree)

**Log-Structured Merge Tree:**
- Writes go to in-memory memtable (sorted)
- When memtable full, flush to disk as SSTable (immutable, sorted)
- Reads check memtable + multiple SSTables (bloom filters to skip)
- Background compaction merges SSTables, removes overwritten/deleted data

**Components:**
- **Memtable:** In-memory write buffer (size: 64-256MB)
- **SSTable:** Sorted string table on disk
- **Bloom filter:** Probabilistic filter to check if key exists in SSTable
- **Compaction:** Merge SSTables to reduce file count and reclaim space

**Write path:**
1. Write to commit log (for durability)
2. Write to memtable
3. If memtable full, flush to SSTable

**Read path:**
1. Check memtable
2. Check bloom filters → fetch relevant SSTables
3. Merge results (memtable + SSTables)

### 5.5 Write/Read Flow

**Write request:**
1. Client sends Put(key, value) to any node (coordinator)
2. Coordinator computes partition key → identifies N replica nodes
3. Sends write to all N replicas (parallel)
4. Waits for W acknowledgements
5. Returns success to client

**Read request:**
1. Client sends Get(key) to any node (coordinator)
2. Coordinator computes partition key → identifies N replica nodes
3. Sends read to all N replicas (or R if read repair disabled)
4. Waits for R responses
5. Returns most recent value (based on timestamp)
6. If inconsistencies detected, trigger read repair (async)

**Hinted Handoff:**
- If a replica is down, coordinator stores "hint" locally
- When replica recovers, hint is replayed
- Allows writes during temporary failures

### 5.6 Failure Handling

**Node failure detection:**
- Gossip protocol (like SWIM) - nodes exchange health info
- Failure detector marks node as down after timeout

**When node recovers:**
- Other nodes stream missing data (hinted handoff + anti-entropy)
- Merkle trees to compare data efficiently
- Repair in background

**Rack/Data center failure:**
- Use NetworkTopologyStrategy to place replicas in different failure domains
- Clients can specify datacenter local quorum for lower latency

### 5.7 Consistency Tuning

**Consistency levels per operation:**
- **ONE:** Wait for 1 replica (fast, eventual consistency)
- **QUORUM:** Wait for majority (R or W = floor(N/2)+1)
- **ALL:** Wait for all replicas (strong consistency, slower)
- **LOCAL_QUORUM:** Quorum within local datacenter
- **ANY:** Write accepted even if all replicas down (hinted handoff)

**Trade-off:** Higher consistency → higher latency, lower availability

## 6. Database Schema Example

**Cassandra-style table:**

```sql
-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email TEXT,
    name TEXT,
    created_at TIMESTAMP,
    last_login TIMESTAMP,
    preferences MAP<TEXT, TEXT>
);

-- Tweets table (partition by user_id, cluster by created_at)
CREATE TABLE tweets (
    user_id UUID,
    tweet_id TIMEUUID, -- time-based UUID for sorting
    content TEXT,
    media_urls LIST<TEXT>,
    likes INT,
    retweets INT,
    PRIMARY KEY (user_id, tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);

-- Timeline (denormalized)
CREATE TABLE timeline (
    user_id UUID,
    tweet_id TIMEUUID,
    author_id UUID,
    content TEXT,
    PRIMARY KEY (user_id, tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);
```

## 7. Scalability Considerations

### 7.1 Adding Nodes
- New node joins ring, gets assigned virtual nodes
- Other nodes stream relevant data ranges to new node
- Rebalancing happens in background

### 7.2 Hot Partitions
- Problem: Some keys get disproportionate traffic
- Solution: Add random suffix to partition key (salting)
  - `partition_key = hash(key + random_suffix)`
  - Query all suffixes and merge

### 7.3 Large Partitions
- Problem: One partition grows too large (> 100GB)
- Solution: Split partition key (composite key)
  - `PRIMARY KEY ((user_id, bucket), timestamp)`
  - Bucket based on time or hash

## 8. Advanced Features

### 8.1 Secondary Indexes
- Global secondary index (GSI): Separate table with indexed column as partition key
- Local secondary index (LSI): Same partition key, different clustering key
- Trade-off: Indexes hurt write performance, need maintenance

### 8.2 Materialized Views
- Pre-computed queries for common access patterns
- Automatically updated on base table changes
- Use for denormalization

### 8.3 Batch Operations
- Batch writes across multiple partitions
- Atomic within a partition only
- Logged batch for replayability

### 8.4 Lightweight Transactions
- Compare-and-set (CAS) for linearizable consistency
- Uses Paxos/consensus protocol
- Slower than normal writes (4 round trips)

## 9. Monitoring & Alerting

- **Metrics:** Read/write latency (p50, p99), throughput, error rates, compaction times
- **Alerts:** Node down, high disk usage, replication lag
- **Gossip:** Cluster state, token distribution
- **Compaction:** Duration, pending tasks

## 10. Security

- **Authentication:** Kerberos, SSL certificates
- **Authorization:** Role-based access control (RBAC)
- **Encryption:** TLS for transit, at-rest encryption
- **Audit logging:** All operations logged

## 11. Trade-offs

| Decision | Option A | Option B | Trade-off |
|----------|----------|----------|-----------|
| Consistency | Strong | Eventual | Availability vs Consistency |
| Replication | Synchronous | Asynchronous | Consistency vs Latency |
| Indexes | Global | Local | Flexibility vs Performance |
| Compaction | SizeTiered | Leveled | Write vs Read amplification |
| Partitioning | Consistent hashing | Range-based | Load balancing vs Hotspots |

## 12. References

- [Amazon DynamoDB Paper](https://www.allthingsdistributed.com/2007/10/amazons-dynamo.html)
- [Apache Cassandra Architecture](https://cassandra.apache.org/doc/latest/architecture/)
- [Designing Data-Intensive Applications](https://dataintensive.net/) (Martin Kleppmann)
- [LSM Trees](https://en.wikipedia.org/wiki/Log-structured_merge-tree)

---

**Difficulty:** Hard 🔴
**Time to design:** 45-60 minutes
**Key focus areas:** Partitioning, replication, consistency, LSM trees, fault tolerance, CAP theorem