# System Performance Interview Notes

## 1. Latency Percentiles

### What is latency?

**Latency** is the amount of time it takes a request to complete.

For an API, it can be measured from the moment the server receives a request until it sends the response.

Example:

```text
Request starts → 120ms → Response received
```

So the request latency is **120ms**.

When analyzing a production system, looking only at the average latency is often misleading. Instead, we use **percentiles** to understand how different groups of requests perform.

---

## 2. Why Use Percentiles?

Imagine an API processes 100 requests.

If 99 requests take 100ms but one request takes 10 seconds, the average can still look reasonably good even though one user had a terrible experience.

Percentiles let us understand the **distribution of latency**, especially the slow "tail" of requests.

The most commonly used percentiles are:

- **p50** — median latency
- **p95** — 95th percentile latency
- **p99** — 99th percentile latency

They are especially useful for detecting **tail latency**: requests that are significantly slower than normal.

---

## 3. p50 Latency

**p50 is the median latency.**

It means approximately:

> 50% of requests are completed in this amount of time or less, and 50% take longer.

Example:

```text
p50 = 100ms
```

This means the typical/median request takes around **100ms**.

p50 is useful for understanding the experience of the middle of the request population.

However, p50 alone can hide serious problems affecting slower requests.

---

## 4. p95 Latency

**p95 is the 95th percentile latency.**

It means approximately:

> 95% of requests complete in this amount of time or less, while the slowest 5% take longer.

Example:

```text
p95 = 500ms
```

This means 95% of requests finish within 500ms, but the slowest 5% take more than 500ms.

p95 is useful because it exposes problems that affect a meaningful minority of requests.

---

## 5. p99 Latency

**p99 is the 99th percentile latency.**

It means approximately:

> 99% of requests complete in this amount of time or less, while the slowest 1% take longer.

Example:

```text
p99 = 2 seconds
```

This means 99% of requests complete within 2 seconds, but the slowest 1% take longer than 2 seconds.

For a high-traffic system, 1% can still represent a large number of requests.

For example:

```text
1,000,000 requests/day
1% = 10,000 requests
```

So a p99 problem can have a significant real-world impact.

---

## 6. Example: Understanding p50, p95 and p99 Together

Suppose we have:

```text
p50 = 100ms
p95 = 500ms
p99 = 3s
```

I would interpret this as:

- The median request takes about 100ms.
- 95% of requests complete within 500ms.
- The slowest 5% take more than 500ms.
- The slowest 1% take more than 3 seconds.

This tells us that the system has a **long-tail latency problem**.

I would investigate what makes those slow requests different.

Potential causes include:

- Slow database queries
- Database connection-pool contention
- External API latency
- Cache misses
- CPU saturation
- Garbage collection
- Event-loop blocking
- Lock contention
- Network problems
- Large requests/responses
- Specific endpoints or workloads

---

# 7. Interview Question

## "The system suddenly gets slow. How would you check what's causing it?"

A strong approach is to avoid immediately assuming that the database, server, or application is the problem.

Instead, I would systematically identify:

1. **What became slow?**
2. **How much slower is it?**
3. **Where is the latency being introduced?**
4. **What changed around the time the problem started?**

My goal would be:

```text
Symptom
   ↓
Measure
   ↓
Locate bottleneck
   ↓
Form hypothesis
   ↓
Verify with data
   ↓
Mitigate
   ↓
Fix root cause
   ↓
Prevent recurrence
```

---

# 8. Step 1 — Define What "Slow" Means

First I would quantify the problem.

I would look at:

- Request rate / throughput
- p50 latency
- p95 latency
- p99 latency
- Error rate
- Timeout rate
- Number of concurrent requests

For example:

```text
Before:
p50 = 80ms
p95 = 150ms
p99 = 300ms

After:
p50 = 100ms
p95 = 2s
p99 = 8s
```

This tells me that the problem may primarily affect the tail of requests.

I would also ask:

- Are all endpoints slow?
- Is one endpoint slow?
- Are all users affected?
- Are only some users affected?
- Is the issue constant or intermittent?
- Did the error rate increase too?

---

# 9. Step 2 — Check What Changed

Because the system was previously healthy and suddenly became slow, I would investigate recent changes.

I would check:

- Recent deployments
- Configuration changes
- Database schema changes
- New database indexes or removed indexes
- Database migrations
- Feature flags
- New background jobs
- Infrastructure changes
- Traffic increases
- Cache changes
- External dependency changes

If the problem started immediately after a deployment, I would compare the new version against the previous version.

If there is strong evidence that the deployment caused the incident, I would consider rolling it back to mitigate the impact.

---

# 10. Step 3 — Identify Which Layer Is Slow

I would break the system into layers:

```text
Client
   |
Load Balancer
   |
API / Application
   |
   +---- Database
   |
   +---- Cache
   |
   +---- External APIs
   |
   +---- Message Queues
```

I want to determine where the request spends its time.

For example:

```text
Total request latency = 5 seconds

Load balancer:       20ms
Node.js application: 100ms
Database:            4.5s
External API:        200ms
```

The database is clearly suspicious.

But if:

```text
Total request latency = 5 seconds

Load balancer:       3 seconds
Node.js application: 100ms
Database:            50ms
External API:        100ms
```

then I would investigate the load balancer/network layer instead.

The key principle is:

> **Don't guess the bottleneck. Measure where the time is being spent.**

---

# 11. Step 4 — Check Infrastructure Resources

On application servers I would check:

- CPU utilization
- Memory utilization
- Disk I/O
- Disk space
- Network I/O
- Load average
- Number of processes
- Open file descriptors
- Connection counts

On Linux, useful commands include:

```bash
top
htop
vmstat
iostat
free -m
df -h
ss
```

For example:

```text
CPU:              95%
Memory:           90%
Disk I/O:         100%
DB connections:   100/100
```

Any of these could explain increased latency.

I would also compare current metrics against normal baseline values.

---

# 12. Step 5 — Check the Node.js Application

For a Node.js backend, I would pay particular attention to the **event loop**.

Node.js can handle many concurrent I/O operations efficiently, but CPU-heavy synchronous code can block the event loop.

For example:

```javascript
app.get('/data', (req, res) => {
    const result = expensiveCalculation();
    res.json(result);
});
```

If `expensiveCalculation()` takes several seconds, other requests handled by that process may be delayed.

I would investigate:

- Event-loop lag
- CPU utilization
- Synchronous/blocking operations
- CPU-heavy algorithms
- Large JSON serialization/deserialization
- Garbage collection
- Memory pressure
- Excessive concurrency

I would use application profiling/APM tools to identify expensive functions and slow request paths.

---

# 13. Step 6 — Check the Database

The database is one of the most common sources of application latency.

I would check:

- Query latency
- Slow queries
- Database CPU
- Memory
- Disk I/O
- Active connections
- Connection-pool utilization
- Lock contention
- Deadlocks
- Long-running transactions
- Replication lag
- Missing indexes
- Query execution plans

For PostgreSQL, for example:

```sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE user_id = 123;
```

I would look for things such as a sequential scan when an index should be used.

For example, suppose the table contains millions of rows:

```sql
SELECT *
FROM users
WHERE email = 'user@example.com';
```

If `email` is not indexed, the database may have to scan a very large number of rows.

The execution plan might show:

```text
Seq Scan
10,000,000 rows examined
```

After creating an appropriate index, the plan could potentially become:

```text
Index Scan
1 row examined
```

I would verify the actual improvement using the execution plan and real metrics rather than assuming an index automatically fixes the problem.

---

# 14. Step 7 — Check Database Connection Pools

Connection-pool exhaustion is an important case because the database query itself may not actually be slow.

For example:

```text
DB pool size = 20
Concurrent requests = 500
Active DB connections = 20/20
```

The remaining requests may wait for a connection.

The result could be:

```text
Request latency:             5 seconds
DB query execution:          50ms
Waiting for DB connection:   4.8 seconds
```

So I would monitor:

```text
Pool size
Active connections
Idle connections
Waiting requests
Connection acquisition latency
```

This distinction is important:

> The database query may be fast, but obtaining a database connection may be slow.

---

# 15. Step 8 — Check External Dependencies

An application can be slow because a service it depends on became slow.

For example:

```text
API
 |
 +-- Database:       50ms
 |
 +-- Redis:          10ms
 |
 +-- Payment API:    4 seconds
```

The payment API is the bottleneck.

I would investigate:

- External API latency
- Timeout rate
- Connection failures
- DNS resolution
- Network errors
- Retry counts

Retries deserve special attention.

Suppose an external service normally responds in 100ms but starts taking 2 seconds.

If the application retries twice:

```text
Initial request: 2s
Retry 1:         2s
Retry 2:         2s

Total ≈ 6s
```

Retries can also amplify traffic and create a cascading failure.

---

# 16. Step 9 — Check the Cache

If the system uses Redis or another caching layer, I would check:

- Cache hit rate
- Cache miss rate
- Cache latency
- Memory usage
- Evictions
- Connection saturation

For example:

```text
Normal:
Cache hit rate = 95%
DB traffic     = 1,000 queries/sec

Incident:
Cache hit rate = 20%
DB traffic     = 10,000 queries/sec
```

The cache may have been flushed, expired, or become unavailable.

That can cause a huge increase in database traffic and make the entire system slow.

---

# 17. Step 10 — Check Traffic

Sometimes nothing changed in the application—the traffic changed.

I would compare:

```text
Requests/sec
Concurrent users
Endpoint distribution
Geographic distribution
Request size
```

For example:

```text
Normal:   1,000 requests/sec
Current: 10,000 requests/sec
```

Then I would check:

- Did auto-scaling work?
- Is traffic evenly distributed?
- Is one server overloaded?
- Is rate limiting working?
- Can the database handle the additional load?
- Can the cache handle it?

I would also look for one endpoint suddenly receiving a disproportionate amount of traffic.

---

# 18. Step 11 — Use Distributed Tracing

For a distributed system, distributed tracing is extremely useful.

A trace might show:

```text
HTTP Request
   |
   +-- Service A: 100ms
          |
          +-- Service B: 200ms
          |
          +-- Database: 3,000ms
          |
          +-- Redis: 20ms
```

This immediately shows that the database call consumed most of the request time.

Tracing helps answer:

> "Where exactly did this request spend its time?"

This is much more useful than looking at application logs alone.

---

# 19. Step 12 — Check Logs and Request IDs

I would correlate logs using a request ID or trace ID.

For example:

```text
Request ID: abc123

09:00:00.000 request received
09:00:00.010 cache lookup
09:00:00.020 cache miss
09:00:00.025 DB query started
09:00:04.900 DB query finished
09:00:04.910 response returned
```

This tells me that most of the latency came from the database operation.

Structured logs are especially useful when querying by:

- Request ID
- Endpoint
- Status code
- Service
- Latency
- Error type

---

# 20. Step 13 — Check Memory and Garbage Collection

If the system becomes progressively slower, rather than suddenly slow, I would investigate memory.

For example:

```text
10:00 → 40% memory
11:00 → 55%
12:00 → 70%
13:00 → 85%
14:00 → 95%
```

Potential causes include:

- Memory leaks
- Unbounded caches
- Large objects
- Growing arrays/maps
- Unreleased resources
- Excessive concurrency

In Node.js, memory pressure can cause more garbage collection activity, increasing latency.

I would use heap snapshots and profiling to investigate.

---

# 21. Step 14 — Check Queues and Worker Pools

Sometimes the system is slow because requests are waiting in a queue.

For example:

```text
Incoming work
      |
      v
    Queue
      |
      v
 Worker pool
```

Suppose workers process:

```text
100 jobs/sec
```

but traffic produces:

```text
200 jobs/sec
```

The queue will continuously grow.

I would check:

- Queue depth
- Processing rate
- Worker count
- Worker utilization
- Message age
- Retry counts
- Dead-letter queues

---

# 22. Step 15 — Check for Cascading Failures

In a microservice architecture, one slow component can make many other components slow.

For example:

```text
Database becomes slow
        ↓
Service C waits
        ↓
Service B waits for Service C
        ↓
Service A waits for Service B
        ↓
Requests accumulate
        ↓
Connection pools become exhausted
        ↓
CPU/memory increase
        ↓
Entire system becomes slow
```

I would therefore investigate:

- Timeouts
- Retries
- Circuit breakers
- Connection pools
- Queue sizes
- Bulkheads
- Rate limits

The goal is to prevent one unhealthy dependency from taking down the whole system.

---

# 23. Step 16 — Build a Timeline

I would correlate all observations into a timeline.

For example:

```text
12:00  Deployment
12:05  CPU starts increasing
12:07  DB query latency increases
12:10  API p95 latency increases
12:12  Error rate increases
```

This gives me a strong hypothesis:

> A deployment introduced a database query that caused expensive database scans, which saturated the database and eventually exhausted the application's DB connection pool.

The timeline helps distinguish correlation from random symptoms and gives us a concrete direction to investigate.

---

# 24. Production Incident: Mitigate First, Then Investigate

If production is actively degraded, I would prioritize **mitigation** before performing risky investigation.

A possible sequence:

1. Confirm the incident.
2. Determine the blast radius.
3. Check recent deployments.
4. Roll back a suspicious deployment if appropriate.
5. Scale the affected service if that will help.
6. Disable an expensive feature using a feature flag if possible.
7. Protect dependencies using rate limits/timeouts/circuit breakers.
8. Continue root-cause analysis after the system is stable.

I would avoid making random changes in production because they can make the incident worse.

---

# 25. Complete Example

Suppose users report:

> "The API suddenly takes 5–10 seconds instead of 200ms."

I would investigate as follows.

### Step 1 — Check API metrics

```text
p95: 200ms → 7s
p99: 500ms → 12s
```

There is clearly a latency problem.

### Step 2 — Check traffic

```text
1,000 req/sec → 1,100 req/sec
```

Traffic only increased slightly, so traffic volume probably isn't the main cause.

### Step 3 — Check application servers

```text
CPU:    40%
Memory: 55%
```

Application servers look healthy.

### Step 4 — Check database

```text
CPU:              95%
Active connections: 500/500
Query latency:     significantly increased
```

The database looks suspicious.

### Step 5 — Find slow queries

I discover a query introduced by the latest deployment.

### Step 6 — Inspect the execution plan

`EXPLAIN ANALYZE` shows that the query performs a sequential scan over a 20-million-row table.

### Step 7 — Correlate with deployment

The query was introduced approximately 15 minutes before the incident.

### Conclusion

The application became slow because a newly deployed query caused expensive database scans.

Those scans saturated the database, which increased query latency and eventually exhausted the application's database connection pool.

### Immediate mitigation

```text
Rollback deployment
```

### Permanent fix

- Optimize the query.
- Add or adjust the appropriate index.
- Verify using `EXPLAIN ANALYZE`.
- Monitor query latency.
- Add performance/regression tests.
- Deploy again after verification.

---

# 26. My Interview Mental Model

When I hear:

> "The system suddenly became slow."

My mental model is:

```text
                    System is slow
                         |
                         v
                 Define the symptom
                         |
                         v
                 Check p50/p95/p99
                         |
                         v
                Check error/timeout rate
                         |
                         v
                  What changed?
                         |
                         v
                Identify the layer
                         |
          +--------------+--------------+
          |              |              |
        App             DB          Network
          |              |              |
        CPU           Queries         APIs
        Memory        Locks           DNS
        Event loop    Pool            Retries
        GC            I/O             Timeouts
          |              |              |
          +--------------+--------------+
                         |
                         v
                  Form a hypothesis
                         |
                         v
                  Verify with data
                         |
                         v
                    Mitigate
                         |
                         v
                 Fix root cause
                         |
                         v
              Prevent recurrence
```

The key principle I would communicate in the interview is:

> **"I don't want to guess which component is slow. I want to measure where the latency is being introduced, form a hypothesis based on the data, and then verify that hypothesis."**

That demonstrates an understanding of observability, performance debugging, distributed systems, databases, Node.js, and production incident response.
