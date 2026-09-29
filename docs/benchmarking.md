# Benchmarking Methodology & Performance Metrics

## Distributed Priority Task Queue & Resilient Job Orchestrator

This document outlines the benchmark harness, measurement methodology, performance indicators, and placeholder result tables.

> **Note on Benchmark Integrity**: All figures in this document will strictly reflect empirical measurements recorded during Phase 13. No synthetic or estimated values are published.

---

## 1. Key Performance Indicators (KPIs)

1. **Throughput (Jobs / Second)**:
   $$\text{Throughput} = \frac{\text{Total Completed Jobs}}{\text{Total Elapsed Seconds}}$$
2. **Dispatch Latency**:
   - Elapsed duration from `POST /jobs` HTTP submission to worker thread acquisition.
   - Percentiles: $P50$, $P95$, $P99$.
3. **End-to-End Latency**:
   - Elapsed duration from initial submission to final completion event in PostgreSQL.
   - Percentiles: $P50$, $P95$, $P99$.
4. **Crash Recovery Latency**:
   - Elapsed duration from worker termination (`SIGKILL`) to the moment the orphaned job is reclaimed by another worker.
5. **Horizontal Worker Scalability**:
   - Scaling efficiency as worker count increases: $1 \rightarrow 2 \rightarrow 4 \rightarrow 8$ workers.

---

## 2. Benchmark Matrix (To Be Measured in Phase 13)

| Test ID | Workers | Total Jobs | Job Type | Concurrency (Threads/Worker) | Throughput (jobs/sec) | P50 Dispatch (ms) | P95 Dispatch (ms) | P99 Dispatch (ms) |
|---|---:|---:|---|---:|---:|---:|---:|---:|
| BM-101 | 1 | 1,000 | NO-OP | 4 | TBD | TBD | TBD | TBD |
| BM-102 | 2 | 10,000 | NO-OP | 4 | TBD | TBD | TBD | TBD |
| BM-103 | 4 | 50,000 | NO-OP | 4 | TBD | TBD | TBD | TBD |
| BM-104 | 8 | 100,000 | NO-OP | 4 | TBD | TBD | TBD | TBD |
| BM-201 | 4 | 10,000 | CPU-bound (5ms) | 4 | TBD | TBD | TBD | TBD |
| BM-301 | 3 | 5,000 | Chaos / Crash Kill | 4 | TBD | TBD | TBD | TBD |

---

## 3. Tooling and Execution

The benchmark suite (`benchmark/producer.js` and `benchmark/benchmark.js`) will:
- Ingest high volumes of jobs across varying priority bands.
- Query PostgreSQL and Redis for real-time completion status and latency timestamps.
- Output raw metrics to CSV / JSON in `benchmark/results/`.
- Plot throughput curves and percentile histograms.
