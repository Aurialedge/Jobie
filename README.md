# Distributed Priority Task Queue & Resilient Job Orchestrator

A distributed, fault-tolerant job orchestration system that provides priority-based task scheduling, concurrent worker execution, retry handling, dead-letter queues, worker heartbeats, and automatic recovery from worker failures.

The project is designed to demonstrate practical distributed-systems and backend engineering concepts including atomic distributed coordination, concurrency, fault recovery, queue semantics, synchronization, persistence, and containerized service orchestration.

> **Project status:** Under active development. Benchmark numbers will be added only after being measured on the completed implementation.

---

# Architecture

```text
                         ┌─────────────────────┐
                         │       Client        │
                         │     REST API        │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │    Node.js API      │
                         │      Express        │
                         └──────────┬──────────┘
                                    │
                     ┌──────────────┴──────────────┐
                     │                             │
                     ▼                             ▼
              ┌──────────────┐             ┌──────────────┐
              │    Redis     │             │ PostgreSQL   │
              │              │             │              │
              │ Priority     │             │ Job metadata │
              │ Queue        │             │ Job history  │
              │ In-flight    │             │ Worker state │
              │ Heartbeats   │             │              │
              └──────┬───────┘             └──────────────┘
                     │
            ┌────────┼────────┐
            │        │        │
            ▼        ▼        ▼
       ┌─────────┐┌─────────┐┌─────────┐
       │ Worker 1││ Worker 2││ Worker 3│
       │  C++    ││  C++    ││  C++    │
       │Thread   ││Thread   ││Thread   │
       │Pool     ││Pool     ││Pool     │
       └─────────┘└─────────┘└─────────┘
            │
            │ heartbeat monitoring
            ▼
       ┌───────────────┐
       │     Reaper    │
       │ Crash Recovery│
       └───────────────┘
```

---

# Key Features

- Priority-based distributed task scheduling
- Redis Sorted Set based priority queue
- Atomic job acquisition using Redis Lua scripts
- Concurrent C++ worker nodes
- Custom C++ thread pool
- Mutex and condition-variable based synchronization
- Exponential-backoff retry mechanism
- Dead-Letter Queue for permanently failed jobs
- Worker heartbeat monitoring
- Automatic detection of crashed/zombie workers
- Automatic requeueing of unacknowledged jobs
- PostgreSQL-backed job metadata and execution history
- REST API for job submission and monitoring
- Docker Compose based multi-service deployment
- Horizontal worker scaling
- Throughput and latency benchmarking

---

# Processing Semantics

The system is designed around **at-least-once job processing semantics**.

Atomic acquisition prevents multiple healthy workers from simultaneously acquiring the same queued job.

However, a worker can theoretically complete a job and crash before acknowledging completion:

```text
Worker acquires Job A
       |
       v
Worker executes Job A
       |
       v
Job A completes
       |
       X
Worker crashes before acknowledgement
       |
       v
Reaper detects stale worker
       |
       v
Job A is requeued
       |
       v
Another worker may execute Job A
```

Therefore, jobs should be designed to be idempotent when duplicate execution is possible.

The system does **not** claim exactly-once execution.

---

# Job Lifecycle

```text
             ┌──────────┐
             │ CREATED  │
             └────┬─────┘
                  │
                  ▼
             ┌──────────┐
             │  QUEUED  │
             └────┬─────┘
                  │
                  ▼
             ┌──────────┐
             │ RUNNING  │
             └────┬─────┘
                  │
           ┌──────┴───────┐
           │              │
           ▼              ▼
      ┌──────────┐   ┌─────────┐
      │COMPLETED │   │ FAILED  │
      └──────────┘   └────┬────┘
                           │
                      retry available?
                       /         \
                     yes          no
                      │            │
                      ▼            ▼
                   QUEUED         DLQ
```

---

# Technology Stack

| Component | Technology |
|---|---|
| API Gateway | Node.js, Express |
| Worker Engine | C++ |
| Queue | Redis |
| Atomic Operations | Redis Lua |
| Database | PostgreSQL |
| Containerization | Docker |
| Orchestration | Docker Compose |
| Communication | REST, Redis |
| Build System | CMake |

---

# Project Structure

```text
distributed-job-orchestrator/
│
├── api/
│   ├── src/
│   │   ├── server.js
│   │   ├── routes/
│   │   ├── services/
│   │   ├── redis/
│   │   │   └── scripts/
│   │   └── db/
│   ├── package.json
│   └── Dockerfile
│
├── worker/
│   ├── include/
│   ├── src/
│   ├── tests/
│   ├── CMakeLists.txt
│   └── Dockerfile
│
├── reaper/
│   ├── src/
│   ├── package.json
│   └── Dockerfile
│
├── benchmark/
│   ├── producer.js
│   ├── benchmark.js
│   └── results/
│
├── docs/
│   ├── architecture.md
│   ├── failure-recovery.md
│   └── benchmarking.md
│
├── docker-compose.yml
├── PHASES.md
├── README.md
└── .gitignore
```

---

# API

## Create Job

```http
POST /jobs
```

Example:

```json
{
  "type": "cpu",
  "payload": {
    "duration": 500
  },
  "priority": 10
}
```

Example response:

```json
{
  "jobId": "job_123",
  "status": "queued"
}
```

## Get Job

```http
GET /jobs/:id
```

## List Jobs

```http
GET /jobs
```

## Retry Job

```http
POST /jobs/:id/retry
```

## Cancel Job

```http
POST /jobs/:id/cancel
```

## List Workers

```http
GET /workers
```

## Get Queue Status

```http
GET /queue
```

## Get Dead-Letter Queue

```http
GET /dlq
```

## Health Check

```http
GET /health
```

---

# Priority Scheduling

Jobs are stored in a Redis Sorted Set.

Conceptually:

```text
priority 20 → score -20
priority 10 → score -10
priority 5  → score -5
```

Lower Redis scores are processed first, allowing higher-priority jobs to be selected before lower-priority jobs.

---

# Atomic Job Acquisition

A naive implementation could perform:

```text
read highest-priority job
        ↓
remove job
        ↓
mark job as running
```

as separate Redis commands.

With multiple workers, this can introduce race conditions.

The project uses a Redis Lua script to perform the critical acquisition sequence atomically.

```text
Worker A ──┐
Worker B ──┼──► Redis Lua Script
Worker C ──┘
                 │
                 ▼
        Atomic job acquisition
```

This allows concurrent workers to safely compete for queued jobs.

---

# Worker Thread Pool

Each C++ worker contains a thread pool:

```text
                Dispatcher
                    │
                    ▼
               Job Queue
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       Thread 1  Thread 2  Thread 3
          │         │         │
          ▼         ▼         ▼
        Job A     Job B     Job C
```

The worker uses C++ concurrency primitives including:

- `std::thread`
- `std::mutex`
- `std::condition_variable`
- task queues
- synchronization mechanisms

---

# Retry Mechanism

Failed jobs are retried using exponential backoff.

Example:

```text
Attempt 1 → failure
     ↓
1 second
     ↓
Attempt 2 → failure
     ↓
2 seconds
     ↓
Attempt 3 → failure
     ↓
4 seconds
     ↓
Maximum attempts reached
     ↓
Dead-Letter Queue
```

Conceptually:

```text
delay = baseDelay × 2^attempt
```

Jitter may be introduced to reduce synchronized retry bursts.

---

# Dead-Letter Queue

Jobs that exceed the configured retry limit are moved to:

```text
queue:dlq
```

The DLQ allows operators to inspect permanently failed jobs without losing their metadata.

---

# Worker Heartbeats

Workers periodically publish heartbeat information.

```text
Worker 1
   │
   ├── heartbeat
   ├── heartbeat
   ├── heartbeat
   └── heartbeat
```

The reaper monitors heartbeat timestamps.

If a worker exceeds the configured heartbeat timeout:

```text
Worker heartbeat expires
          ↓
Worker marked as dead
          ↓
Find in-flight jobs
          ↓
Requeue unacknowledged jobs
          ↓
Other workers process them
```

---

# Failure Recovery

Example failure scenario:

```text
                 Worker 1
                    │
                    │ Job 101
                    ▼
                  RUNNING
                    │
                    X
                  CRASH
                    │
                    ▼
             Heartbeat timeout
                    │
                    ▼
                 Reaper
                    │
                    ▼
              Job 101 found
                    │
                    ▼
                 REQUEUE
                    │
                    ▼
              Worker 2
                    │
                    ▼
               Job 101
                    │
                    ▼
                COMPLETE
```

---

# Running Locally

## Prerequisites

- Docker
- Docker Compose
- Git

Clone the repository:

```bash
git clone <repository-url>
cd distributed-job-orchestrator
```

Start the complete cluster:

```bash
docker compose up --build
```

---

# Scaling Workers

Multiple worker containers can be started with Docker Compose:

```bash
docker compose up --build --scale worker=3
```

Resulting architecture:

```text
              Redis
             /  |  \
            /   |   \
           ▼    ▼    ▼
       Worker1 Worker2 Worker3
```

---

# Failure Recovery Demo

Start multiple workers:

```bash
docker compose up --build --scale worker=3
```

Submit several jobs.

Then terminate one worker:

```bash
docker stop <worker-container>
```

Expected sequence:

```text
Worker crashes
     ↓
Heartbeat stops
     ↓
Reaper detects timeout
     ↓
In-flight jobs are identified
     ↓
Jobs are requeued
     ↓
Healthy workers acquire jobs
```

---

# Benchmarking

The project includes a benchmark suite for measuring:

- Job throughput
- Dispatch latency
- End-to-end latency
- Retry overhead
- Worker scaling
- Failure recovery time

Example benchmark dimensions:

| Workers | Jobs | Throughput | P50 Dispatch | P95 Dispatch |
|---:|---:|---:|---:|---:|
| 1 | 10,000 | TBD | TBD | TBD |
| 2 | 10,000 | TBD | TBD | TBD |
| 4 | 10,000 | TBD | TBD | TBD |
| 8 | 10,000 | TBD | TBD | TBD |

> Benchmark values will be populated only from actual measurements of the implemented system.

---

# Design Goals

The project focuses on several distributed-systems problems.

### 1. Concurrent Job Acquisition

Multiple workers must safely compete for jobs without concurrently acquiring the same queued job.

### 2. Fault Recovery

Worker failures should not permanently lose unacknowledged in-flight jobs.

### 3. Priority Scheduling

Higher-priority tasks should be selected before lower-priority tasks.

### 4. Retry Handling

Transient failures should be retried without immediately losing the job.

### 5. Persistent Metadata

PostgreSQL maintains durable job and execution metadata.

### 6. Horizontal Worker Scaling

Additional worker instances can be added without modifying the API layer.

---

# Important Semantics

The system aims for:

```text
Atomic job acquisition
+
At-least-once processing
+
Crash recovery
+
Priority scheduling
```

It does not claim exactly-once execution.

Exactly-once behavior in distributed systems is substantially more complicated because a worker can fail between completing a task and recording its completion.

---

# Future Improvements

Potential extensions include:

- Job scheduling for future timestamps
- Job dependencies
- DAG-based workflows
- Rate limiting
- Per-tenant queues
- Priority aging to reduce starvation
- Prometheus metrics
- Grafana dashboards
- Authentication and authorization
- Distributed tracing
- Kubernetes deployment
- Redis Sentinel/Cluster
- PostgreSQL replication
- Idempotency keys
- Administrative dashboard

---

# Learning Objectives

This project demonstrates practical experience with:

- Distributed task scheduling
- Redis data structures
- Redis Lua scripting
- Race-condition prevention
- C++ multithreading
- Thread pools
- Mutexes and condition variables
- Retry and backoff strategies
- Dead-letter queues
- Heartbeat-based failure detection
- Crash recovery
- At-least-once delivery
- REST API design
- PostgreSQL persistence
- Docker containerization
- Horizontal scaling
- Performance benchmarking

---

# License

MIT License