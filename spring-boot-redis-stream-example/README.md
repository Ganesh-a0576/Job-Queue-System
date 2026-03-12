# ⚙️ Distributed Job Queue — Background Worker System

> A production-grade background job processing system built with **Spring Boot** and **Redis Streams** — designed for ordered, fault-tolerant, duplicate-free job execution.

---

##  Overview

This project implements a **distributed job queue** using Redis Streams as the messaging backbone. Clients submit jobs via a REST API; a producer enqueues them into a Redis Stream; a worker consumer processes them sequentially and acknowledges completion.

| Guarantee | Mechanism |
|---|---|
|  Ordered execution | Redis Stream FIFO semantics |
|  Persistence across restarts | Redis Stream log durability |
|  Worker failure recovery | Consumer group re-delivery |
|  Duplicate prevention | Job ID deduplication in `JobService` |

---

##  Architecture

```
┌──────────────┐
│    Client    │  HTTP POST /jobs/start
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│   Spring Boot API    │  JobController → JobService
└──────┬───────────────┘
       │  validates + deduplicates
       ▼
┌──────────────────────┐
│    Job Producer      │  XADD → Redis Stream
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│    Redis Stream      │  Persistent ordered message log
│    (Valkey compat.)  │
└──────┬───────────────┘
       │  XREADGROUP
       ▼
┌──────────────────────┐
│    Job Consumer      │  Subscribes · Processes · ACKs
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  Worker Processing   │  Sequential job execution
└──────────────────────┘
```

>  Place your architecture diagram at `/docs/architecture.png` and reference it here:
> ```html
> <img src="docs/architecture.png" width="700"/>
> ```

### Request Flow

1. **Client** sends a `POST` request with a job payload
2. **JobController** receives and delegates to `JobService`
3. **JobService** validates the job and checks for duplicates
4. **JobProducer** publishes the job to the Redis Stream via `XADD`
5. **JobConsumer** reads jobs from the stream via `XREADGROUP`
6. **Worker** processes each job sequentially
7. Completed jobs are **acknowledged** (`XACK`) and cleared from the pending list

---

##  Tech Stack

| Layer | Technology |
|---|---|
| **Language** | Java |
| **Framework** | Spring Boot |
| **Queue / Messaging** | Redis Streams (Valkey compatible) |
| **Database** | Redis / Valkey |
| **Containerization** | Docker |
| **Build Tool** | Maven |

---

##  Project Structure

```
redis-stream-example/
└── src/main/java/com/example/redisstream/
    ├── controller/
    │   └── JobController.java             # REST API — add, list, health
    ├── service/
    │   └── JobService.java                # Validation + deduplication logic
    ├── producer/
    │   └── JobProducer.java               # Publishes jobs to Redis Stream
    ├── consumer/
    │   └── JobConsumer.java               # Subscribes + processes + ACKs
    ├── config/
    │   └── RedisStreamConfig.java         # Stream + consumer group setup
    └── RedisStreamExampleApplication.java
```

### Component Descriptions

**`JobProducer.java`** — Receives job requests from the API and publishes them into the Redis Stream using `XADD`.

**`JobConsumer.java`** — Subscribes to the Redis stream via a consumer group, processes jobs sequentially, and acknowledges each completed job with `XACK`.

**`JobService.java`** — Core business logic handling job validation, duplicate prevention, and queue state management.

**`JobController.java`** — Spring Boot REST controller exposing endpoints to submit jobs, list queued jobs, and check worker health.

**`RedisStreamConfig.java`** — Configures the Redis Stream connection, consumer groups, and stream listener container.

---

##  API Reference

### Add Job to Queue

```http
POST /redis-stream-example/v1/jobs/start
Content-Type: application/json
```

**Request Body:**
```json
{
  "id": 1,
  "name": "example job"
}
```

**cURL:**
```bash
curl -X POST http://localhost:8080/redis-stream-example/v1/jobs/start \
  -H 'Content-Type: application/json' \
  -d '{"id":1,"name":"example job"}'
```

---

### Get Queued Jobs

```http
GET /redis-stream-example/v1/jobs/queued
```

```bash
curl http://localhost:8080/redis-stream-example/v1/jobs/queued
```

---

### Health Check

```http
GET /redis-stream-example/actuator/health
```

Verifies: Redis stream connection · worker subscription · overall application health

```bash
curl http://localhost:8080/redis-stream-example/actuator/health
```

---

##  Getting Started

### Option A — Docker (Recommended)

**Step 1 — Start Redis / Valkey**
```bash
docker run -p 6379:6379 valkey/valkey
```

**Step 2 — Run the application**
```bash
mvn spring-boot:run
```

Server starts at → `http://localhost:8080`

---

### Option B — Manual Setup

**1. Start Redis or Valkey**
```bash
redis-server
# or
valkey-server
```

**2. Build the project**
```bash
mvn clean install
```

**3. Run the application**
```bash
mvn spring-boot:run
```

---

##  Job Processing Guarantees

| Property | Description |
|---|---|
| **Ordered Execution** | Jobs processed in exact submission order |
| **Persistence** | Jobs survive application restarts via Redis Stream log |
| **Worker Recovery** | Unacknowledged jobs are re-delivered after worker failure |
| **Deduplication** | Duplicate job IDs are rejected before entering the queue |

---

##  Scalability

This architecture scales horizontally without code changes:

- **Multiple consumers** — Add more `JobConsumer` instances to parallelize processing
- **Stream partitioning** — Distribute load across multiple Redis Streams by key
- **Horizontal API scaling** — Stateless Spring Boot layer supports multiple replicas
- **Redis Cluster** — Scale the message broker across nodes for high availability

---

##  Future Improvements

- [ ] Job retry with exponential backoff
- [ ] Priority queues for urgent job types
- [ ] Distributed worker pools across services
- [ ] Job monitoring dashboard (status, throughput, lag)
- [ ] Dead letter queue (DLQ) for persistently failed jobs

---

##  License

MIT © Bathula Ganesh
