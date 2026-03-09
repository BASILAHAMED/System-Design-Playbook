# System Design Playbook 🎯

A comprehensive collection of system design case studies covering the most common and challenging interview questions in the tech industry. Each design includes detailed architecture diagrams, component breakdowns, scalability considerations, and trade-offs.

## 📋 What's Inside

This playbook covers **8 major system designs** that frequently appear in FAANG, Meta, Google, Amazon, Microsoft, and other top tech company interviews:

1. **[YouTube/Netflix - Video Streaming](designs/01-youtube-netflix-video-streaming.md)** 🎥
   - Video transcoding, adaptive bitrate streaming, CDN, global distribution
   - 15+ sections covering architecture, scalability, and cost optimization

2. **[Uber - Ride-Sharing Platform](designs/02-uber-ride-sharing.md)** 🚗
   - Real-time matching, geospatial queries, location tracking, surge pricing
   - Focus on real-time systems and matching algorithms

3. **[Twitter/X - Social Media](designs/03-twitter-social-media.md)** 🐦
   - Timeline generation (fan-out vs pull), feed algorithms, scalability
   - Hybrid approach for handling celebrities

4. **[TinyURL - URL Shortener](designs/04-tinyurl-url-shortener.md)** 🔗
   - Hash generation, caching strategies, high read-to-write ratios
   - Multiple algorithm comparisons

5. **[WhatsApp - Messaging System](designs/05-whatsapp-messaging.md)** 💬
   - Real-time messaging, WebSocket, offline queues, end-to-end encryption
   - Distributed message persistence

6. **[Rate Limiter - API Gateway](designs/06-rate-limiter-api-gateway.md)** 🛡️
   - Token bucket, sliding window, distributed rate limiting
   - Low-latency design with Redis

7. **[Google Docs - Collaborative Editor](designs/07-google-docs-collaboration.md)** 📝
   - Operational Transformation (OT), CRDTs, real-time sync
   - Conflict resolution and offline support

8. **[Distributed Key-Value Store](designs/08-distributed-key-value-store.md)** 🗄️
   - DynamoDB/Cassandra style, consistent hashing, LSM trees
   - CAP theorem, tunable consistency, replication

9. **[How ChatGPT Works](designs/09-how-chatgpt-works.md)** 🤖
   - Transformer architecture, tokenization, attention mechanisms
   - Training pipeline, fine-tuning, inference optimization

## 🎯 Interview Preparation

### Difficulty Levels
- 🟢 **Easy:** 30-40 minutes
- 🟡 **Medium:** 30-45 minutes
- 🔴 **Hard:** 45-60 minutes

### Common Interview Questions Covered
- "Design YouTube/Netflix"
- "Design Uber/Ola/Lyft"
- "Design Twitter/Facebook News Feed"
- "Design a URL shortener"
- "Design WhatsApp/Telegram/Messenger"
- "Design a rate limiter"
- "Design Google Docs/Notion/Figma"
- "Design a distributed key-value store (like DynamoDB/Cassandra)"
- "Design a web crawler"
- "Design a cache (like Redis)"
- "Design a payment system (like Stripe/PayPal)"
- "Design an e-commerce platform (like Amazon/Flipkart)"
- "Design a hotel booking system (like Oyo/Booking.com)"
- "Design a food delivery app (like Swiggy/Zomato)"
- "Design Netflix/YouTube video streaming"
- "Design Uber/Ola ride-sharing"
- "Design Twitter social media feed"
- "Design WhatsApp messenger"
- "Design Google Docs real-time collaboration"
- "Design a distributed key-value store"
- "Design a rate limiter"
- "Design a URL shortener"
- "Design a notification system"
- "Design a search autocomplete system"
- "Design a distributed cache"
- "Design a logging system"
- "Design a metrics monitoring system"
- "Design a task queue (like Celery)"
- "Design a file storage system (like Dropbox/Google Drive)"
- "Design ChatGPT/LLM architecture"
- "How does ChatGPT work internally"

## 📚 How to Use

### For Interviewees
1. **Read the design thoroughly** - Understand all components and their interactions
2. **Draw the architecture** - Practice sketching the diagrams from memory
3. **Deep dive into trade-offs** - Be prepared to explain why you chose X over Y
4. **Practice scalability questions** - "What if traffic increases 10x?"
5. **Learn the algorithms** - Understand OT vs CRDT, consistent hashing, token bucket, etc.

### For Interviewers
- Use these designs as reference for evaluating candidate responses
- Check if candidate covers: requirements, capacity estimation, core components, scalability, fault tolerance
- Look for awareness of trade-offs and alternatives

### Quick Reference
Each design includes:
- ✅ **Requirements** (Functional & Non-functional)
- ✅ **Capacity Estimation** (Back-of-envelope calculations)
- ✅ **System Architecture** (High-level diagram)
- ✅ **Component Deep Dive** (Detailed explanations)
- ✅ **Database Schema** (SQL/NoSQL design)
- ✅ **Scalability Considerations** (How to scale to millions/billions)
- ✅ **Advanced Features** (Nice-to-haves)
- ✅ **Fault Tolerance** (How system handles failures)
- ✅ **Monitoring & Alerting** (Observability)
- ✅ **Security Considerations**
- ✅ **Trade-offs Table** (Key decisions and their implications)
- ✅ **Future Enhancements** (What to build next)
- ✅ **References** (Further reading)

## 🏗️ Design Principles

All designs follow these principles:

1. **Start with Requirements** - Clarify functional and non-functional requirements
2. **Capacity Estimation** - Back-of-envelope calculations to size the system
3. **High-Level Architecture** - Component diagram with clear boundaries
4. **Deep Dives** - Detailed explanations of critical components
5. **Data Modeling** - Database schema with indexes
6. **Scalability** - How to scale horizontally
7. **Fault Tolerance** - Handle node/region failures
8. **Trade-offs** - Every design decision has pros/cons

## 🛠️ Common Patterns

You'll see these patterns across multiple designs:

- **Caching:** Redis/Memcached for hot data
- **Message Queues:** Kafka/RabbitMQ for async processing
- **CDN:** CloudFront/Cloudflare for static assets
- **Load Balancing:** Round-robin, consistent hashing
- **Database Partitioning:** Sharding by user_id or key hash
- **Replication:** Master-slave, multi-master
- **Consistency Models:** Strong, eventual, quorum-based
- **Rate Limiting:** Token bucket, sliding window

## 📖 Algorithm Cheat Sheet

| Algorithm | Use Case | Trade-offs |
|-----------|----------|------------|
| **Consistent Hashing** | Partitioning/distribution | Load balancing, minimal movement on changes |
| **Token Bucket** | Rate limiting | Allows burst, simple |
| **Sliding Window Log** | Accurate rate limiting | Memory intensive |
| **LSM Tree** | Write-heavy storage | Read amplification, compaction overhead |
| **Bloom Filter** | Fast negative lookups | False positives, no false negatives |
| **Merkle Tree** | Efficient data synchronization | Tree maintenance overhead |
| **Operational Transformation** | Real-time collaborative editing | Central coordinator needed |
| **CRDTs** | Decentralized collaboration | Metadata overhead |

## 🎓 Prerequisites

Before diving into these designs, ensure you understand:

- **Networking:** HTTP/HTTPS, TCP/IP, DNS, CDNs, Load Balancers
- **Databases:** SQL vs NoSQL, indexing, partitioning, replication
- **Caching:** Redis, cache invalidation strategies
- **Message Queues:** Kafka, RabbitMQ, pub/sub patterns
- **Distributed Systems:** CAP theorem, consensus (Paxos/Raft), gossip protocols
- **Cloud Services:** AWS/GCP/Azure offerings (S3, EC2, CloudFront, etc.)

## 🚀 Getting Started

1. **Clone this repository**
   ```bash
   git clone https://github.com/BASILAHAMED/System-Design-Playbook.git
   ```

2. **Pick a design** - Start with something familiar (like URL shortener) before tackling harder ones (video streaming, distributed databases)

3. **Practice explaining** - Try to explain the design to a friend or record yourself

4. **Draw diagrams** - Use tools like Excalidraw, Lucidchart, or even pen & paper

5. **Identify weaknesses** - Think about what could fail and how you'd improve it

## 📊 Comparison Table

| System | Write-heavy | Read-heavy | Real-time | Consistency | Scalability |
|--------|------------|------------|-----------|-------------|-------------|
| Video Streaming | Medium | Very High | Medium | Eventual | Very High |
| Ride-Sharing | High | High | Very High | Strong | High |
| Social Media | Very High | Very High | High | Eventual | Very High |
| URL Shortener | Low | Very High | Low | Strong | Very High |
| Messaging | Very High | Very High | Very High | Strong | Very High |
| Rate Limiter | High | High | High | Eventual | Very High |
| Collaborative Editor | Very High | Very High | Very High | Strong | Medium |
| Key-Value Store | Very High | Very High | High | Tunable | Very High |
| ChatGPT/LLM | Medium | High | High | Strong | Very High |

## 🔄 Updates & Contributions

This playbook is actively maintained. Contributions welcome!

**Found a bug? Have a suggestion?**
- Open an issue: https://github.com/BASILAHAMED/System-Design-Playbook/issues
- Submit a PR with improvements

## 📚 Further Reading

### Books
- **Designing Data-Intensive Applications** by Martin Kleppmann (Bible of system design)
- **System Design Interview** by Alex Xu (Vol 1 & 2)
- **Designing Distributed Systems** by Brendan Burns

### Blogs & Resources
- [High Scalability](http://highscalability.com/) - Real-world architecture case studies
- [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/)
- [Netflix Tech Blog](https://netflixtechblog.com/)
- [Uber Engineering](https://eng.uber.com/)
- [Google Research](https://research.google/)

### YouTube Channels
- [System Design Interview](https://www.youtube.com/c/SystemDesignInterview)
- [Gaurav Sen](https://www.youtube.com/c/GauravSen)
- [Hussein Nasser](https://www.youtube.com/c/HusseinNasser-software-engineering)

## 📝 License

MIT License - feel free to use, share, and modify.

## 🙏 Acknowledgments

Inspired by countless system design interviews, engineering blogs, and the open-source community. Special thanks to all the engineers who've shared their knowledge online.

---

**Happy designing! May your interviews be smooth and your architectures scalable. 🚀**

*Last updated: March 2025*