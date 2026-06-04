# Performance Report — `check-session` (`POST /api/session`)

**Track:** Load / Performance · **Date:** 2026-06-04 · **Region:** eu · **Tools:** k6 v2.0.0, Locust 2.34.0
**Endpoint:** `POST https://eu.whitecircle.ai/api/session` (+ `GET /api/session/{id}`) · **Version hdr:** `2026-04-15`

Latency is a **product feature**: `check-session` is the hot path of a runtime guardrail, and WhiteCircle's own
`circle-guard-bench` scores an *"integral score combining accuracy and speed"* — so speed is a graded dimension.
**No numeric latency SLO is published**, therefore every threshold below is tagged `[ASSUMPTION: industry default]`.

---

## 1. Honesty layer — what ran live vs designed

| | What |
|---|---|
| **Ran LIVE (shared, smoke discipline)** | version probe · k6 smoke (2 VU/20 s) · Locust smoke (2 u/20 s) · 1-VU latency curves (size + turns) · paced curls for image + mixed POST→GET. **Peak concurrency = 2 VU**; ~310 benign requests total; heaviest burst = the sanctioned 2-VU/20 s smoke. |
| **DESIGNED only (need a dedicated env + permission)** | load · stress · spike · soak · breakpoint (profiles coded in `check_session_k6.js`, **not** run on shared — would DoS / only measure the shared rate-limit, not capacity). |
| **Blocked on this workspace** | image-content latency (team feature-gated, F-PERF-6). |

⚠ **Safety:** the shared endpoint is **smoke-only**. load/stress/spike/soak/breakpoint were defined but **not executed**.

### Coverage matrix — load-testing criteria

| Criterion | Method | Shared-safe? | Status |
|---|---|---|---|
| Latency p50/p95/p99 | smoke + 1-VU curves | ✓ | ✅ done (§3, §4a/b) |
| Throughput ceiling / rate-limit knee | arrival-rate sweep 2→6 RPS | ✓ | ✅ done — ~4 RPS safe (§4c) |
| Error rate / 429 semantics | burst + header inspect | ✓ | ✅ done — no `Retry-After` (§4c) |
| Latency vs payload size | curve + size probe to 5 MB | ✓ | ✅ done — flat ≤10 k, linear >50 k (§4a, §4d) |
| Latency vs dialogue length | turns curve 1→50 | ✓ | ✅ done — flat (§4b) |
| Request/resource size limit | size probe to 5 MB | ✓ | ✅ done — **no cap** (§4d, F-PERF-7) |
| Burst spike recovery | 18-req burst + probe | ✓ | ✅ done — immediate (§4e) |
| Read vs write path | mixed POST→GET | ✓ | ✅ done — read ~2× faster (§3) |
| Cold start | first-hit timing | ✓ | ✅ done — ~3 s (§2) |
| **Server capacity breakpoint** | stress / breakpoint | ✗ on shared | 🔬 **executor validated** vs synthetic mock (`MOCK_VALIDATION.md`); real numbers → dedicated env (§6) |
| **Soak / latency-memory drift (1 h)** | soak | ✗ on shared | 🔬 designed → **dedicated env** (§6) |
| **High-concurrency spike** | spike 0→100 VU | ✗ on shared | 🔬 **executor validated** vs synthetic mock; real numbers → dedicated env (§6) |

Everything observable on the shared endpoint is characterized. The heavy profiles can't run on shared (DoS), so
their **executors are validated end-to-end against a controlled synthetic target** (`MOCK_VALIDATION.md` — the
methodology finds a saturation knee at the mock's modeled capacity); only the *real WhiteCircle numbers* await a
dedicated env. Soak drift is a time-only run on that env.

---

## 2. Headline results

- ✅ **Latency is good.** Warm p50 ~250–270 ms, p95 ~0.6–1.3 s, p99 ~1.3–1.7 s. **Flat** vs dialogue length (1→50
  turns) and vs text size for realistic inputs (≤10 k chars) — consistent with *last-message-only* evaluation.
- 🚩 **Rate-limit knee at ~5 RPS/key (F-PERF-2).** ≤4 RPS clean · 5 RPS onset (2 %) · 6 RPS 18 % · ~8 RPS 44 % `429`.
  Server-side; **no `Retry-After`** header; recovery immediate. Shared **eval** quota — not a product ceiling.
- 📦 **No payload-size cap (F-PERF-7).** Bodies up to **5 MB** accepted (no `413`); latency grows ~linearly above
  ~50 k chars (5 MB → 11.5 s) → a resource-exhaustion vector + the only real size-driven latency.
- 🐛 **Config bug (F-PERF-1).** `.env WHITECIRCLE_VERSION=2025-06-15` → `400 Unsupported API version`; must be `2026-04-15`.
- ❄ **Cold start ~3 s** on first hit; warm tail occasionally spikes to 1.8–6 s (model jitter).

---

## 3. Scenario × metrics

p50/p95/p99 in **ms**. "200-only" excludes the fast `429` rejections; "all-status" is every response.

| Scenario | Tool | Offered load | RPS (actual) | p50 | p95 | p99 | max | Error rate | Notes |
|---|---|---|---:|---:|---:|---:|---:|---:|---|
| Single request (warm) | curl | 1 seq | ~2–3 | ~270 | — | — | ~450 | 0 % | hand baseline |
| Single request (cold) | curl | 1st hit | — | — | — | — | **3074** | 0 % | cold start (first hit after idle) |
| **Smoke — CI gate (paced)** | k6 | 2 VU · 20 s · think-time 1 s | 1.36 | **248** | **1287** | **1656** | 1730 | **0 %** | ✅ all thresholds pass |
| **Smoke (paced)** | Locust | 2 u · 20 s | 1.34 | 270 | 570 | 690 | 694 | **0 %** | think-time paced |
| Rate-limit burst (200-only) | k6 | 2 VU · `SLEEP=0` | 8.2 | 264 | 932 | 1327 | 1930 | — | warm 200 latency under burst |
| Rate-limit burst (all-status) | k6 | 2 VU · `SLEEP=0` | 8.2 | 225 | 607 | 1225 | 1930 | **43.98 %** | 73× `429`, 0× 5xx |
| Size curve (1 VU) | k6 | 1 VU paced | 1.2 | ~270 | — | — | 1336 | 0 % | flat; see §4 |
| Turns curve (1 VU) | k6 | 1 VU paced | 1.1 | ~270 | — | — | 1864 | 0 % | flat; see §4 |
| Mixed POST→GET | curl | 1 seq | — | POST ~430 / **GET ~280** | — | — | 788 | 0 % | read ≈ 2× faster than write |

**Paced vs burst.** The CI-gate smoke carries 1 s think-time (2 VU ≈ 1.4 RPS, like real traffic) → green.
The **rate-limit burst** (`SLEEP=0`, back-to-back) offers 8.2 RPS → 44 % `429`. Locust (think-time paced to
1.34 RPS) independently confirms the green path. Together they **bracket the limit at ~4–5 RPS** (the burst let
~4.6 successful RPS through). Warm-latency tails are sample-sensitive at smoke N (28 samples): p95 ranges
~0.6–1.3 s across runs, p99 ~1.3–1.7 s — all within the `[ASSUMPTION]` thresholds.

---

## 4. Scaling & stability boundaries

Curves (4a/4b) swept at **1 VU sequentially** (peak concurrency 1 — gentler than the 2-VU smoke), paced ~0.5 s/req
to stay under the rate limit, 5 reps/bucket, **0 errors**. Raw: `load/results/curve_size.{json,md}`, `curve_turns.{json,md}`.

### 4a. Message length 100 → 10 000 chars
| chars | p50 ms | p95 ms | p99 ms | avg | min | max |
|---:|---:|---:|---:|---:|---:|---:|
| 100 | 258 | 625 | 670 | 366 | 244 | 681 |
| 500 | 302 | 529 | 563 | 333 | 211 | 571 |
| 1 000 | 238 | 348 | 356 | 271 | 226 | 358 |
| 2 000 | 325 | 1200 | 1309 | 576 | 240 | 1336 |
| 4 000 | 241 | 456 | 491 | 293 | 221 | 500 |
| 8 000 | 237 | 329 | 347 | 258 | 230 | 351 |
| 10 000 | 378 | 462 | 472 | 369 | 253 | 475 |

→ **Flat.** p50 stays ~237–378 ms across a 100× size range. Text length is **not** a latency driver.
(The 2 000-char p95/p99 spike is a single jitter sample, n=5 — not a size effect.)

### 4b. Dialogue length 1 → 50 turns
| turns | p50 ms | p95 ms | p99 ms | avg | min | max |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 293 | 598 | 649 | 356 | 238 | 662 |
| 2 | 251 | 658 | 685 | 392 | 243 | 692 |
| 5 | 262 | 1548 | 1801 | 582 | 244 | 1864 |
| 10 | 303 | 906 | 921 | 512 | 237 | 924 |
| 20 | 279 | 494 | 521 | 333 | 243 | 527 |
| 35 | 235 | 293 | 297 | 251 | 221 | 298 |
| 50 | 288 | 452 | 476 | 321 | 245 | 482 |

→ **Flat p50** (~235–303 ms). The p95/p99 blip at 5 turns is a single jitter sample (n=5, max 1.86 s), not a
trend. Confirms **last-message-only**: prior context turns add no measurable latency. (p95/p99 here are near-max
with only 5 samples — directional, not statistical; widen reps on a dedicated env for true tail.)

### 4c. Rate-limit knee (throughput boundary) — `ratelimit_k6.js`
Constant arrival rate, 10 s/step. Server `429` (k6 doesn't retry). Raw: `load/results/ratelimit_{2..6}rps.json`.

| offered RPS | 2 | 3 | 4 | 5 | 6 | 8.2 |
|---|---:|---:|---:|---:|---:|---:|
| **429 %** | 0 | 0 | 0 | **2 %** | **18 %** | **44 %** |
| accepted p50/p95 ms | 254 / 707 | 289 / —* | 249 / 611 | 256 / 521 | 239 / 500 | 264 / 932 |

→ **Sustained-safe ≈ 4 RPS/key; throttling onset at 5 RPS; ~half rejected by ~8 RPS.** Accepted-request latency
stays healthy through the knee. 429s carry **no `Retry-After` / `RateLimit-*`** header. *(3-RPS p95 6.0 s = one
stuck-tail sample during VU ramp — see 4e tail note.)*

### 4d. Payload-size boundary — input/resource limit
Single POSTs, growing body. **No `413`/`400` up to 5 MB** (largest tried); latency then ~linear (≈2 µs/char).

| content chars | 50 k | 200 k | 500 k | 1 M | 2 M | 5 M |
|---|---:|---:|---:|---:|---:|---:|
| body size | 50 KB | 200 KB | 500 KB | 1 MB | 2 MB | 5 MB |
| HTTP | 200 | 200 | 200 | 200 | 200 | 200 |
| latency | 0.70 s | 1.67 s | 2.34 s | 3.68 s | 6.28 s | **11.46 s** |

→ **No size guard** (resource-exhaustion / cost vector, F-PERF-7; AC-security expects a limit — absent ≤5 MB).
Normal chat (≤10 k) is flat (§4a); the linear cost only bites on multi-MB inputs.

### 4e. Burst recovery & tail
- **Recovery:** 18-request parallel burst → 13×`429` / 5×`200`; a single request **+1 s later → `200` in 0.45 s**.
  No lasting lockout → short-window (token-bucket-like) limiter.
- **Tail:** warm latency is usually ~250 ms but **occasionally spikes to 1.8–6 s** (model/backend jitter; seen in
  the turns curve and the 3-RPS step). p99 budgeting must assume a multi-second tail, not the p50.

---

## 5. Pass / fail vs AC-PERF

> **No documented latency SLO exists.** All thresholds are `[ASSUMPTION: industry default for a hot-path guardrail]`.
> The only doc signal is `circle-guard-bench`'s speed-weighted integral score (qualitative).

| AC | Threshold | Source | Result |
|---|---|---|---|
| AC-PERF-01 | p95 < 1500 ms | `[ASSUMPTION]` | ✅ **PASS** — warm p95 570–1287 ms (sample-sensitive at smoke N) |
| AC-PERF-02 | p99 < 3000 ms | `[ASSUMPTION]` | ✅ **PASS (warm)** — p99 ~0.7–1.66 s; ⚠ cold-start first request ~3.07 s sits at the edge |
| AC-PERF-03 | error rate < 1 % | `[ASSUMPTION]` | ✅ **PASS ≤4 RPS** (0 %) · ❌ **FAIL above knee** (5 RPS 2 % → 8 RPS 44 %). Failures are **rate-limit `429`**, not capacity 5xx (§4c) |
| AC-PERF-04 | latency stable vs size & dialogue | `[ASSUMPTION]` | ✅ **PASS** — flat ≤10 k chars & 1→50 turns; ⚠ ~linear >50 k (§4a/d) |
| AC-PERF-05 | throughput ≥ target @ p95 SLO | `[ASSUMPTION]` | 🔬 **DESIGNED** — shared caps at ~4 RPS knee; real ceiling needs dedicated env (stress/breakpoint) |
| AC-PERF-06 | no latency/memory drift over 1 h | `[ASSUMPTION]` | 🔬 **DESIGNED** — soak on dedicated env |
| AC-PERF-07 | request size limit enforced | `[ASSUMPTION]`/docs (AC-security) | ❌ **FAIL** — no `413`; bodies ≤5 MB accepted, 5 MB → 11.5 s (§4d, F-PERF-7) |

---

## 6. Designed profiles (run ONLY on a dedicated env + permission)

Coded in `load/check_session_k6.js` (`SCENARIO=<name>`). **Not executed on shared.**

| Scenario | Shape | Question it answers |
|---|---|---|
| load | ramp→10 VU, hold 1 m | p95/p99 + error rate at expected traffic |
| stress | ramp 50→100 VU | where do `429`/5xx start; throughput ceiling |
| spike | 0→100 VU in 10 s, hold, drop | recovery after a burst |
| soak | 10 VU · 1 h | latency / memory drift, leaks (AC-PERF-06) |
| breakpoint | arrival-rate 1→200 rps | absolute capacity (AC-PERF-05) |

On a dedicated env, **separate server `429` from client retries** (already done: `server_rate_limited_429`,
`server_errors_5xx` counters; neither k6 nor Locust retry by default), and find the RPS where `429` first appears.

---

## 7. Reproduce

```bash
set -a; source .env; set +a                                  # token + deployment_id (version is pinned in-script)
SCENARIO=smoke k6 run load/check_session_k6.js               # ✅ shared-safe (paced) → results/summary.json
SCENARIO=smoke SLEEP=0 k6 run load/check_session_k6.js       # rate-limit burst probe (2 VU back-to-back → 429s)
MODE=size  REPS=5 k6 run load/curve_k6.js                    # ✅ 1-VU size curve  → results/curve_size.md
MODE=turns REPS=5 k6 run load/curve_k6.js                    # ✅ 1-VU turns curve → results/curve_turns.md
.venv/bin/locust -f load/locustfile.py --headless -u 2 -r 1 -t 20s --host "$API_BASE_URL"   # ✅ parity
# parameterize the smoke payload:
SCENARIO=smoke MSG_LEN=2000 NUM_MSGS=10 MIX=postget k6 run load/check_session_k6.js
# dedicated env ONLY (never on shared):
# SCENARIO=stress k6 run load/check_session_k6.js
```

---

## 8. Recommendations & doc gaps

1. **Publish a latency SLO** (p50/p95/p99) and the **rate-limit policy** (RPS/key, burst, knee). Currently ~4 RPS
   with **no `Retry-After`/`RateLimit-*` headers** on 429 — add them so clients can back off deterministically (F-PERF-2).
2. **Enforce a request-size limit** (return `413` over N MB): bodies up to 5 MB are accepted today and a 5 MB body
   ties up a worker for ~11.5 s — a cheap resource-exhaustion/cost vector (F-PERF-7; AC-security gap).
3. **Fix `.env`** `WHITECIRCLE_VERSION` for session usage, or document version-per-endpoint clearly (F-PERF-1).
4. **Integrators on the hot path** should: stay ≤4 RPS/key (or request a higher quota) with client-side
   backoff/queueing for `429`; expect a ~3 s cold start and an occasional multi-second tail; keep request bodies
   bounded (≤10 k chars is flat; multi-MB is slow). Long dialogue histories are safe (no latency penalty).
5. **Run load/stress/spike/soak/breakpoint on a dedicated env** to get the true throughput ceiling and soak drift.
6. **Enable media** on a test team to characterize image-moderation latency (F-PERF-6).

*Findings detail: `load/FINDINGS.md`. Raw outputs: `load/results/`.*
