# System Design Notes

A first-principles system-design revision guide.

## Topics

### 01. Caching

- [Caching 101](caching.md)
  - What is caching?
  - Why caching exists: latency, capacity, expensive computation
  - Cache economics
  - Why caches are faster than databases
  - Database vs. cache
  - When an in-memory KV store can be a primary database
  - Cache locality: temporal and spatial
  - Cache hits, misses, and hit rate
  - Cache-aside
  - Read-through
  - Write-through
  - Write-back / write-behind
  - TTL (Time To Live)
  - Explicit invalidation
  - Refresh-ahead
  - Stale-while-revalidate
  - Cache stampede / thundering herd
  - Hot keys
  - Cache eviction: LRU, LFU, FIFO
  - Cache sizing and working set
  - Cache key design
  - What to cache: data, query results, computations, API responses
  - Cache consistency and freshness
  - Cache invalidation strategies
  - Consistency races
  - Cache and database failure
  - Distributed caches
  - Partitioning and consistent hashing
  - Cache replication
  - Multi-level caching
  - Local vs. distributed cache
  - Cache penetration
  - Cache avalanche
  - Negative caching
  - Cache warming
  - Serialization
  - Cache object size and memory overhead
  - Cache security
  - Cache observability
  - Caching in system-design interviews

### 02. Real-Time Architecture

- Real-time systems
- Latency and freshness requirements
- Polling
- Long polling
- Server-Sent Events (SSE)
- WebSockets
- Push vs. pull
- Connection management
- Real-time fan-out
- Pub/Sub
- Message brokers
- Ordering
- Delivery semantics
- Backpressure
- Presence and ephemeral state
- Scaling real-time connections
- Partitioning and sharding
- Failure handling and reconnection
- Idempotency
- End-to-end latency
- Real-time system observability

### 03. Distributed Systems Fundamentals

- Scalability
- Availability
- Reliability
- Durability
- Latency and throughput
- Horizontal vs. vertical scaling
- Stateless vs. stateful services
- Load balancing
- Partitioning and sharding
- Replication
- Consistency models
- CAP theorem
- Quorum
- Failure modes
- Idempotency
- Distributed coordination

### 04. Databases

- Relational vs. NoSQL databases
- Indexes
- B-trees and hash indexes
- Query execution
- Transactions
- ACID
- MVCC
- Isolation levels
- Locking
- Replication
- Read replicas
- Partitioning / sharding
- Database scaling
- Connection pooling
- Database bottlenecks
- OLTP vs. OLAP

### 05. Messaging and Event-Driven Systems

- Queues vs. streams
- Pub/Sub
- Kafka fundamentals
- Partitions
- Consumer groups
- Ordering
- Delivery semantics
- At-most-once / at-least-once / exactly-once
- Retries
- Dead-letter queues
- Backpressure
- Event-driven architecture
- Event sourcing
- CQRS

### 06. APIs and Service Communication

- REST
- RPC / gRPC
- Synchronous vs. asynchronous communication
- API gateways
- Service discovery
- Timeouts
- Retries
- Exponential backoff
- Circuit breakers
- Bulkheads
- Rate limiting
- Idempotency
- Request tracing

### 07. Storage and Data Systems

- Object storage
- Block storage
- File storage
- SSD vs. HDD
- Storage hierarchy
- Data locality
- Replication
- Erasure coding
- Data lifecycle
- Cold vs. hot storage
- CDN and edge caching

### 08. Reliability and Resilience

- Failure domains
- Redundancy
- Health checks
- Failover
- Graceful degradation
- Load shedding
- Backpressure
- Circuit breakers
- Disaster recovery
- RPO / RTO
- Multi-region architecture
- Chaos engineering

### 09. Observability

- Logs
- Metrics
- Traces
- Distributed tracing
- SLIs
- SLOs
- SLAs
- Error budgets
- Alerting
- High-cardinality data
- Debugging distributed systems

### 10. System Design Interview Framework

- Requirements clarification
- Functional vs. non-functional requirements
- Capacity estimation
- Traffic estimation
- Storage estimation
- API design
- Data model
- High-level architecture
- Bottleneck identification
- Scaling strategy
- Consistency trade-offs
- Failure modes
- Security considerations
- Observability
- Trade-off discussion

> **Status:** Caching is the first topic developed in detail. The remaining sections are the roadmap for the concepts we will work through from first principles.
