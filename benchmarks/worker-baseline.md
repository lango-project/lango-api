# Worker load baseline

Measured: 2026-11-05T02:46:17.202Z

- Node: v24.13.0
- CPU: 13th Gen Intel(R) Core(TM) i5-13420H; available parallelism: 12.
- 8 persistent HTTP clients, 100 measured requests each, 2 warmup requests each.
- A separate health observer probes throughout the measured workload.
- Each application request verifies and renews JWT claims and commits one SQLite
  insert (WAL, shared file, synchronous per-worker connections).
- CPU work: 0 ms or 5 ms busy work per application request.
- A common start barrier excludes process startup and warmup from throughput.
- HTTP latency includes network, host FIFO waiting, worker execution and body read.
- RSS covers the API process and its worker threads, excluding client processes.

| Workers | CPU ms | Requests/s | Request p50 ms | Request p99 ms | Request max ms | Health p99 ms | RSS MiB before → after |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 0 | 1089.92 | 6.57 | 18.47 | 28.14 | 3.72 | 98.46 → 132.75 |
| 2 | 0 | 1369.86 | 4.96 | 26.43 | 37.81 | 8.49 | 122.41 → 162.02 |
| 4 | 0 | 1436.27 | 4.43 | 21.14 | 45.26 | 6.09 | 167.76 → 213.60 |
| 1 | 5 | 164.95 | 47.77 | 63.78 | 73.47 | 3.16 | 98.03 → 123.60 |
| 2 | 5 | 332.78 | 23.54 | 35.26 | 38.36 | 6.12 | 121.75 → 157.52 |
| 4 | 5 | 635.42 | 11.94 | 33.46 | 50.27 | 4.09 | 168.73 → 224.34 |

All six scenarios completed without HTTP or health errors: **4,800 measured
application requests**. Each scenario verified **816 committed rows**, including
16 warmups, so successful responses correspond to actual database writes.

The 5 ms CPU workload scales from 164.95 requests/s with one worker to 635.42
with four (about 3.85×). Lightweight work sees smaller gains and additional
worker memory cost. The default two-worker pool remains a reasonable starting
point; choose pool/queue sizes using the application's own workload and memory
budget. These numbers are a short local baseline, not a production capacity
promise, long-running leak check or statistical confidence interval.

The first cancellation-enabled experiment exposed multi-second tail waits with
one worker. Host-owned FIFO admission removed that queue-order artifact: the
recorded one-worker 5 ms workload has a 73.47 ms maximum rather than the earlier
roughly 3 second outlier. The resilience suite independently checks FIFO order,
queue deadlines, replacement and shutdown; throughput alone is not the proof.

Reproduce from the API checkout with the current compiler:

```sh
npm run bench:workers
```

The command rebuilds its Lango fixtures/support and rewrites
`benchmarks/worker-baseline.json`. Repeat measurements before tuning a deployment;
client scheduling, SQLite contention, JIT/GC and host load affect results.
