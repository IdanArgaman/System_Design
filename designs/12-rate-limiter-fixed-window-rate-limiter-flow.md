# Fixed Window Rate Limiter — Summary

## What it does

`FixedWindowLimiter` limits how many units a caller can consume during a fixed time window.

Example:

```ts
new FixedWindowLimiter(5, 60_000)
```

means **5 units per 60 seconds per key**.

## Internal state

The limiter stores a counter for each key:

```ts
Map<string, {
  windowStart: number;
  count: number;
}>
```

Example:

```text
"user:123" → { windowStart: ..., count: 3 }
```

Each key has its own independent counter.

## How `tryAcquire()` works

### 1. Get the current time

```ts
const now = this.now();
```

`Date.now` is used by default, but a custom clock can be injected for testing.

### 2. Find the current window

```ts
const windowStart = now - (now % this.windowMs);
```

This aligns time to fixed boundaries.

For a 60-second window:

```text
12:34:00 ───────────────── 12:35:00
          current window
```

All requests during that minute share the same counter.

### 3. Get or reset the key's state

```ts
let s = this.state.get(key);

if (!s || s.windowStart !== windowStart) {
  s = { windowStart, count: 0 };
  this.state.set(key, s);
}
```

If the key is new or the previous window has expired, its counter resets to `0`.

### 4. Calculate when the window resets

```ts
const resetIn = windowStart + this.windowMs - now;
```

This tells the caller how many milliseconds remain before a new window starts.

### 5. Check the limit

```ts
if (s.count + cost > this.limit)
```

The request is rejected if adding `cost` would exceed the limit.

Rejected result:

```ts
{
  allowed: false,
  remaining: this.limit - s.count,
  retryAfterMs: resetIn
}
```

### 6. Accept and count the request

If the request fits within the limit:

```ts
s.count += cost;
```

Result:

```ts
{
  allowed: true,
  remaining: this.limit - s.count,
  retryAfterMs: 0
}
```

## Example

With:

```ts
new FixedWindowLimiter(3, 60_000)
```

The first three requests succeed:

```text
Request 1 → allowed, remaining: 2
Request 2 → allowed, remaining: 1
Request 3 → allowed, remaining: 0
Request 4 → rejected
```

After the window expires, the counter resets:

```text
12:34:00 → 3 requests allowed
12:35:00 → counter resets to 0
```

## `cost`

The optional `cost` parameter allows weighted requests.

```ts
tryAcquire("user:123", 1)  // costs 1 unit
tryAcquire("user:123", 10) // costs 10 units
```

So the limiter can represent a quota of units, not just individual requests.

## Important limitation: boundary burst

Fixed windows can allow bursts around a boundary.

For a limit of 5 requests/minute:

```text
12:34:59 → 5 requests
12:35:01 → 5 requests
```

That's 10 requests within about 2 seconds, because the requests belong to two different windows.

This is the main weakness of a fixed-window limiter.

## Important production consideration

The state is stored in a local JavaScript `Map`.

Therefore, the limiter only works independently inside one Node.js process.

If you run 5 application instances, each instance has its own limit.

For distributed rate limiting, shared storage such as Redis is typically used.

## Algorithm at a glance

```text
tryAcquire(key, cost)
        │
        ▼
   Get current time
        │
        ▼
 Calculate windowStart
        │
        ▼
 Get state for key
        │
        ├── new/expired ──→ reset count to 0
        │
        ▼
 Calculate reset time
        │
        ▼
 count + cost > limit?
      /       \
    YES        NO
     │          │
     ▼          ▼
  Reject     count += cost
     │          │
     ▼          ▼
 return       return
```

## In one sentence

A fixed-window rate limiter keeps **one counter per key for each aligned time window**, accepts a request when adding its `cost` stays within the limit, and rejects it otherwise until the next window.
