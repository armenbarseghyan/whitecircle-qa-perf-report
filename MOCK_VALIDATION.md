# Harness Validation — heavy profiles run end-to-end + breakpoint methodology demonstrated

> ⚠ **Synthetic target, NOT WhiteCircle.** Run against `load/mock/server.py` (a local capacity-limited stub),
> because the heavy profiles must never touch the shared eval API and no dedicated WhiteCircle env is obtainable
> (SaaS-only, no self-host). **The numbers below reflect the stub + this laptop — they say nothing about
> WhiteCircle's capacity.** What they *do* prove: the load harness runs `stress/spike/breakpoint`-class profiles
> correctly at scale, and the methodology finds a saturation knee — i.e. the exact procedure we'd point at a real
> dedicated env the moment one exists.

## The controlled target
`load/mock/server.py`: a fixed pool of `CAP` worker slots + a bounded queue; each request holds a slot for a
synthetic service time (base + jitter); past `QUEUE_MAX` it sheds load with `503`. Run here with `CAP=6,
base=50ms` → **modeled capacity ≈ 80 RPS**. So there is a real knee to discover.

## 1. Knee sweep — `constant-arrival-rate`, 8 s/step (the "find the limit" test)
| offered RPS | 30 | 50 | 70 | 90 | 110 |
|---|---:|---:|---:|---:|---:|
| 200 | 240 | 401 | 561 | 648 | 652 |
| 503 (shed) | 0 | 0 | 0 | 73 | 228 |
| **error %** | 0 | 0 | 0 | **10.1 %** | **25.9 %** |
| p50 ms | 82 | 80 | 82 | **568** | 570 |
| p95 ms | 101 | 101 | 104 | **625** | 631 |

→ **Knee ≈ 80 RPS** (== the modeled capacity). Below it: flat ~80 ms, 0 errors. At/above it: latency cliff (~7×,
queue fills) **then** load-shedding `503`s climb. This is the classic throughput-vs-latency saturation signature —
discovered, not assumed.

## 2. `breakpoint` executor end-to-end — `ramping-arrival-rate` 1→160 RPS / 40 s
3210 requests @ ~79 RPS · **26.4 % `5xx`** (846, via `server_errors_5xx`) · iteration p95 630 ms. The ramp crossed
the ~80 RPS knee and shed ~26 % once past capacity → the breakpoint executor works and locates the break.

## 3. `spike` executor end-to-end — `ramping-vus` 0→100→0, no think-time
~399 k requests @ ~9 958 RPS offered · **99.2 % `5xx`** load-shed · the target **stayed up** (fast 503s, no crash).
Validates the spike executor and the target's load-shedding under an extreme burst.

## What this validates (and what it doesn't)
- ✅ All three k6 executor classes — `constant-arrival-rate`, `ramping-arrival-rate`, `ramping-vus` — run
  correctly at scale and emit the right metrics (`server_errors_5xx`, latency percentiles, per-rate breakdown).
- ✅ The **methodology finds a saturation knee** (latency cliff + error onset) — the procedure for a real env.
- ✅ The safety guard correctly *allows* a non-shared host (the mock) while still refusing shared.
- ❌ Says nothing about WhiteCircle's real capacity — that needs a dedicated WhiteCircle env (see `DEDICATED_RUNBOOK.md`).

## Reproduce
```bash
MOCK_CAP=6 MOCK_BASE_MS=50 .venv/bin/python load/mock/server.py &     # controlled target on :8787
BASE=http://127.0.0.1:8787/api
for R in 30 50 70 90 110; do API_BASE_URL=$BASE RATE=$R DURATION=8s PREVUS=60 MAXVUS=160 k6 run load/ratelimit_k6.js; done
API_BASE_URL=$BASE SCENARIO=breakpoint SLEEP=0 BP_DUR=40s BP_TARGET=160 BP_PREVUS=40 BP_MAXVUS=220 k6 run load/check_session_k6.js
API_BASE_URL=$BASE SCENARIO=spike SLEEP=0 k6 run load/check_session_k6.js
kill %1   # stop the mock
```
