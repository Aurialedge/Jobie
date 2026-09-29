# Development Phases

# Distributed Priority Task Queue & Resilient Job Orchestrator

This document defines the incremental development plan for the project.

The project must be implemented **phase-by-phase**.

## Critical Development Rules

1. Do NOT implement all phases at once.
2. Implement only the current phase.
3. Do not introduce technologies that are not required by the current phase.
4. Do not create fake benchmark numbers.
5. Do not mark a feature as complete until it has been tested.
6. Preserve working functionality from previous phases.
7. After each phase:
   - build the project
   - run tests
   - run integration checks where applicable
   - fix errors
   - document important decisions
8. Do not move to the next phase until explicitly instructed.
9. Prefer simple, maintainable implementations over unnecessary abstraction.
10. All important distributed-systems behavior must be explainable by the developer.

---

# Phase 0 — Project Foundation

## Objective

Create the repository foundation and development environment.

## Tasks

- Create repository structure.
- Create `.gitignore`.
- Create `README.md`.
- Create `PHASES.md`.
- Create initial `docs/` directory.
- Create Docker Compose skeleton.
- Establish environment-variable configuration.
- Add basic project documentation.

## Deliverables

```text
README.md
PHASES.md
.gitignore
docker-compose.yml
docs/
```

## Exit Criteria

- Repository structure is clean.
- README explains the project.
- Docker Compose configuration is syntactically valid.
- No application logic is implemented yet.

---

# Phase 1 — REST API + PostgreSQL Foundation

## Objective

Build the basic job-management API and persistent storage layer.

## Components

```text
Node.js
Express
PostgreSQL
Docker
```

## Implement

### API

```http
POST /jobs
GET /jobs
GET /jobs/:id
GET /health
```

### PostgreSQL

Create:

```text
jobs
workers
job_events
```

The `jobs` table should contain:

```text
id
type
payload
priority
status
attempts
worker_id
created_at
started_at
completed_at
last_error
```

Initial statuses:

```text
QUEUED
RUNNING
COMPLETED
FAILED
DEAD
```

## Requirements

- Parameterized SQL queries.
- Environment-based database configuration.
- Request validation.
- Proper HTTP status codes.
- Centralized error handling.
- Clean separation between routes, services, and database access.

## Testing

Verify:

```text
POST /jobs
GET /jobs
GET /jobs/:id
GET /health
```

## Exit Criteria

A job can be:

```text
HTTP request
      ↓
Express
      ↓
PostgreSQL
      ↓
persisted job
```

No Redis functionality yet.

No C++ worker yet.

---

# Phase 2 — Redis Integration

## Objective

Introduce Redis as the high-speed coordination layer.

## Components

```text
Node.js
Redis
PostgreSQL
```

## Implement

Add Redis to Docker Compose.

Create Redis client abstraction.

Introduce:

```text
queue:jobs
```

using a Redis Sorted Set.

Priority representation:

```text
priority 20 → score -20
priority 10 → score -10
priority 5  → score -5
```

## Job Submission Flow

Change:

```text
POST /jobs
      ↓
PostgreSQL
```

to:

```text
POST /jobs
      ↓
PostgreSQL
      ↓
Redis priority queue
```

## Add

```http
GET /queue
```

## Testing

Submit jobs with different priorities.

Verify Redis contains the expected ordering.

## Exit Criteria

Jobs are persisted in PostgreSQL and queued in Redis.

Do not implement Lua yet.

---

# Phase 3 — Atomic Job Acquisition with Redis Lua

## Objective

Solve the concurrent job-acquisition race condition.

## Problem

Avoid:

```text
ZRANGE
  ↓
ZREM
  ↓
state update
```

as independent operations.

## Implement

Create:

```text
api/src/redis/scripts/acquireJob.lua
```

The script should atomically:

1. Identify the highest-priority available job.
2. Remove it from the priority queue.
3. Register it as in-flight.
4. Associate it with a worker.
5. Return the job information.

Create an in-flight structure such as:

```text
jobs:inflight
```

or an appropriate Redis data structure.

## Requirements

The operation must be atomic from Redis's perspective.

## Testing

Simulate multiple concurrent acquisition requests.

Verify:

```text
One queued job
+
Multiple concurrent workers
=
One successful acquisition
```

## Exit Criteria

Atomic job acquisition is working and tested.

---

# Phase 4 — C++ Worker Engine

## Objective

Create the first real worker capable of acquiring and executing jobs.

## Components

```text
C++
CMake
Redis
```

## Create

```text
worker/
├── include/
├── src/
├── tests/
└── CMakeLists.txt
```

## Worker Responsibilities

```text
Start
 ↓
Register worker
 ↓
Acquire job
 ↓
Execute job
 ↓
Report result
 ↓
Repeat
```

## Job Types

Initially support simulated jobs such as:

```text
CPU
SUCCESS
FAIL
SLOW
```

Example payload:

```json
{
  "duration": 500
}
```

## Requirements

- Clean C++ architecture.
- RAII where appropriate.
- Proper error handling.
- Graceful shutdown.
- Redis communication abstraction.

## Exit Criteria

A C++ worker can:

```text
Redis queue
    ↓
Acquire job
    ↓
Execute job
    ↓
Update job result
```

---

# Phase 5 — C++ Thread Pool

## Objective

Convert the worker from single-threaded execution into a concurrent worker.

## Implement

A reusable thread pool using:

```cpp
std::thread
std::mutex
std::condition_variable
std::queue
```

Architecture:

```text
             Worker Dispatcher
                    │
                    ▼
               Task Queue
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       Thread 1  Thread 2  Thread 3
          │         │         │
          ▼         ▼         ▼
        Job A     Job B     Job C
```

## Requirements

- Configurable number of worker threads.
- Thread-safe task queue.
- Proper worker shutdown.
- No busy waiting.
- Correct synchronization.

## Testing

Submit multiple jobs.

Verify multiple jobs execute concurrently.

## Exit Criteria

A single C++ worker can process multiple jobs concurrently using its thread pool.

---

# Phase 6 — Job Completion and State Consistency

## Objective

Make job state transitions reliable across Redis and PostgreSQL.

## Implement

Clear state transitions:

```text
QUEUED
  ↓
RUNNING
  ↓
COMPLETED
```

Failure:

```text
RUNNING
  ↓
FAILED
```

Retryable failure:

```text
FAILED
  ↓
QUEUED
```

Permanent failure:

```text
FAILED
  ↓
DEAD
```

## Requirements

Define exactly:

- when Redis state changes
- when PostgreSQL state changes
- when worker acknowledgement occurs
- what happens if a database operation fails
- what happens if Redis becomes unavailable

## Exit Criteria

Job state transitions are deterministic and documented.

---

# Phase 7 — Retry + Exponential Backoff

## Objective

Handle transient job failures.

## Implement

Configurable:

```text
MAX_RETRIES
BASE_RETRY_DELAY
```

Backoff:

```text
delay = baseDelay × 2^attempt
```

Example:

```text
Attempt 1 → 1s
Attempt 2 → 2s
Attempt 3 → 4s
Attempt 4 → 8s
```

Optionally add jitter.

## Requirements

- Increment attempt count.
- Preserve job metadata.
- Avoid tight retry loops.
- Retry only appropriate failures.

## Exit Criteria

Failed jobs are automatically retried according to configuration.

---

# Phase 8 — Dead-Letter Queue

## Objective

Prevent permanently failing jobs from retrying forever.

## Implement

Create:

```text
queue:dlq
```

When:

```text
attempts >= MAX_RETRIES
```

move the job to the DLQ.

## API

Implement:

```http
GET /dlq
POST /jobs/:id/retry
```

## Testing

Create a job guaranteed to fail.

Verify:

```text
failure
 ↓
retry
 ↓
retry
 ↓
retry
 ↓
DLQ
```

## Exit Criteria

Permanently failing jobs are moved to the DLQ and can be inspected.

---

# Phase 9 — Worker Registration + Heartbeats

## Objective

Detect worker failures.

## Implement

Workers must register themselves.

Worker state should include:

```text
worker_id
status
last_heartbeat
current_job
registered_at
```

Heartbeat interval should be configurable.

Example:

```text
Worker 1
   ↓
heartbeat
   ↓
Redis
   ↓
heartbeat
   ↓
Redis
```

## Requirements

- Unique worker ID.
- Graceful registration.
- Periodic heartbeat.
- Graceful shutdown.
- Worker status tracking.

## API

Implement:

```http
GET /workers
GET /workers/:id
```

## Exit Criteria

The system can determine whether workers are alive based on heartbeat timestamps.

---

# Phase 10 — Reaper / Crash Recovery

## Objective

Automatically recover jobs from failed workers.

## Create

```text
reaper/
```

The reaper periodically checks worker heartbeats.

Conceptually:

```text
Current time
     -
last heartbeat
     =
heartbeat age
```

If:

```text
heartbeat age > timeout
```

mark worker as dead.

## Recovery

For a dead worker:

```text
find in-flight jobs
       ↓
verify ownership
       ↓
requeue jobs
       ↓
remove stale in-flight state
       ↓
update persistent metadata
```

Critical recovery operations should use atomic Redis operations/Lua where required.

## Testing

Run:

```text
Worker 1
Worker 2
Worker 3
```

Assign jobs.

Kill Worker 2.

Verify:

```text
Worker 2 crashes
      ↓
Heartbeat expires
      ↓
Reaper detects failure
      ↓
Worker 2 marked DEAD
      ↓
In-flight jobs requeued
      ↓
Worker 1/3 acquire jobs
```

## Exit Criteria

Unacknowledged jobs from crashed workers are automatically recovered.

---

# Phase 11 — Dockerized Distributed Cluster

## Objective

Containerize the entire system.

## Services

```text
api
redis
postgres
reaper
worker
```

## Requirements

Each service should have an appropriate Dockerfile where needed.

Docker Compose should provide networking and environment configuration.

## Worker Scaling

Support:

```bash
docker compose up --build --scale worker=3
```

Expected:

```text
                 Redis
              /    |    \
             /     |     \
            ▼      ▼      ▼
        Worker1 Worker2 Worker3
```

## Exit Criteria

The entire distributed system starts through Docker Compose.

---

# Phase 12 — Integration Testing

## Objective

Test the complete system as a distributed application.

## Test Scenarios

### Test 1 — Priority

Submit:

```text
Priority 1
Priority 10
Priority 5
Priority 20
```

Verify priority ordering.

### Test 2 — Concurrent Acquisition

Submit many jobs with multiple workers.

Verify that the same queued job is not simultaneously acquired by multiple healthy workers.

### Test 3 — Successful Execution

```text
submit
 ↓
queue
 ↓
worker
 ↓
complete
```

### Test 4 — Retry

```text
submit
 ↓
failure
 ↓
retry
 ↓
success
```

### Test 5 — DLQ

```text
submit
 ↓
failure
 ↓
max retries
 ↓
DLQ
```

### Test 6 — Worker Crash

```text
worker
 ↓
running job
 ↓
kill worker
 ↓
heartbeat timeout
 ↓
reaper
 ↓
requeue
 ↓
another worker
```

### Test 7 — Multiple Workers

Run:

```bash
docker compose up --build --scale worker=4
```

Submit thousands of jobs.

Verify stable processing.

## Exit Criteria

All major scenarios pass.

---

# Phase 13 — Benchmarking

## Objective

Measure actual system performance.

Do NOT invent benchmark numbers.

## Measure

### Throughput

```text
completed jobs / second
```

### Dispatch latency

```text
job submitted
      ↓
worker receives job
```

Measure:

```text
P50
P95
P99
```

### End-to-End latency

```text
submission
    ↓
completion
```

### Recovery time

```text
worker crash
      ↓
job successfully requeued
```

## Benchmark Dimensions

Test multiple worker counts:

```text
1
2
4
8
```

And multiple workloads:

```text
1,000
10,000
50,000
100,000
```

## Record

```text
Workers
Jobs
Throughput
P50
P95
P99
Recovery Time
```

## Exit Criteria

Actual benchmark results are stored in:

```text
benchmark/results/
```

and documented in:

```text
docs/benchmarking.md
```

---

# Phase 14 — Observability + Production Polish

## Objective

Make the project presentable as a serious engineering project.

## Implement

Structured logs containing:

```text
timestamp
service
worker_id
job_id
event
status
attempt
latency
```

Example:

```text
[WORKER]
worker_id=worker-2
job_id=job-921
event=JOB_FAILED
attempt=2
```

## Add

- Graceful shutdown
- Configuration validation
- Consistent error responses
- Request logging
- Worker logging
- Redis error handling
- PostgreSQL error handling
- Health checks

Optional:

```text
Prometheus
Grafana
```

Do not add them if they complicate the core system unnecessarily.

## Exit Criteria

The system is stable and easy to demonstrate.

---

# Phase 15 — Documentation + Resume Validation

## Objective

Prepare the project for GitHub, resume, and interviews.

## Documentation

Finalize:

```text
README.md
PHASES.md
docs/architecture.md
docs/failure-recovery.md
docs/benchmarking.md
```

README should include:

- Architecture
- Features
- Technology stack
- API
- Setup instructions
- Docker instructions
- Worker scaling
- Failure recovery demo
- Benchmark results
- Design decisions
- Processing semantics

## Add

Architecture diagram.

Failure recovery diagram.

Benchmark table.

Example API requests.

Example logs.

## Resume Validation

Every resume claim must correspond to an implemented and demonstrable feature.

For example:

```text
"Redis Lua atomic job acquisition"
```

must have:

```text
actual Lua script
+
actual concurrent test
```

Similarly:

```text
"2,000+ jobs/sec"
```

must have:

```text
actual benchmark
+
test configuration
+
recorded result
```

Do not use unsupported performance claims.

## Final Exit Criteria

The project must be:

- Buildable
- Runnable
- Testable
- Dockerized
- Benchmarkable
- Documented
- Explainable in an interview

---

# Final Development Sequence

```text
PHASE 0
Foundation
   ↓
PHASE 1
REST API + PostgreSQL
   ↓
PHASE 2
Redis
   ↓
PHASE 3
Lua Atomic Acquisition
   ↓
PHASE 4
C++ Worker
   ↓
PHASE 5
C++ Thread Pool
   ↓
PHASE 6
State Consistency
   ↓
PHASE 7
Retry + Backoff
   ↓
PHASE 8
DLQ
   ↓
PHASE 9
Heartbeats
   ↓
PHASE 10
Reaper + Crash Recovery
   ↓
PHASE 11
Dockerized Cluster
   ↓
PHASE 12
Integration Testing
   ↓
PHASE 13
Benchmarking
   ↓
PHASE 14
Observability + Polish
   ↓
PHASE 15
Documentation + Resume
```

---

# Antigravity Operating Instructions

When working on this repository, follow these rules:

```text
READ README.md
       +
READ PHASES.md
       ↓
Identify current phase
       ↓
Implement ONLY current phase
       ↓
Test current phase
       ↓
Fix errors
       ↓
Explain implementation
       ↓
STOP
```

Never automatically continue to the next phase.

The developer must explicitly request the next phase.

Do not replace working implementations with unnecessary abstractions.

Do not fabricate test results, benchmark results, performance numbers, or reliability claims.

When a design decision has distributed-systems implications, explain the trade-off before implementing it.