# gRPC and HTTP/2 — Technical Summary

## 1. Big picture

gRPC is an RPC framework that normally runs over HTTP/2.

The conceptual stack is:

```text
Application
    ↓
gRPC
    ↓
HTTP/2
    ↓
TCP
    ↓
IP
    ↓
Network
```

The key relationship is:

> **gRPC uses HTTP/2 features such as multiplexing, streams, binary framing, header compression, flow control, and long-lived connections.**

---

## 2. Multiplexing

Multiplexing means multiple logical communications can share one underlying connection.

Instead of creating a separate TCP connection for every RPC:

```text
RPC A → TCP connection A
RPC B → TCP connection B
RPC C → TCP connection C
```

HTTP/2 can put them on one TCP connection:

```text
                    One TCP connection
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
         Stream 1       Stream 3       Stream 5
          RPC A           RPC B           RPC C
```

This is one of the major reasons gRPC uses HTTP/2.

---

## 3. What is an HTTP/2 stream?

A stream is **not just the server response**.

It is a **logical, bidirectional communication channel inside an HTTP/2 connection**.

For a normal request/response:

```text
Client                         Server
  │                              │
  │──── request ────────────────>│
  │                              │
  │<──────── response ───────────│
  │                              │
```

Both directions belong to the same HTTP/2 stream.

For example:

```text
HTTP/2 Stream #7
├── Client → Server: request
└── Server → Client: response
```

Each stream has a stream ID.

A useful mental model is:

> **TCP connection = physical transport; HTTP/2 stream = virtual channel inside that connection.**

---

## 4. A stream can carry multiple messages

A stream does not necessarily mean one small message in each direction.

This becomes important with gRPC streaming.

For example:

```text
Client                         Server
  │                              │
  │── message 1 ────────────────>│
  │── message 2 ────────────────>│
  │── message 3 ────────────────>│
  │                              │
  │<──────── message A ──────────│
  │<──────── message B ──────────│
  │<──────── message C ──────────│
  │                              │
  │        stream closes         │
```

The HTTP/2 stream can remain open while multiple gRPC messages are exchanged.

---

# 5. gRPC RPC patterns

gRPC supports four common RPC patterns.

### Unary RPC

One request → one response:

```text
Client ── request ──> Server
Client <── response ── Server
```

### Server streaming

One request → multiple responses:

```text
Client ── request ──> Server

Client <── message 1 ── Server
Client <── message 2 ── Server
Client <── message 3 ── Server
```

### Client streaming

Multiple requests/messages → one response:

```text
Client ── message 1 ──> Server
Client ── message 2 ──> Server
Client ── message 3 ──> Server

Client <──── response ──── Server
```

### Bidirectional streaming

Both sides exchange multiple messages:

```text
Client ── message 1 ──> Server
Client <── message A ── Server
Client ── message 2 ──> Server
Client <── message B ── Server
Client ── message 3 ──> Server
Client <── message C ── Server
```

These messages can be carried by the same HTTP/2 stream for that RPC.

---

# 6. Long-lived gRPC connections

A gRPC client normally reuses its underlying HTTP/2/TCP connection for multiple RPCs.

For example:

```text
One TCP connection
│
├── gRPC RPC 1
├── gRPC RPC 2
├── gRPC RPC 3
├── ...
└── gRPC RPC 1000
```

Finishing an RPC normally does **not** close the TCP connection.

The connection may stay open for minutes, hours, or potentially indefinitely, depending on:

- Client/server configuration
- HTTP/2/gRPC keepalive settings
- Idle timeouts
- Load balancers and proxies
- Network failures
- Server restarts

The exact behavior depends on the gRPC implementation and infrastructure.

---

# 7. Why not create a gRPC client per request?

A common bad pattern is:

```text
Incoming request
    ↓
Create gRPC client
    ↓
Make RPC
    ↓
Discard client
```

That can cause unnecessary connections, handshakes, sockets, memory use, and connection-management overhead.

A better general pattern is to create a client/channel and reuse it:

```js
const client = new MyGrpcService(
  "server:50051",
  grpc.credentials.createInsecure()
);

// Reuse the client
client.getUser(...);
client.getUser(...);
client.getUser(...);
```

The exact connection/channel behavior depends on the gRPC library, resolver, load-balancing configuration, and transport implementation, but client reuse is the important principle.

---

# 8. The 64K connection misconception

The commonly mentioned `~65,535` value is generally related to TCP port/address space, especially ephemeral ports, and is **not a universal limit on the number of connections a Node.js process can have**.

More importantly, with HTTP/2/gRPC:

```text
Many RPCs
    ↓
Many HTTP/2 streams
    ↓
One or a small number of TCP connections
```

rather than:

```text
Many RPCs
    ↓
One TCP connection per RPC
```

So thousands of RPCs do not inherently require thousands of TCP connections.

However, an application can still create too many connections if it creates many independent clients/channels or if its connection/load-balancing configuration requires multiple connections.

---

# 9. HTTP/2 uses binary framing

HTTP/1.1 is primarily text-oriented:

```http
GET /users HTTP/1.1
Host: example.com
```

HTTP/2 uses a **binary framing layer**.

HTTP/2 communication is divided into frames.

Important frame types include:

- `HEADERS`
- `DATA`
- `SETTINGS`
- `WINDOW_UPDATE`
- `PING`
- `RST_STREAM`
- `GOAWAY`

Frames contain information such as the stream they belong to.

This framing is what allows HTTP/2 to interleave data belonging to different streams over the same connection.

Conceptually:

```text
TCP byte stream
       ↓
HTTP/2 frames
       ↓
┌─────────┬─────────┬─────────┬─────────┐
│ Stream A│ Stream B│ Stream C│ Stream A│
│ frame   │ frame   │ frame   │ frame   │
└─────────┴─────────┴─────────┴─────────┘
```

---

# 10. HTTP/2 stream IDs

Streams have IDs.

The client and server use different stream-ID number sequences.

A simplified example:

```text
Stream 1
Stream 3
Stream 5
Stream 7
...
```

The IDs allow HTTP/2 to associate each frame with its logical stream.

This is how frames for different requests can be interleaved while still being associated with the correct RPC/request.

---

# 11. HTTP/2 header compression — HPACK

HTTP headers can be repetitive.

For example, many requests may contain headers such as:

```text
content-type: application/grpc
authorization: Bearer ...
user-agent: ...
```

HTTP/2 uses **HPACK** to compress headers.

HPACK uses techniques including:

- A static table of commonly used headers
- A dynamic table of previously seen headers
- Indexed representations
- Huffman coding for string representations

The benefit is reduced header overhead when many requests share the same connection.

---

# 12. HTTP/2 flow control

HTTP/2 has flow control to prevent a fast sender from overwhelming a receiver.

There are two important levels:

```text
Connection-level flow control
            +
Stream-level flow control
```

This means HTTP/2 can control how much data can be sent:

- Across the entire connection
- For an individual stream

This is especially important for large responses and gRPC streaming.

The `WINDOW_UPDATE` frame is used to increase the available flow-control window.

---

# 13. HTTP/2 long-lived connections

HTTP/2 is designed around connection reuse.

Instead of repeatedly doing:

```text
TCP handshake
TLS handshake
HTTP request
HTTP response
TCP close
```

a connection can remain open:

```text
TCP connection
│
├── HTTP/2 Stream 1
├── HTTP/2 Stream 3
├── HTTP/2 Stream 5
├── HTTP/2 Stream 7
└── ...
```

This is particularly valuable for gRPC because services often make many RPCs between the same endpoints.

---

# 14. HTTP/2 server push

HTTP/2 originally included **server push**.

The idea was that the server could proactively send a resource before the client explicitly requested it.

This is not a major concept for modern gRPC development and is largely obsolete/deprecated in modern browser usage.

---

# 15. HTTP/2 stream prioritization

HTTP/2 was designed with mechanisms for expressing stream priority.

The general idea is that some streams/resources could be considered more important than others.

In practice, behavior and support vary, so stream prioritization is much less important to understand for gRPC than:

- Multiplexing
- Streams
- Flow control
- Binary framing
- HPACK

---

# 16. The subtle problem: HTTP/2 streams are separate, but TCP is not

This is one of the most important technical details.

HTTP/2 gives us:

```text
                 HTTP/2
        ┌──────────┼──────────┐
     Stream A   Stream B   Stream C
        └──────────┼──────────┘
                   │
                  TCP
                   │
                Network
```

The HTTP/2 layer knows A, B, and C are separate streams.

But TCP sees **one ordered byte stream**.

TCP guarantees in-order delivery.

Therefore, HTTP/2 multiplexing does not completely eliminate all forms of blocking.

---

# 17. TCP Head-of-Line blocking

Suppose HTTP/2 frames are carried by TCP packets like this:

```text
TCP packet 1 → Stream A
TCP packet 2 → Stream B
TCP packet 3 → Stream C
TCP packet 4 → Stream A
```

Now packet 2 is lost:

```text
packet 1 → received ✅
packet 2 → LOST ❌
packet 3 → received ✅
packet 4 → received ✅
```

HTTP/2 knows packet/frame 3 may belong to Stream C.

But TCP cannot simply expose the later bytes to HTTP/2 while byte 2 is missing.

TCP must preserve the ordered byte stream.

Therefore:

```text
packet 2
   ↓
retransmission
   ↓
received
   ↓
TCP delivers the ordered bytes
   ↓
HTTP/2 processes the frames
```

This is **TCP-level Head-of-Line (HOL) blocking**.

---

# 18. Important distinction about multiplexing

HTTP/2 multiplexing gives us independent **logical streams**, but they still share the same TCP transport.

Therefore:

> **HTTP/2 multiplexing does not remove TCP-level Head-of-Line blocking.**

The mental model is:

```text
HTTP/2
  ↓
Many logical streams
  ↓
One TCP connection
  ↓
One ordered byte stream
```

If one TCP packet is lost, other HTTP/2 streams can be delayed by TCP's ordering/retransmission behavior.

This is different from saying the HTTP/2 streams themselves are not logically separate. They are separate at the HTTP/2 layer; the limitation comes from the transport underneath.

---

# 19. HTTP/3 and QUIC

HTTP/3 changes the transport architecture.

HTTP/2:

```text
HTTP/2
   ↓
HTTP/2 streams
   ↓
TCP
   ↓
One ordered byte stream
```

HTTP/3:

```text
HTTP/3
   ↓
HTTP/3 streams
   ↓
QUIC
   ↓
UDP
```

QUIC provides independent streams at the transport level.

If Stream A loses a packet:

```text
Stream A → packet lost ❌ → Stream A waits

Stream B → continues ✅
Stream C → continues ✅
```

This avoids the same cross-stream TCP Head-of-Line blocking problem.

This is one of the major architectural advantages of HTTP/3/QUIC over HTTP/2/TCP.

---

# 20. The complete mental model

Keep these layers separate:

```text
                    gRPC
                      │
             RPC / messages
                      │
                    HTTP/2
                      │
        ┌─────────────┼─────────────┐
        │             │             │
     Stream 1      Stream 3      Stream 5
        │             │             │
        └─────────────┼─────────────┘
                      │
                     TCP
                      │
                      IP
                      │
                   Network
```

The important distinction is:

```text
TCP connection
      ≠
HTTP/2 stream
      ≠
gRPC message
```

They are different concepts.

---

# 21. Most important HTTP/2 features for gRPC

If learning HTTP/2 specifically to understand gRPC, prioritize:

1. **Multiplexing** — multiple streams share one TCP connection.
2. **Streams** — logical bidirectional channels inside the HTTP/2 connection.
3. **Binary framing** — HTTP/2 breaks communication into binary frames.
4. **HPACK** — HTTP/2 header compression.
5. **Flow control** — connection-level and stream-level protection against overwhelming receivers.
6. **Long-lived connections** — connections can be reused for many RPCs.
7. **TCP Head-of-Line blocking** — packet loss can delay multiple HTTP/2 streams because they share one ordered TCP byte stream.
8. **HTTP/3 + QUIC** — independent transport streams avoid the same cross-stream TCP HOL problem.

---

# 22. One-sentence summary

> **gRPC normally uses HTTP/2, where many logical streams can be multiplexed over a long-lived TCP connection; each RPC generally maps to an HTTP/2 stream, gRPC streaming can carry multiple messages over a stream, HTTP/2 uses binary frames, HPACK compression, and flow control, but because HTTP/2 commonly runs over one ordered TCP byte stream, packet loss can cause TCP-level Head-of-Line blocking across streams—one of the major problems addressed by HTTP/3 and QUIC.**
