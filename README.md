# System_Design

Interview-ready system design write-ups, converted and expanded from `System Design and Architecture.pptx`. Each document includes a Mermaid architecture diagram, component-by-component justification (what it does and *why that technology*), scale/concurrency handling, back-of-envelope bandwidth/storage estimates, a list of bugs/gaps found in the original slide with the fix applied, and a bank of likely interviewer questions with pointers to the answer.

## Designs

| # | Design | Key ideas covered |
|---|---|---|
| 1 | [Notification System](designs/01-notification-system.md) | Priority topics without starvation, idempotency, per-vendor rate limiting, DLQ |
| 2 | [WhatsApp / Chat System](designs/02-whatsapp-chat-system.md) | **Fixes a real race condition** in the original design, corrects the "64K connections" myth |
| 3 | [Twitter / X Timeline](designs/03-twitter-timeline.md) | Hybrid fan-out-on-write / fan-out-on-read, the celebrity problem, adds missing search component |
| 4 | [TinyURL](designs/04-tinyurl.md) | Snowflake/ticket-server ID generation, adds the missing read path + cache, 301 vs 302 trade-off |
| 5 | [Web Crawler](designs/05-web-crawler.md) | **Adds missing `robots.txt` compliance**, fleet-wide politeness via consistent hashing, crawler traps |
| 6 | [ETL vs ELT & Streaming Pipelines](designs/06-etl-elt-data-pipeline.md) | Decision framework, corrects JDBC polling vs. true CDC, columnar storage rationale |
| 7 | [Airflow-style Task Scheduler](designs/07-airflow-task-scheduler.md) | Adds the missing Triggerer and DAG File Processor, Celery vs. Kubernetes executor trade-off |
| 8 | [Real-Time Price Feed](designs/08-realtime-price-feed.md) | **Fixes a missing OHLC field** and a cache-key design flaw, consistent-hash fan-out, backpressure |

Original source: [`System Design and Architecture.pptx`](System%20Design%20and%20Architecture.pptx).

## Concept deep-dives

Standalone explanations of recurring interview topics — not tied to one system design.

| Design | Key ideas covered |
|---|---|
| [Interview Concept Deep-Dives](designs/10-interview-concepts-deep-dive.md) | Atomicity/ACID, CQRS, Command vs. Strategy pattern, TCP statefulness, **Kafka internals** (partitions, throughput, offset storage), **RabbitMQ internals** (exchanges, AMQP), Kafka vs. RabbitMQ, outbox pattern, adapter pattern, plus CAP theorem, saga, circuit breaker, consistent hashing, rate limiting, single source of truth, and more |

## Real-world systems

Write-ups of actual systems (not interview practice) — same depth of component-by-component justification, scale/concurrency handling, and concrete improvements.

| Design | Key ideas covered |
|---|---|
| [Optima — Hotel Channel Manager](designs/09-optima-channel-manager.md) | Why RabbitMQ over Kafka here specifically, the per-module `EXT` facade pattern, multi-tenant isolation, the adapter pattern for adding new OTA providers, transactional outbox fix for a dual-write gap |
