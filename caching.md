# Caching 101

## A First-Principles Guide to Caching in Distributed Systems

------------------------------------------------------------------------

## 1. What is caching?

A cache is a **faster, usually more expensive copy of data or the result
of computation**, kept so that future requests can avoid repeating
expensive work.

The fundamental question is:

> **Can I avoid doing the same work repeatedly by storing the result
> somewhere faster?**

Caching is not inherently about Redis, RAM, or databases. Those are
implementation choices.

------------------------------------------------------------------------

## 2. Why does caching exist?

There are three primary motivations.

### 2.1 Reduce latency

Memory access is generally much faster than disk access or a trip
through a database query engine.

The exact latency depends heavily on the system, hardware, workload, and
network path.

### 2.2 Increase capacity

Suppose a database can comfortably handle 10,000 reads/sec but the
application receives 100,000 reads/sec.

If 90% of those reads can be served from cache:

``` text
100,000 requests
      |
      +---- 90,000 --> cache
      |
      +---- 10,000 --> DB
```

The cache has reduced the DB's read workload by 90%.

### 2.3 Reduce expensive computation

The underlying work doesn't have to be a DB lookup. It could be:

-   complex SQL
-   aggregation
-   recommendation generation
-   ML inference
-   geospatial computation
-   external API calls
-   rendering
-   expensive business logic

The cache stores the result so subsequent requests can reuse it.

------------------------------------------------------------------------

## 3. The key economic principle

A cache is usually **more expensive per GB** than database storage.

RAM is expensive; SSD/object storage is much cheaper.

Therefore, caching is not primarily about cheaper storage.

Instead:

> **The cost of maintaining a fast copy can be justified by the cost of
> repeatedly performing the underlying work.**

A simplified model is:

``` text
Benefit
≈ requests × hit rate × cost of avoided work

Cost
≈ cache infrastructure
 + network
 + operational complexity
 + consistency/invalidation cost
```

Caching makes sense when the benefit exceeds the cost.

Also, "cost of avoided work" does not necessarily mean literal dollars
per query. It can mean scarce DB CPU, IOPS, connections, latency budget,
or database capacity.

------------------------------------------------------------------------

## 4. Why is a cache faster than a database?

It is tempting to think:

``` text
Database = disk
Cache    = RAM
```

That is incomplete.

Modern databases keep frequently accessed data in memory.

A PostgreSQL read may conceptually look like:

``` text
Query
  |
  v
Query execution
  |
  v
Index
  |
  v
Buffer pool
  |
  v
RAM
  |
  +--> possibly SSD
```

A simple key-value cache lookup may look like:

``` text
GET user:123
     |
     v
hash table
     |
     v
RAM
     |
     v
value
```

Even when both are completely memory-resident, a cache can be faster
because it performs less work.

A database provides things such as:

-   transactions
-   MVCC
-   concurrency control
-   durability
-   indexes
-   query processing
-   constraints
-   recovery
-   rich query semantics

A simple KV cache may only need:

``` text
key -> value
```

Therefore:

> **A cache is fast partly because it deliberately solves a narrower
> problem.**

------------------------------------------------------------------------

## 5. Why not use the cache as the database?

There is nothing fundamentally wrong with using an in-memory KV system
as a primary datastore.

The real question is:

> **What guarantees and access patterns does the application require?**

A traditional database may provide:

-   transactions
-   durability
-   rich queries
-   secondary indexes
-   constraints
-   concurrency control
-   crash recovery

A simple KV system provides something closer to:

``` text
GET(key)
SET(key, value)
```

If the workload only needs the latter, a KV database may be perfectly
appropriate.

But if the application needs arbitrary relational queries, joins,
complex predicates, transactions, or strong durability guarantees, a
database designed for those requirements is more appropriate.

The important insight is:

> **"Database" and "cache" describe roles and guarantees more than they
> describe physical hardware.**

------------------------------------------------------------------------

## 6. The defining property of a cache

A cache generally contains a **reconstructible copy**.

A useful mental model is:

> **If I delete the cache, can I rebuild it from the source of truth?**

If yes, it behaves like a cache.

If deleting it destroys irreplaceable business data, it is functioning
as a primary datastore and should be designed with appropriate
durability and recovery guarantees.

------------------------------------------------------------------------

## 7. Cache locality

Caching works because workloads often exhibit locality.

### Temporal locality

If something was requested recently, it is likely to be requested again
soon.

### Spatial locality

If one item is accessed, nearby or related items may also be accessed.

The important point is:

> **Caching is useful when the workload has enough reuse to make keeping
> a fast copy worthwhile.**

A workload that randomly accesses every item once may have a very low
cache hit rate and gain little from caching.

------------------------------------------------------------------------

## 8. Cache hit and cache miss

A **cache hit** means the requested data is present.

``` text
Request
   |
   v
 Cache
   |
   v
 HIT
   |
   v
Response
```

A **cache miss** means it isn't.

``` text
Request
   |
   v
 Cache
   |
   v
 MISS
   |
   v
DB / source
   |
   v
Cache
   |
   v
Response
```

### Cache hit rate

``` text
hit rate = cache hits / total cache requests
```

For example:

``` text
100,000 requests
95,000 hits
 5,000 misses

Hit rate = 95%
```

Hit rate is important, but the more meaningful question is:

> **How much expensive underlying work did the cache actually
> eliminate?**

------------------------------------------------------------------------

## 9. Cache-aside

The most common caching pattern is **cache-aside**.

The application explicitly manages the cache.

### Read

``` text
Application
     |
     v
   Cache
   /    hit   miss
  |      |
  v      v
return   DB
          |
          v
        Cache
          |
          v
        return
```

### Write

Typically:

``` text
Application
     |
     +----> DB
     |
     +----> invalidate cache
```

The application decides:

-   what to cache
-   how long
-   when to invalidate
-   what consistency is acceptable

------------------------------------------------------------------------

## 10. Read-through cache

With read-through caching, the application talks to the cache and the
cache handles misses.

``` text
Application
     |
     v
   Cache
   /    hit   miss
  |      |
  v      v
return   DB
          |
          v
        Cache
```

The difference is ownership of the miss path:

### Cache-aside

The application knows how to fetch from the DB.

### Read-through

The cache layer owns the DB lookup.

Read-through can simplify application code but requires a cache layer
capable of integrating with the underlying datastore.

------------------------------------------------------------------------

## 11. Write-through cache

Writes go through the cache.

``` text
Application
     |
     v
   Cache
    |
    +----> update cache
    |
    +----> update DB
```

### Advantage

The cache can be kept fresh as part of the write path.

### Problem

There are now multiple systems participating in the write:

``` text
Cache update succeeds
DB update fails
```

or the reverse.

That creates a distributed consistency problem.

------------------------------------------------------------------------

## 12. Write-back / write-behind

The application writes to the cache first and the DB is updated
asynchronously.

``` text
Application
     |
     v
   Cache
     |
     +---- async ----> DB
```

### Advantage

Very fast writes.

### Risk

If the cache loses the data before it is persisted, the DB may never
receive the write.

Therefore write-back requires careful durability, replication, and
recovery design.

------------------------------------------------------------------------

## 13. TTL --- Time To Live

A cache entry can have an expiration time.

``` text
product:123
TTL = 300 seconds
```

After expiration:

``` text
entry expires
    |
    v
cache miss
    |
    v
fetch fresh value
```

A TTL effectively says:

> **"I am willing to serve data that is up to N seconds old."**

TTL is one of the simplest ways to trade freshness for simplicity.

------------------------------------------------------------------------

## 14. Explicit invalidation

Instead of waiting for TTL:

``` text
DB update
   |
   v
invalidate cache
```

For example:

``` text
UPDATE product 123
       |
       v
DELETE product:123 from cache
```

The next read repopulates it.

The challenge is that the DB update and cache invalidation are separate
operations. If invalidation fails, stale data can remain.

This is why cache invalidation is fundamentally a distributed
consistency problem.

------------------------------------------------------------------------

## 15. Refresh-ahead

Instead of waiting for expiration, refresh popular entries proactively.

``` text
Cache entry
     |
     | approaching expiry
     v
background refresh
     |
     v
DB
```

This can prevent a popular key from becoming a cache miss at the worst
possible moment.

------------------------------------------------------------------------

## 16. Stale-while-revalidate

Serve slightly stale data while refreshing asynchronously.

``` text
Request
   |
   v
Stale cache
   |
   +--------> return immediately
   |
   +--------> background refresh
```

This prioritizes latency over perfect freshness.

It is useful for:

-   homepages
-   product catalogs
-   feeds
-   recommendations
-   public content

------------------------------------------------------------------------

## 17. Cache stampede

Suppose a popular entry expires.

``` text
50,000 requests
       |
       v
50,000 cache misses
       |
       v
50,000 DB queries
```

This is a **cache stampede** or **thundering herd**.

Common mitigations:

### Request coalescing

Only one request regenerates the value.

``` text
50,000 requests
       |
       v
one DB request
       |
       v
populate cache
       |
       v
50,000 responses
```

### Distributed locking

One process acquires a lock to regenerate the value.

### Stale-while-revalidate

Continue serving the old value while refreshing.

### Jittered TTLs

Avoid synchronized expiration.

### Refresh-ahead

Refresh before expiration.

------------------------------------------------------------------------

## 18. Hot keys

A hot key is a disproportionately popular cache entry.

For example:

``` text
celebrity:123
```

might receive millions of requests.

Even in a distributed cache:

``` text
Redis 1
Redis 2
Redis 3
Redis 4
```

a single key may be assigned to one node and make that node a
bottleneck.

Possible approaches include:

-   local application caching
-   replication
-   request coalescing
-   distributing reads
-   special handling of very hot data

Important principle:

> **Caching doesn't eliminate bottlenecks; it can move them.**

------------------------------------------------------------------------

## 19. Cache eviction

Cache memory is finite.

Eventually something must be removed.

### LRU --- Least Recently Used

Remove the item that has not been accessed recently.

### LFU --- Least Frequently Used

Remove items with low access frequency.

### FIFO

Remove the oldest entries.

### TTL

Entries disappear after their configured lifetime.

Real systems may combine multiple mechanisms.

------------------------------------------------------------------------

## 20. Cache sizing

The goal is not necessarily to cache the entire database.

Instead, identify the **working set**: the portion of the dataset that
generates most requests.

For example:

``` text
Database: 10 TB

Frequently accessed working set: 200 GB

Cache: ~200 GB
```

Cache sizing depends on:

-   dataset size
-   access distribution
-   object size
-   TTL
-   eviction policy
-   hit-rate target
-   replication factor
-   memory overhead

Actual memory footprint is usually larger than logical data size because
of hash tables, metadata, pointers, allocator overhead, and
fragmentation.

------------------------------------------------------------------------

## 21. Cache key design

Good keys are:

-   deterministic
-   unique
-   easy to construct
-   easy to invalidate

Examples:

``` text
user:123
product:456
recommendations:user:123
```

For compound results:

``` text
search:products:iphone:sort=price:page=1
```

Beware of:

-   huge keys
-   inconsistent key construction
-   collisions
-   unbounded cardinality
-   keys that are impossible to invalidate

------------------------------------------------------------------------

## 22. What exactly can be cached?

### Raw data

``` text
user:123 -> user object
```

### Query results

``` text
orders:user:123:last30days -> result
```

### Computation

``` text
recommendations:user:123 -> [A,B,C]
```

### API response

``` text
GET /popular-products -> complete response
```

### External API result

``` text
geo:lat:lng -> location
```

The key question is:

> **What expensive or repeated work am I trying to avoid?**

------------------------------------------------------------------------

## 23. Caching and consistency

Caching creates a second copy.

Therefore you need to decide:

> **How stale is acceptable?**

Possible requirements include:

### Strong freshness

Users should essentially always see the latest value.

Caching becomes harder.

### Bounded staleness

Data may be up to a known amount of time old.

TTL often works well.

### Eventual consistency

Temporary staleness is acceptable.

Caching becomes much easier.

A system-design interview should clarify this requirement before
choosing a cache strategy.

------------------------------------------------------------------------

## 24. Cache invalidation strategies

Common approaches:

### Time-based

``` text
TTL
```

### Event-based

``` text
DB change
   |
   v
event
   |
   v
invalidate cache
```

### Write-time

``` text
write DB
write/update cache
```

### Version-based

``` text
user:123:v17
```

A new version naturally makes the old representation obsolete.

------------------------------------------------------------------------

## 25. Cache consistency races

Consider:

``` text
Initial:
DB = A
Cache = A
```

One operation writes `B` while another reads.

Depending on timing, the reader may see `A` even though the DB now
contains `B`.

More complicated races occur when cache updates and DB updates happen in
different orders.

Therefore:

> **Caching is fundamentally a distributed consistency problem once you
> have multiple copies.**

------------------------------------------------------------------------

## 26. Cache and database failure

A robust system should define what happens when the cache disappears.

Ideally:

``` text
Cache failure
     |
     v
Application
     |
     v
DB
```

The system continues functioning, perhaps more slowly.

But if cache failure causes all traffic to fall through to the DB:

``` text
Cache failure
     |
     v
DB traffic increases
     |
     v
DB overload
     |
     v
Higher latency
     |
     v
Requests pile up
     |
     v
System failure
```

This is why cache failure must be included in capacity planning.

------------------------------------------------------------------------

## 27. Distributed caches

A single cache server is eventually limited by:

-   memory
-   CPU
-   network
-   throughput

So caches are commonly distributed:

``` text
             Application
                  |
          +-------+-------+
          |       |       |
          v       v       v
        Node 1  Node 2  Node 3
```

Keys need to be distributed across nodes.

A common approach is **consistent hashing**.

Conceptually:

``` text
hash(key)
    |
    v
choose cache node
```

------------------------------------------------------------------------

## 28. Cache replication

Partitioning answers:

> **How do I store more data?**

Replication answers:

> **How do I survive failures and handle more reads?**

For example:

``` text
        Primary
        /            v       v
 Replica 1  Replica 2
```

Replication introduces its own consistency and failover questions.

------------------------------------------------------------------------

## 29. Cache tiers

A system can have multiple caching layers:

``` text
Client
  |
  v
Browser cache
  |
  v
CDN
  |
  v
Application local cache
  |
  v
Distributed cache
  |
  v
Database
```

Each layer has different latency, capacity, scope, consistency, and
cost.

------------------------------------------------------------------------

## 30. Local vs distributed cache

### Local cache

``` text
Application instance
       |
       v
    RAM cache
```

Advantages:

-   extremely low latency
-   no network hop

Problems:

-   duplicated memory
-   inconsistent copies across instances
-   cache disappears with the instance

### Distributed cache

``` text
Application
     |
     v
Redis cluster
```

Advantages:

-   shared cache
-   larger capacity
-   centralized management

Problems:

-   network hop
-   additional infrastructure
-   distributed failure modes

------------------------------------------------------------------------

## 31. Cache stampede vs penetration vs avalanche

### Cache stampede

Many requests simultaneously regenerate the same expired item.

### Cache penetration

Requests repeatedly ask for data that does not exist.

Example:

``` text
Request
   |
   v
Cache miss
   |
   v
DB miss
```

Mitigations include:

-   negative caching
-   Bloom filters
-   request validation

### Cache avalanche

Many cache entries expire around the same time and cause a sudden surge
of DB traffic.

Mitigations include:

-   TTL jitter
-   staggered expiration
-   refresh-ahead
-   background warming

------------------------------------------------------------------------

## 32. Negative caching

You can cache the fact that something does not exist.

``` text
user:99999 -> NOT_FOUND
TTL = 30 seconds
```

Repeated requests then avoid hitting the DB.

This is particularly useful for preventing cache penetration.

------------------------------------------------------------------------

## 33. Cache warming

After a restart, a cache may be empty.

``` text
Application starts
      |
      v
Empty cache
      |
      v
Huge number of misses
      |
      v
DB traffic spike
```

Cache warming proactively loads important data before or during traffic
ramp-up.

------------------------------------------------------------------------

## 34. Serialization

Cached objects must be serialized:

``` text
Object
  |
  v
JSON / Protobuf / MessagePack / etc.
  |
  v
bytes
  |
  v
cache
```

Serialization affects:

-   CPU
-   memory
-   network bandwidth
-   latency
-   compatibility

At very high throughput, serialization can become a significant part of
request cost.

------------------------------------------------------------------------

## 35. Cache security

Caching can introduce security vulnerabilities.

Be careful with:

-   private responses
-   user-specific data
-   authentication information
-   authorization-sensitive results
-   multi-tenant data

A key such as:

``` text
profile
```

could be dangerous if different users share it.

Prefer:

``` text
profile:user:123
```

CDN and HTTP caches require careful handling of authentication, cookies,
authorization, and cache-control semantics.

------------------------------------------------------------------------

## 36. Observability

At minimum monitor:

### Hit rate

``` text
hits / total requests
```

### Miss rate

``` text
misses / total requests
```

### Eviction rate

How frequently entries are removed because of capacity pressure.

### Memory utilization

How close the cache is to capacity.

### Latency

Measure p50, p95, and p99.

### Backend load

The most important operational question is:

> **Is the cache actually reducing the expensive underlying workload?**

------------------------------------------------------------------------

## 37. Cache vs database

Do not memorize:

``` text
Database = slow
Cache = fast
```

A better model is:

                    Database            Cache
  ----------------- ------------------- -------------------
  Primary role      Own data            Accelerate access
  Durability        Usually essential   Often optional
  Query model       Rich                Usually limited
  Transactions      Often important     Usually limited
  Storage           Large               Relatively small
  Cost/GB           Lower               Higher
  Latency           Higher              Lower
  Dataset           Broad               Selective
  Reconstructible   No                  Ideally yes

There are important exceptions: some in-memory systems are full-fledged
primary databases.

------------------------------------------------------------------------

## 38. How to approach caching in a system-design interview

Don't start by saying:

> "I'll add Redis."

Instead reason through:

### Step 1 --- What is expensive?

``` text
DB query?
Computation?
External API?
Rendering?
```

### Step 2 --- Is the result reusable?

``` text
Same inputs -> same result?
```

### Step 3 --- What's the access pattern?

``` text
How frequent?
How concentrated?
How large is the working set?
```

### Step 4 --- What's the freshness requirement?

``` text
Strong?
Seconds?
Minutes?
Eventually consistent?
```

### Step 5 --- Choose the strategy

``` text
Cache-aside?
Read-through?
Write-through?
Write-back?
TTL?
Invalidation?
Refresh-ahead?
```

### Step 6 --- Design failure behavior

``` text
What happens if cache dies?
What happens on a cache-miss storm?
What happens if invalidation fails?
```

### Step 7 --- Scale it

``` text
How much memory?
How many nodes?
Partitioning?
Replication?
Hot keys?
```

------------------------------------------------------------------------

## 39. The one-minute interview explanation

If asked, "Why do we use caching?":

> **Caching lets us avoid repeatedly performing expensive or high-volume
> work by keeping a faster copy of the result. The cache is usually more
> expensive per unit of storage, so we don't cache everything---we cache
> data with strong locality or expensive access patterns. The key
> trade-off is that we've introduced a second copy, so we need to reason
> about freshness, invalidation, eviction, and failure. At scale, the
> cache itself becomes a distributed system, so we also need to consider
> partitioning, replication, hot keys, stampedes, and capacity.**

------------------------------------------------------------------------

## 40. Final mental model

``` text
                 SOURCE OF TRUTH
                       |
                       v
                      DB
                       |
              expensive / slow work
                       |
                       v
                    CACHE
                       |
                cheap / fast hit
                       |
                       v
                    CLIENT

              But now we have:
                       |
          +------------+------------+
          |            |            |
          v            v            v
      Freshness     Capacity      Failure
          |            |            |
      TTL /         eviction    stampede
   invalidation     / sizing    hot keys
                                   |
                                   v
                            distributed cache
                                   |
                              partitioning
                              replication
```

The fundamental question behind every caching decision is:

> **"What work am I avoiding, how often can I avoid it, and what am I
> willing to sacrifice to avoid it?"**

If you can reason through that question, the individual caching patterns
become consequences rather than things you have to memorize.
