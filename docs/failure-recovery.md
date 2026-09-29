# Failure Modes & Recovery Strategies

## Distributed Priority Task Queue & Resilient Job Orchestrator

This document outlines the failure modes addressed by the orchestrator, the recovery mechanisms used, and the processing semantics provided.

---

## 1. Processing Semantics: At-Least-Once Delivery

The system guarantees **at-least-once processing**.

### Why Not Exactly-Once?
In distributed systems, the Two Generals Problem and network partitions prevent true exactly-once execution across independent distributed components without two-phase commit overhead or idempotency guarantees at the consumer boundary:

```text
Worker acquires Job A
       │
       ▼
Worker executes Job A (side effect occurs, e.g., payment processed)
       │
       ▼
Job A completes locally
       │
       X  <-- Worker crashes before network ACK reaches Redis / DB
       ▼
Reaper detects worker heartbeat timeout
       │
       ▼
Job A is requeued and dispatched to Worker B
       │
       ▼
Worker B executes Job A again
```

Therefore:
- Tasks must be designed to be **idempotent**.
- The system prevents concurrent duplicate execution under normal operation via atomic Lua scripts, but safely re-executes tasks when a worker abruptly crashes.

---

## 2. Failure Scenarios and Mitigations

| Failure Mode | Detection Mechanism | Recovery Action |
|---|---|---|
| **Worker Process Crash** (`kill -9`, segfault, OOM) | Missing heartbeats in `workers:heartbeats` exceeding `WORKER_HEARTBEAT_TIMEOUT_MS`. | Reaper detects timeout, marks worker `DEAD`, releases in-flight jobs back to `queue:jobs`, and logs recovery event. |
| **Worker Network Partition / Hung Thread** | Heartbeat timestamp stalls. | Reaper reclaims in-flight tasks. If the hung worker revives, its subsequent ACK is rejected because it no longer owns the in-flight lease. |
| **Transient Job Task Failure** (network blip, external API error) | Worker catches error / execution failure code. | Increments `attempts`. Computes backoff: `delay = BASE_RETRY_DELAY * 2^(attempts-1)`. Requeues job when delay elapses. |
| **Permanent Job Task Failure** (corrupt payload, fatal bug) | `attempts >= MAX_RETRIES`. | Job is transferred to Dead-Letter Queue (`queue:dlq`) and marked `DEAD` in PostgreSQL for operator intervention. |
| **Redis Restart / Failure** | Connection error handler in API / Workers. | Reconnect with exponential backoff. Persistent Redis AOF/RDB configuration preserves queued tasks across restarts. |
| **Database Failure** | Connection pooling error in Node.js / Worker. | Transactions fail gracefully; unacknowledged jobs remain tracked in Redis in-flight and are re-synchronized upon DB reconnection. |

---

## 3. Reaper Recovery Sequence

```text
             ┌──────────────────────┐
             │ Periodic Reaper Tick │
             └──────────┬───────────┘
                        │
                        ▼
             ┌──────────────────────┐
             │ Scan Heartbeat Index │
             └──────────┬───────────┘
                        │
             now - last_heartbeat > timeout?
                        │
                 ┌──────┴──────┐
                 │             │
                Yes            No
                 │             │
                 ▼             ▼
        ┌────────────────┐ ┌────────┐
        │ Mark Worker    │ │ Continue
        │ as DEAD        │ └────────┘
        └────────┬───────┘
                 │
                 ▼
        ┌────────────────────────────┐
        │ Find In-flight Jobs Bound   │
        │ to Dead Worker             │
        └────────┬───────────────────┘
                 │
                 ▼
        ┌────────────────────────────┐
        │ Atomic Lua Reclaim:        │
        │ - Remove from in-flight    │
        │ - Increment attempts       │
        │ - Check MAX_RETRIES        │
        │ - Push to queue:jobs / DLQ │
        └────────┬───────────────────┘
                 │
                 ▼
        ┌────────────────────────────┐
        │ Update PostgreSQL & Log    │
        │ Recovery Event             │
        └────────────────────────────┘
```
