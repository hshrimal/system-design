# Real-Time Architecture 101

Real-time architecture is about getting information from the moment an event happens to the consumers that care about it with sufficiently low delay.

The important word is **sufficiently**.

"Real-time" does not always mean milliseconds. A stock-trading system may require millisecond-level latency, while a food-delivery status update may be perfectly useful if it arrives within a few seconds.

The first principle is therefore:

> **Design for the required freshness and latency, not for the label "real-time."**

---

## 1. What makes a system real-time?

A traditional request/response system looks like:

```text
Client
  │
  │ request
  ▼
Server
  │
  │ response
  ▼
Client
```

The client asks for information when it needs it.

A real-time system often reverses the direction:

```text
Event happens
     │
     ▼
Backend detects/processes event
     │
     ▼
Interested clients receive update
```

For example, in a ride-hailing system:

```text
Driver moves
    │
    ▼
Location update
    │
    ▼
Backend
    │
    ▼
Rider watching the trip
```

The rider should not have to repeatedly ask:

> "Did the driver move?"

The system should deliver the update when it becomes available.

---

# 2. Real-time is primarily a latency + freshness problem

Two related concepts matter.

### Latency

How long does it take for an event to travel through the system?

```text
Event occurs ───────────────► Client receives event
             <--- latency --->
```

### Freshness

How old is the information when the client sees it?

A system can have low processing latency but still show stale information if the source itself updates infrequently.

For example:

```text
Driver location updated every 5 seconds
Backend processing latency = 50 ms
```

The backend is fast, but the location can still be almost 5 seconds old.

So real-time requirements should be expressed quantitatively:

- Maximum acceptable end-to-end latency
- Required update frequency
- Acceptable staleness
- Expected traffic
- Number of connected clients

---

# 3. Push vs. Pull

There are two fundamental ways to deliver updates.

## Pull

The client asks the server for updates.

```text
Client ──► Server
         "Anything new?"

Client ◄── Server
         "No"

...later...

Client ──► Server
         "Anything new?"

Client ◄── Server
         "Yes"
```

## Push

The server sends an update when something changes.

```text
Client ◄──────── Server
             "Something changed"
```

Push is generally more efficient when updates are unpredictable and the client needs them quickly.

---

# 4. Polling

The simplest pull-based approach is polling.

```text
Client ── request ──► Server
Client ◄─ response ── Server

wait

Client ── request ──► Server
Client ◄─ response ── Server
```

For example, poll every 5 seconds:

```text
t=0   request
t=5   request
t=10  request
t=15  request
```

### Advantages

- Very simple
- Uses normal HTTP
- Easy to load balance
- Easy to debug
- Works through most infrastructure

### Problems

If updates are rare:

```text
100 requests
  ↓
1 actual update
99 useless requests
```

If you poll frequently:

```text
100 clients × 10 requests/sec
= 1,000 requests/sec
```

Most of those requests may still return "nothing changed."

Polling creates a trade-off:

```text
Lower polling interval
       ↓
Lower latency
       ↓
Higher request volume
```

---

# 5. Long Polling

Long polling improves on regular polling.

Instead of immediately responding, the server keeps the request open until:

- an update is available, or
- a timeout occurs.

```text
Client ───────────────► Server

                        wait...

                        event occurs

Client ◄─────────────── Server
```

The client then immediately makes another request.

```text
Client ───────────────► Server
Client ◄─────────────── Server

Client ───────────────► Server
             ...
```

### Advantages

- Lower unnecessary traffic than polling
- Still based on HTTP
- Easier to introduce into existing systems

### Problems

- Long-lived HTTP requests
- More connection management
- Still fundamentally request/response
- Less efficient than purpose-built persistent connections at very large scale

---

# 6. Server-Sent Events (SSE)

SSE provides a persistent HTTP connection where the server can continuously send events to the client.

```text
Client ───────────────► Server
        connection

Client ◄─────────────── Server
        event

Client ◄─────────────── Server
        event

Client ◄─────────────── Server
        event
```

The communication is primarily:

```text
Server ─────────► Client
```

### Good use cases

- Notifications
- Live dashboards
- Progress updates
- Activity feeds
- Monitoring
- Streaming server-generated updates

### Important limitation

SSE is primarily one-way.

If the client also needs to send real-time messages, it generally uses a separate mechanism such as normal HTTP requests.

---

# 7. WebSockets

WebSockets provide a persistent, bidirectional connection.

```text
Client ═══════════════ Server
       bidirectional
```

Both sides can send messages at any time.

```text
Client ───────► Server
Client ◄─────── Server
Client ───────► Server
Client ◄─────── Server
```

### Good use cases

- Chat
- Multiplayer games
- Collaborative applications
- Real-time control
- Live location
- Interactive trading applications

### Key property

The important feature is not simply "WebSocket is faster."

The important feature is:

> **A persistent bidirectional communication channel.**

---

# 8. Polling vs Long Polling vs SSE vs WebSockets

| Approach | Direction | Persistent connection | Typical use |
|---|---|---|---|
| Polling | Client → Server | No | Simple periodic updates |
| Long polling | Mostly Server → Client | Temporarily | Legacy/simple real-time |
| SSE | Server → Client | Yes | Feeds, dashboards, notifications |
| WebSocket | Bidirectional | Yes | Chat, games, interactive systems |

The correct choice depends on requirements.

Do not start a system-design interview by saying:

> "We'll use WebSockets."

Start with:

> "What latency and communication pattern does the product require?"

---

# 9. WebSocket is not the whole architecture

A common beginner architecture is:

```text
Client
  │
  ▼
WebSocket Server
  │
  ▼
Client
```

This works for a small system.

But imagine:

```text
10 million connected users
100,000 events/sec
```

Now the difficult questions begin.

When an event occurs:

> Which clients need to receive it?

And:

> Which server currently owns those client connections?

This is where real-time architecture becomes a distributed-systems problem.

---

# 10. Connection management

A WebSocket connection is long-lived.

For example:

```text
User A ──► WS Server 1
User B ──► WS Server 1
User C ──► WS Server 2
User D ──► WS Server 3
```

Suppose an event for User C arrives at Server 1.

But User C's connection lives on Server 2.

Server 1 needs a way to communicate with Server 2.

This is why large-scale real-time systems typically need an internal event-distribution mechanism.

---

# 11. Pub/Sub

A common pattern is:

```text
                 ┌─────────────┐
                 │  Producer   │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │   Pub/Sub   │
                 │    / Bus    │
                 └──────┬──────┘
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
             WS1       WS2       WS3
```

The producer does not need to know which WebSocket server has which connection.

It publishes an event.

The appropriate real-time servers consume it and deliver it to their connected clients.

This creates an important separation:

```text
Event generation
       │
       ▼
Event distribution
       │
       ▼
Client delivery
```

---

# 12. Fan-out

Fan-out means taking one event and delivering it to multiple consumers.

Suppose:

```text
Driver 42 moves
```

There may be:

```text
Rider A
Rider B
Rider C
```

watching that driver.

The system needs:

```text
Driver 42 location update
           │
           ▼
        Fan-out
       /    |          A     B     C
```

Fan-out can become one of the biggest scaling challenges.

Consider:

```text
1 event
×
1,000,000 interested users
=
1,000,000 deliveries
```

The architecture must handle this efficiently.

---

# 13. Connection registry

The system often needs to know:

```text
User → active connection → server
```

For example:

```text
User 42
   │
   ▼
Connection abc123
   │
   ▼
WebSocket Server 17
```

A connection registry can maintain this mapping.

The registry may contain ephemeral information such as:

- Connection ID
- User ID
- Server ID
- Session information
- Last heartbeat
- Presence state

This information usually does not belong in a durable primary database.

---

# 14. Presence and ephemeral state

Presence answers questions such as:

```text
Is the user online?
Which device is connected?
Which server owns the connection?
When was the last heartbeat?
```

This state is often:

> **Ephemeral**

If a server crashes, the connection disappears.

The system should be able to reconstruct the state when the client reconnects.

This is different from durable business data.

For example:

```text
Message sent
    ↓
Durable data

User currently online
    ↓
Ephemeral state
```

---

# 15. Scaling connections

There are two distinct scaling dimensions:

### Request scaling

```text
requests/sec
```

### Connection scaling

```text
concurrent connections
```

A real-time system can have relatively low requests/sec but enormous numbers of persistent connections.

For example:

```text
1 million connections
10 messages/sec

= potentially 10 million messages/sec
```

Therefore, scaling a real-time system requires thinking about:

- Connection limits
- CPU
- Memory
- Network bandwidth
- Load balancing
- Connection distribution
- Connection lifecycle

---

# 16. Partitioning

Large real-time systems often partition traffic.

For example:

```text
Users 0–999,999
       ↓
Partition 1

Users 1,000,000–1,999,999
       ↓
Partition 2
```

Or partition by a domain entity:

```text
driver_id
   ↓
partition
```

Partitioning allows multiple servers to process different portions of the workload independently.

A good partition key should distribute load reasonably evenly while preserving any ordering requirements.

---

# 17. Ordering

Suppose a client receives:

```text
Location A
Location B
Location C
```

But due to distributed processing it receives:

```text
A
C
B
```

That may be problematic.

Therefore we need to decide:

> **What ordering guarantee does the system actually require?**

Possible guarantees include:

- No ordering guarantee
- Per-user ordering
- Per-entity ordering
- Global ordering

Global ordering is expensive and usually unnecessary.

A common design is to preserve ordering within a partition or entity.

---

# 18. Delivery semantics

Distributed systems can fail.

A message might:

- Never arrive
- Arrive once
- Arrive multiple times
- Arrive late
- Arrive out of order

Common delivery semantics are:

### At-most-once

```text
0 or 1 delivery
```

Fast/simple, but messages can be lost.

### At-least-once

```text
1 or more deliveries
```

Messages should not be lost, but duplicates are possible.

### Exactly-once

```text
Exactly one logical effect
```

This is difficult to achieve end-to-end.

In practice, many systems use:

```text
At-least-once delivery
+
Idempotent processing
```

---

# 19. Idempotency

Suppose a client receives:

```text
event_id = 123
```

and processes it.

Due to retry, it receives:

```text
event_id = 123
```

again.

The system should avoid producing an incorrect second effect.

A common pattern is to include a unique event ID:

```text
{
  "event_id": "123",
  "type": "location_update",
  "sequence": 57,
  ...
}
```

Consumers can use event IDs or sequence numbers to detect duplicates or stale events.

---

# 20. Backpressure

What happens if producers generate events faster than consumers can process them?

```text
Producer
  │
  │ 100,000 events/sec
  ▼
Consumer
  │
  │ can process 50,000/sec
  ▼
Backlog grows
```

Without controls, the system eventually runs out of resources.

Possible strategies include:

- Buffering
- Rate limiting
- Batching
- Dropping stale updates
- Load shedding
- Slowing producers
- Prioritizing important events

Real-time systems often have an important optimization:

> **For some data, the latest value matters more than every intermediate value.**

For example, if a driver moves:

```text
A → B → C → D → E
```

a rider may only need:

```text
E
```

rather than receiving every intermediate position.

This can dramatically reduce load.

---

# 21. Reconnection

Persistent connections fail.

Reasons include:

- Mobile network changes
- Wi-Fi → cellular transition
- Server failure
- Load balancer failure
- Network interruption
- Client backgrounding
- Deployment

Therefore:

```text
Connect
   ↓
Connected
   ↓
Disconnected
   ↓
Reconnect
```

must be a normal part of the architecture.

Clients typically use:

- Exponential backoff
- Jitter
- Heartbeats
- Connection timeouts

---

# 22. Failure handling

Consider this architecture:

```text
Client
  ↓
WS Server
  ↓
Pub/Sub
```

What happens if the WebSocket server crashes?

The client should reconnect to another server.

What if the connection drops after the server processed a message but before the client received it?

Now we need to decide whether the client can recover missed events.

This may require:

- Sequence numbers
- Event IDs
- Replay
- Durable event streams
- Client acknowledgements

---

# 23. Stateful vs stateless thinking

WebSocket servers are naturally stateful because they maintain live connections.

But we generally want the *business logic* to remain as stateless as possible.

A useful separation is:

```text
                 ┌─────────────────────┐
                 │ Stateless services  │
                 │ Business logic      │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Event infrastructure│
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Connection layer    │
                 │ Stateful connections│
                 └─────────────────────┘
```

This makes the system easier to scale and recover.

---

# 24. Generic scalable real-time architecture

Putting the pieces together:

```text
                         ┌───────────────┐
                         │   Producers   │
                         │ Web / Mobile  │
                         └───────┬───────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │ API / Ingest  │
                         └───────┬───────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │ Event Bus / Pub-Sub     │
                    └────────────┬───────────┘
                                 │
                ┌────────────────┼────────────────┐
                ▼                ▼                ▼
          Processing        Persistence      Other consumers
                │
                ▼
          Fan-out service
                │
        ┌───────┼────────┐
        ▼       ▼        ▼
      WS-1    WS-2     WS-3
        │       │        │
        ▼       ▼        ▼
      Clients / subscribers
```

The important architectural separation is:

```text
Ingest
  ↓
Process
  ↓
Distribute
  ↓
Deliver
```

---

# 25. The end-to-end latency budget

When someone says:

> "The system must update users within 1 second."

Break it down.

```text
Event generation
      ↓ 100 ms
Ingestion
      ↓ 100 ms
Processing
      ↓ 200 ms
Message distribution
      ↓ 100 ms
WebSocket delivery
      ↓ 100 ms
Client rendering
      ↓
Total ≈ 600 ms
```

This is much more useful than simply saying:

> "We need a low-latency system."

Real-time performance should be considered **end-to-end**.

---

# 26. Observability

A real-time system needs visibility into:

### Latency

```text
event_timestamp
        ↓
delivery_timestamp
        ↓
end-to-end latency
```

### Connection metrics

- Active connections
- Connection creation rate
- Disconnect rate
- Reconnection rate
- Connections per server

### Message metrics

- Events/sec
- Messages/sec
- Fan-out size
- Queue depth
- Consumer lag
- Dropped events
- Duplicate events

### Reliability

- Connection failures
- Delivery failures
- Retry rates
- Server failures
- Broker failures

---

# 27. A simple mental model

For system-design interviews, remember:

```text
             REAL-TIME SYSTEM
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   TRANSPORT     EVENTS       CONNECTIONS
       │            │            │
       ▼            ▼            ▼
 Polling         Pub/Sub      WebSocket
 Long polling    Broker       SSE
 SSE             Stream       Connection registry
 WebSocket       Queue        Presence
       │            │            │
       └────────────┼────────────┘
                    ▼
              DISTRIBUTED
                SYSTEM
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Ordering     Delivery      Failure
                semantics     handling
       │            │            │
       └────────────┼────────────┘
                    ▼
              SCALE + LATENCY
```

---

# 28. How to approach a real-time system-design question

When asked to design a real-time system, walk through these questions.

### 1. What needs to be real-time?

Identify the actual events.

### 2. What is the latency requirement?

Milliseconds? Seconds? Minutes?

### 3. Is communication one-way or bidirectional?

This helps determine SSE vs WebSockets.

### 4. How many clients are connected?

Think about concurrent connections, not just requests/sec.

### 5. How many events are generated?

Estimate event rate and bandwidth.

### 6. Who needs each event?

This determines fan-out strategy.

### 7. What ordering is required?

Global? Per user? Per entity? None?

### 8. Can events be lost?

Determine delivery semantics.

### 9. Can events be duplicated?

If yes, make processing idempotent.

### 10. What happens when connections fail?

Design reconnection and recovery.

### 11. What happens when consumers are slower than producers?

Design backpressure and load shedding.

### 12. What state must be durable?

Separate durable business state from ephemeral connection state.

---

# 29. The most important takeaway

Real-time architecture is **not synonymous with WebSockets**.

WebSockets solve one problem:

> **Maintaining a bidirectional communication channel between a client and server.**

A scalable real-time architecture must solve much more:

```text
                    Event
                      │
                      ▼
                  Ingestion
                      │
                      ▼
                 Distribution
                      │
                      ▼
                    Fan-out
                      │
                      ▼
               Connection layer
                      │
                      ▼
                    Client
```

And around that pipeline we need:

```text
Partitioning
Ordering
Delivery semantics
Idempotency
Backpressure
Reconnection
Failure handling
Observability
```

The core mental model is:

> **Real-time architecture = event propagation + efficient delivery + connection management + distributed-systems guarantees.**
