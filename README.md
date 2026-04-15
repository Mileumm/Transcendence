# System Architecture & Observability Implementation 
*(Transcendence Showcase Repository)*

## 📌 Executive Summary
This repository serves as a technical showcase of a high-performance, distributed backend architecture. While the user-facing application is a real-time multiplayer chess platform, the core engineering focus of this project lies in its **infrastructure, strict data governance, and advanced observability**. 

It demonstrates production-ready patterns relevant to **Data Engineering and MLOps**, particularly in handling high-frequency streaming data, separating transactional from analytical workloads, and ensuring ACID compliance across a distributed stack.

---

## 🏗️ 1. System Design & The Asynchronous Data Bridge
Handling real-time gaming data introduces significant concurrency and latency challenges. Writing every move or timer tick directly to a relational database would create massive bottlenecking. To resolve this, I implemented an asynchronous, decoupled data bridge.

* **High-Frequency Ingestion (Redis):** The system relies on Redis as a low-latency, in-memory datastore to manage active game states. All real-time WebSocket events (moves, time degradation) are processed asynchronously via Django Channels and mutated directly in Redis.
* **ACID Persistence (PostgreSQL):** PostgreSQL is strictly reserved for the single source of truth. Data is only flushed from Redis to PostgreSQL atomically upon game termination.
* **Consistency Guarantees:** 
  * I implemented race-condition safeguards. Before executing a move, the system fetches the current state, validates standard timestamps (`turn_start_timestamp`), and applies the delta.
  * Timeouts and client disconnections are gracefully handled by server-side heartbeat checks, ensuring that interrupted processes are properly persisted to the database without data corruption or ghost states.

This architecture closely mirrors **streaming data pipelines**, where high-velocity data is buffered, aggressively processed in memory, and eventually persisted in batches.

---

## 🔍 2. Advanced Computational Offloading (Elasticsearch)
In typical architectures, Elasticsearch is relegated strictly to log searching. In this system, I elevated Elasticsearch to a **primary computational engine**, explicitly adopting a **CQRS (Command Query Responsibility Segregation)** pattern.

Instead of writing complex, expensive `GROUP BY` SQL queries to extract analytics from PostgreSQL's JSONB arrays, the analytical workload is entirely offloaded to the Elasticsearch cluster.

**Key Technical Achievements:**
* **Granular Data Aplatissement:** Game records are flattened upon ingestion. Player actions are aggregated to reconstruct a sequential, real-time timeline of move durations, bypassing arbitrary "average game times" in favor of absolute granularity.
* **Painless Scripting for Normalization:** I utilized advanced scripted nested aggregations (`Painless`) to dynamically calculate weighted player profiles. 
  * *Example:* To calculate a player's "Piece Usage Preference", the script iterates over the arrays inside ES, sums the occurrences, divides by respective game lengths to avoid biases from 100-move vs 10-move games, and applies a strict mathematical normalization to guarantee a perfect `100.0%` sum output, seamlessly handling heterogeneous legacy data.

By isolating analytical heavy lifting into Elasticsearch, the primary PostgreSQL instance retains maximum throughput for transactional operations.

---

## 📊 3. Observability Excellence (The ELK Pipeline)
A distributed system is inherently a black box without comprehensive telemetry. I deployed the **ELK Stack (Elasticsearch, Logstash, Kibana)** to guarantee full system observability and establish a rapid feedback loop.

* **Structured Logging:** The Python backend outputs highly structured, context-rich logs (e.g., precise move latency in milliseconds, indexing statuses, connection drops). 
* **Logstash Processing:** Acts as the data shipper and parser, structuring raw stdout logs into searchable JSON objects.
* **Kibana Telemetry:** Enables real-time debugging. During development, this pipeline allowed me to precisely track data leakage (e.g., milliseconds lost during async loops) and identify silent race conditions on the exact line of code.

---

## 💼 4. Professional Value: Preparedness for MLOps & Data Engineering
Building and optimizing this architecture directly translates to the core requirements of modern Data Engineering and MLOps roles:

1. **Pipeline Reliability:** Navigating the complexities of transferring state safely between Redis (volatile) and Postgres (persistent) demonstrates a robust understanding of fault-tolerant data pipelines.
2. **Compute vs. Storage Isolation:** Offloading heavy analytical aggregations to Elasticsearch proves an ability to architect scalable systems. In MLOps, keeping the prediction service (transactional) entirely isolated from the model monitoring metrics (analytical) is a fundamental best practice.
3. **Observability as a First-Class Citizen:** In machine learning systems, data drift and silent inference failures must be caught instantly. Designing a system governed by deep ELK telemetry ensures I am prepared to build infrastructures where anomalies are identified proactively, not reactively.

---
*Built as part of the 42 Network curriculum core program.*
