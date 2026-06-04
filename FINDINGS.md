# Load / Performance findings (this track)

Logged from live smoke + 1-VU characterization on the **shared** endpoint (2026-06-04, region eu).
All content benign. Peak concurrency = 2 VU (the sanctioned smoke). FACT = observed live; [ASSUMPTION] = no doc.
These feed `deliverables/Q2_BUGS.md` + the perf section of the report (`load/PERF_REPORT.md`).

Baseline from discovery (single-request, not load): 200 ≈ 0.4–1.3 s. No documented latency SLO.

---

## F-PERF-1 — `.env WHITECIRCLE_VERSION=2025-06-15` is wrong for `/api/session` (→ 400) [FACT]
- **Severity:** P2 · **Priority:** P2 · **Type:** Config / Doc drift
- **Scenario:** version probe (3 sequential requests)
- **Observed:** `whitecircle-version: 2025-06-15` (the value shipped in `.env`) → **HTTP 400 `"Unsupported API version"`**.
  `2026-04-15` → 200 (0.45 s); omitting the header → 200 (3.07 s, cold).
- **Impact:** any load/test script that reuses `$WHITECIRCLE_VERSION` for the session endpoint fails **100%**.
  `2025-06-15` is the CLI/legacy-drift version (V9), valid for `/api/protect/check`, **not** for `/api/session`.
- **Fix used here:** scripts pin `SESSION_VERSION=2026-04-15` (override-able), independent of `WHITECIRCLE_VERSION`.
- **Evidence:** version probe in session log; `load/check_session_k6.js` header block.

## F-PERF-2 — Rate-limit knee at ~5 RPS/key; 429s carry no `Retry-After` [FACT]
- **Severity:** P2 · **Priority:** P1 (must characterize before any prod sizing) · **Type:** Performance / Capacity
- **Scenario:** constant arrival-rate sweep (10 s/step) + 2-VU burst, all server-side (k6/Locust don't retry).
- **Knee (precise):**

  | offered RPS | 2 | 3 | 4 | 5 | 6 | 8.2 |
  |---|---|---|---|---|---|---|
  | 429 % | 0 | 0 | 0 | **2 %** | **18 %** | **44 %** |

  → **sustained-safe ≈ 4 RPS/key; throttling onset at 5 RPS; ~half rejected by ~8 RPS.** Latency on the 200s stays
  healthy through the knee (p50 ~240–290 ms, p95 ~500–700 ms).
- **429 semantics (gap):** the 429 response has **no `Retry-After` and no `RateLimit-*`/`X-RateLimit-*` headers** —
  clients get no machine-readable backoff hint and must guess. **Recovery is immediate** (single request +1 s after
  an 18-req burst → 200 in 0.45 s) → looks like a short-window/token-bucket limiter, no lasting lockout.
- **Impact:** a hot-path integrator exceeding ~4–5 RPS/key gets throttled with no `Retry-After`; needs client-side
  backoff/queueing or a higher quota. **AC-PERF error-rate (<1%) cannot hold above ~4 RPS on shared.**
- **Caveat (not a defect):** this is the **shared eval** quota, *not* a product capacity ceiling. The server's true
  saturation breakpoint is gated by this limiter → needs a **dedicated env** (designed; not run here).
- **Evidence:** `load/results/ratelimit_{2..6}rps.json`, `summary_smoke.json` (`SLEEP=0` burst), `locust_smoke_stats.csv`.

## F-PERF-3 — Latency is flat vs payload size and dialogue length [FACT, positive]
- **Severity:** info · **Type:** Performance characterization
- **Scenario:** 1-VU paced curve (5 reps/bucket, 0 errors)
- **Observed:** message length **100→10 000 chars** → p50 ~238–374 ms (no trend). Dialogue **1→50 turns** →
  p50 ~235–303 ms (no trend). Variation is jitter, not size-driven.
- **Interpretation:** consistent with documented **last-message-only** evaluation + model-inference cost dominating.
  Long multi-turn sessions are **not** a latency risk. Flat holds for realistic text sizes (**≤10 k chars**);
  beyond ~50 k it grows roughly linearly — see **F-PERF-7**.
- **Evidence:** `load/results/curve_size.{json,md}`, `load/results/curve_turns.{json,md}`.

## F-PERF-4 — Cold-start ~3 s on first request; warm p50 ~250–300 ms, p99 ~1.3 s [FACT]
- **Severity:** P3 · **Type:** Performance
- **Observed:** first request after idle ≈ **3.07 s** (TLS + connection + warmup); warm 200s p50 264 ms /
  p95 932 ms / p99 1 327 ms (k6 smoke). Occasional warm tail spikes to ~1.8 s (model jitter).
- **Impact:** low-traffic / serverless integrators pay the cold-start tail on first hit; keep-alive / warmers help.

## F-PERF-5 — `GET /api/session/{internal_session_id}` needs `deployment_id` for an All-scoped key [FACT]
- **Severity:** P3 · **Type:** Contract / Doc nuance
- **Observed:** GET by internal id with no `deployment_id` → **400 `"Missing required deployment_id"`** (any
  version header). With `?deployment_id=<id>` → **200** (~0.28 s). CONTEXT/02 recorded a bare 200 → reconciled:
  200 holds **only** when deployment_id is supplied (same All-scoped-key rule as POST). Not edited in CONTEXT.
- **Perf note:** read path (~0.28 s) is ~2× faster than the POST write path (~0.5–0.8 s) — GET is a cheap lookup.

## F-PERF-6 — Image content blocked at team level (designed-only here) [FACT]
- **Severity:** info · **Type:** Feature gate
- **Observed:** text+image parts → **400 `"Media uploads are not allowed for this team."`** → image-moderation
  latency (text-vs-image comparison) is **designed-only** on this workspace; enable the feature to measure.

## F-PERF-7 — No request-size limit (≥5 MB accepted); latency grows linearly with large payloads [FACT]
- **Severity:** P2 · **Priority:** P2 · **Type:** Reliability / Security (resource exhaustion) + Performance
- **Scenario:** single sequential POSTs, content length 50 k → 5 M chars.
- **Observed:** **all 200, no `413`/`400`** up to **5 MB** body (the largest tried). Latency scales ~linearly with
  size above ~50 k (≈2 µs/char): 50 k→0.70 s · 200 k→1.67 s · 500 k→2.34 s · 1 M→3.68 s · 2 M→6.28 s · **5 M→11.46 s**.
- **Impact:** (a) **no size guard** → a buggy/malicious client can submit multi-MB bodies and tie up a worker for
  10 s+ each → a cheap amplification/DoS & cost vector (CONTEXT AC-security expects "size limits enforced" — they
  are **not**, ≤5 MB; doc/impl gap). (b) Real latency-vs-size cost only appears for very large inputs; normal
  chat traffic (≤10 k) is flat (F-PERF-3).
- **Evidence:** payload-size probe in session log; cross-checks `flagged:false` 200 envelopes returned each time.
```
Title: [Perf] short symptom
Severity / Priority / Scenario (VUs/RPS) / Observed (p50/p95/p99, error rate, first 429) /
Expected threshold (from docs / [ASSUMPTION]) / Evidence (results file) / Notes
```
