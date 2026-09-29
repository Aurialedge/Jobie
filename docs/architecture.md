# Architecture Overview

## Distributed Priority Task Queue & Resilient Job Orchestrator

This document details the architectural layout, core subsystems, data flow, and distributed coordination principles of the system.

---

## 1. System Topology

```text
                         ┌─────────────────────┐
                         │       Client        │
                         │   (HTTP REST / SDK) │
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
              │    Redis     │             │  PostgreSQL  │
              │              │             │              │
              │ Priority     │             │ Durable Jobs │
              │ In-flight    │             │ State Log    │
              │ Heartbeats   │             │ Worker Info  │
              └──────┬───────┘             └──────────────┘
                     │
            ┌────────┼────────┐
            │        │        │
            ▼        ▼        ▼
       ┌─────────┐┌─────────┐┌─────────┐
       │ Worker 1││ Worker 2││ Worker 3│
       │  (C++)  ││  (C++)  ││  (C++)  │
       │ Thread  ││ Thread  ││ Thread  │
       │  Pool   ││  Pool   ││  Pool   │
       └─────────┘└─────────┘└─────────┘
            │
            │ periodic heartbeats
            ▼
       ┌───────────────┐
       │     Reaper    │
       │ Crash Recovery│
       └───────────────┘
```

---

## 2. Core Subsystems

### 2.1 API Gateway (Node.js & Express)
- **Role**: Ingestion, validation, query interface, and operational control.
- **Responsibilities**:
  - Accept job submissions with priority and payload.
  - Insert durable job records into PostgreSQL with state `QUEUED`.
  - Push jobs into the Redis Priority Queue sorted set with inverted priority scores.
  - Expose query and management endpoints (`/jobs`, `/queue`, `/workers`, `/dlq`, `/health`).

### 2.2 Redis Coordination Engine
- **Role**: Ultra-low-latency in-memory queueing, atomic scheduling, and ephemeral state tracking.
- **Data Structures**:
  - `queue:jobs` (Sorted Set): Holds waiting jobs scored by `-priority`.
  - `jobs:inflight` (Hash or Set): Tracks currently active jobs mapped to executing worker IDs.
  - `workers:heartbeats` (Sorted Set or Hash): Tracks worker liveness timestamps.
  - `queue:dlq` (List or Sorted Set): Permanently dead jobs after exhausted retries.
- **Atomicity**: Critical multi-key transitions (popping top job, adding to in-flight, associating worker) are executed via Redis Lua scripts to eliminate race conditions.

### 2.3 C++ Worker Engine
- **Role**: High-performance compute engine executing prioritized tasks concurrently.
- **Concurrency Model**:
  - Dedicated Dispatcher thread fetches tasks atomically from Redis.
  - Multi-threaded Worker Pool (`std::thread`, `std::mutex`, `std::condition_variable`) dispatches tasks without blocking the main event/fetch loop.
  - Periodic background heartbeat thread publishes liveness updates to Redis.

### 2.4 Reaper & Crash Recovery Engine
- **Role**: Continuous safety guardian detecting crashed or unresponsive workers.
- **Responsibilities**:
  - Scans `workers:heartbeats` for timestamps older than `WORKER_HEARTBEAT_TIMEOUT_MS`.
  - Identifies stranded in-flight jobs associated with dead workers.
  - Atomically reclaims jobs, increments attempt counts, and requeues them or routes them to DLQ.
  - Marks dead workers in PostgreSQL and Redis.

### 2.5 PostgreSQL Persistence Layer
- **Role**: Source of truth for durable metadata, execution history, and audit trails.
- **Tables**:
  - `jobs`: Permanent record of all jobs, current status, retry count, error history.
  - `workers`: Worker registration records and status history.
  - `job_events`: Append-only event log capturing every state transition for auditing and observability.

---

## 3. Data Flow & Job Lifecycle

1. **Submission**: Client calls `POST /jobs`. The API writes the job to PostgreSQL (`QUEUED`), then adds it to Redis `queue:jobs`.
2. **Acquisition**: A C++ worker calls the atomic Lua acquisition script, which moves the highest-priority job from `queue:jobs` to `jobs:inflight`.
3. **Execution**: The job is passed to a worker thread pool thread. Job status in PostgreSQL is transitioned to `RUNNING`.
4. **Completion**:
   - On success: Worker removes job from `jobs:inflight`, updates PostgreSQL to `COMPLETED`.
   - On failure: If attempts < `MAX_RETRIES`, job is delayed with exponential backoff and requeued to `QUEUED`. If attempts >= `MAX_RETRIES`, job is moved to `queue:dlq` and marked `DEAD`.
5. **Crash Recovery**: If a worker abruptly halts, its heartbeat expires. The Reaper reclaims the job from `jobs:inflight` and places it back in `queue:jobs`.
