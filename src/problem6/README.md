
# Problem 6 - Scoreboard System Specification

## A. Functional Requirements
- Display the top 10 users’ scores
- Support real-time updates for all connected users
- Allow users to increment their score by 1
- Validate user authentication and authorization
- Prevent unauthorized score modifications
- Ensure score data is durable and recoverable

## B. Non-Functional Requirements
- Support high request rates and frequent reads
- Optimize for low-latency, read-heavy workloads
- Scale horizontally across all services
- Ensure eventual consistency between systems
- Provide high availability and fault tolerance
- Enforce rate limiting to prevent abuse and DDoS

## C. System Architecture

### C.1 Write Flow
 - Client sends a score increment request to the API
 - API validates authentication and rate limits the request
 - Score is updated in Redis synchronously
 - An event is published to a queue for asynchronous persistence
 - A background worker consumes events and writes to PostgreSQL

### C.2 Read Flow
 - Clients request leaderboard data from the API
 - API reads directly from Redis
 - The top 10 users are returned with minimal latency

### C.3 Real-Time Updates
 - When a score changes, an event is published
 - WebSocket servers broadcast updated leaderboard data to connected clients
 - Clients update their UI in real time

## D. Step-by-Step Implementation Guide

### D.1 Frontend
 - Displays the top 10 leaderboard
 - Connects to backend using WebSocket (preferred) or polling
 - Sends score increment requests
 - Applies client-side rate limiting (1 request per 10 seconds)
 - Updates UI in real time when receiving new data

### D.2 Backend API (Stateless)
 - Receive score update requests
 - Authenticate and authorize users
 - Enforce server-side rate limiting (1 update per user every 10 seconds)
 - Update score in Redis using atomic operations
 - Publish events for asynchronous persistence
 - Broadcast leaderboard updates via a pub/sub mechanism, then broadcast with WebSocket

**Key Design Choices:**
 - No locking is required; Redis operations are atomic
 - Stateless design allows horizontal scaling
 - API does not directly depend on database performance

## E. Data Storing

### E.1 Redis (Primary Data Store)
 - Stores leaderboard using sorted sets (ZSET)
 - Acts as the real-time source of truth
 - Strategy: Write-Behind (Asynchronous Persistence)
 - Key format: leaderboard:{category}
 - Operations: Increment score (ZINCRBY), Fetch top 10 (ZREVRANGE)

### E.2 PostgreSQL (Persistence Layer)
 - Stores durable user score data
 - Used for recovery, analytics, and long-term storage
 - Updated asynchronously via background workers

### E.3 Queue / Worker
 - Queue receives score update events
 - Worker processes events and writes to PostgreSQL
 - Supports batching and retry mechanisms

#### Pros
 - Redis is always up-to-date due to write-first design
 - Cache misses only occur during cold start or failure recovery

#### Cons
 - Redis can only be scale vertical, else we will need a strategy to sync between redis instance if scale
 - PostgreSQL is updated asynchronously, need a strategy to prevent data loss

## F. Security
 - Authenticate all API requests
 - Authorize score updates for valid users only
 - Rate limiting is enforced on both client and server
 - Input validation is applied to all incoming requests

## G. Infrastructure

### G.1 API Layer
 - Stateless services behind a load balancer
 - Scales horizontally based on CPU and request rate

### G.2 Redis
 - Managed service (cluster mode recommended)
 - Scales based on memory usage and throughput

### G.3 PostgreSQL
 - Managed database service
 - Scales vertically and with read replicas
 - Tuned based on IOPS and CPU usage

### G.4 CDN (Optional)
 - Used for serving static frontend assets
 - Can cache leaderboard responses with short TTL if slight delay is acceptable

---

## H. Scaling Strategy (AWS preferred)
 - API servers scale horizontally with traffic
 - Redis scales via sharding when memory or throughput limits are reached
 - PostgreSQL scales via increased IOPS and read replicas
 - WebSocket layer scales using pub/sub distribution
 - Use SNS Fan-out to public the topic after redis update
    For update to Postgres, can use lambda as worker
    For websocket broadcast, can use event bridge event to trigger the websocket server

---

## I. Failure Handling

### I.1 Redis Failure
 - Rebuild leaderboard from PostgreSQL snapshot or event logs

### I.2 Queue Failure
 - Use durable messaging with retry support

### I.3 Database Failure
 - Retry writes through worker
 - Buffer events temporarily until recovery

---

## J. Future Improvements
 - Multi-region deployment for global users
 - Sharding leaderboard by category
 - Time-based leaderboards (daily, weekly resets)
 - Snapshot and replay mechanisms
 - Load test (k6) and security test (burp suite)