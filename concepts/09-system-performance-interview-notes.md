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

### How to Fix It

- Move CPU-heavy work off the main event loop — for example into worker threads, a background job, or a separate service.
- Break large synchronous operations into smaller chunks so the event loop gets a chance to breathe between them.
- Cache expensive results instead of recalculating them on every request.
- Avoid building huge objects in memory (e.g. stream large responses instead of assembling them all at once).
- Add more application instances so blocking work in one process doesn't stall all traffic.
- Re-check event-loop lag and CPU metrics after the change to confirm it actually helped.

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

### How to Fix It

- Add the missing index (or fix a poorly matched one) and confirm the improvement with `EXPLAIN ANALYZE`.
- Rewrite inefficient queries — avoid `SELECT *`, filter earlier, and add pagination instead of pulling huge result sets.
- Add caching in front of expensive or frequently repeated queries.
- Break up long-running transactions so they hold locks and resources for less time.
- Scale the database if the load is genuinely more than it can handle — e.g. a read replica for read-heavy traffic, or a bigger instance.
- Set up ongoing slow-query monitoring so regressions like this are caught before they become incidents.

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

### How to Fix It

- Fix the underlying slow queries or long-running transactions first — the fewer connections take a long time, the fewer get tied up at once.
- Increase the pool size only after confirming the database itself can handle more concurrent connections; making the pool bigger than the database can support just moves the bottleneck.
- Introduce a connection pooler (e.g. PgBouncer for PostgreSQL) so many application connections can share a smaller number of real database connections.
- Add read replicas so read traffic isn't competing with writes for the same pool.
- Set a reasonable connection-acquisition timeout so requests fail fast with a clear error instead of piling up silently.
- Add alerting on pool utilization so it's flagged well before it hits 100%.

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

### How to Fix It

- Set a sensible timeout on every external call so a slow dependency can't hold up the whole request indefinitely.
- Add a circuit breaker that stops calling a dependency for a short period once it starts failing or timing out repeatedly, instead of hammering it with more requests.
- Use exponential backoff with jitter for retries, and cap the total number of attempts, so retries don't pile up and amplify the load on an already struggling service.
- Where possible, make the call to a non-critical dependency asynchronous or optional, so its failure degrades the response gracefully instead of failing the entire request.
- Cache responses from the dependency when appropriate, to reduce how often you need to call it live.
- Track the dependency's latency and error rate over time, and raise it with the provider if it's outside agreed expectations.

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

### How to Fix It

- Find out why the cache emptied — a restart, an eviction due to memory limits, a TTL misconfiguration, or an outage — and address that root cause.
- Warm the cache proactively after a deploy or restart instead of letting it fill up cold under live traffic.
- Increase cache memory or tune the eviction policy if entries are being evicted too aggressively under normal load.
- Add request coalescing (so many simultaneous misses for the same key trigger only one database query) to avoid overwhelming the database when the cache is cold.
- Add alerting on cache hit rate so a drop is caught within minutes rather than discovered through a database overload.

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

### How to Fix It

- Confirm autoscaling is actually triggering, and tune its thresholds so new capacity comes online quickly enough.
- Add or tighten rate limiting so a single client, IP, or endpoint can't consume a disproportionate share of capacity.
- Check that the load balancer is spreading traffic evenly and isn't sending too much to one instance.
- Scale the database and cache alongside the application layer — extra application servers don't help if the shared database is the ceiling.
- Add a queue in front of spiky, non-time-critical work so sudden bursts are absorbed instead of overwhelming the system directly.
- If the traffic is unexpected or abusive (e.g. a scraper or bot), block or throttle it at the edge.

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

### How to Fix It

- Find and fix the actual leak — commonly an unbounded cache, a growing array/map that's never cleaned up, or event listeners/connections that are never released.
- Put explicit limits on in-memory caches and collections (e.g. an LRU cache with a maximum size) instead of letting them grow forever.
- Make sure resources like file handles, sockets, and DB connections are always released, including on error paths.
- As a short-term stopgap while the root cause is fixed, restart affected processes on a schedule or before they reach a dangerous memory level.
- Add memory monitoring and alerting so a slow leak is caught well before it causes an incident.

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

### How to Fix It

- Add more workers (or scale them automatically) so processing capacity matches or exceeds the incoming rate.
- Speed up how long each job takes to process, since that has the same effect as adding workers.
- Add backpressure so producers slow down, reject, or shed low-priority work once the queue grows past a healthy size, rather than letting it grow unbounded.
- Prioritize critical jobs ahead of lower-priority ones when the queue is under pressure.
- Route jobs that repeatedly fail to a dead-letter queue so they don't block healthy jobs behind them.
- Alert on queue depth and message age, not just on whether the queue exists, so a backlog is caught early.

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

### How to Fix It

- Set timeouts on every call between services, so a service never waits indefinitely on a slow dependency.
- Add circuit breakers so that once a dependency starts failing or timing out, calls to it are paused for a short period instead of continuing to pile up.
- Use bulkheads — isolate resources (like connection pools or thread pools) per dependency, so a problem with one dependency can't consume the resources needed to talk to a healthy one.
- Apply rate limits between services so one overloaded caller can't overwhelm a downstream service and spread the problem further.
- Size and monitor connection pools per dependency, since pool exhaustion is often the mechanism by which a slowdown spreads from one service to the next.
- Use retries carefully, with backoff and a strict limit — unrestrained retries under load tend to make cascading failures worse, not better.

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
