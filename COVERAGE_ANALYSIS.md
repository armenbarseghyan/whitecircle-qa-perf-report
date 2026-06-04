# Load / Performance — Coverage Analysis

> What the perf track covers, in what **state** (live · mock-validated · designed · blocked), with what
> **confidence** (FACT vs [ASSUMPTION]), and where the honest ceiling is. Companion to `PERF_REPORT.md`
> (numbers), `FINDINGS.md` (defects), `ARCHITECTURE.md` (infra), `MOCK_VALIDATION.md` (harness proof).

## Verdict — maturity **L4 / 5**
Comprehensive characterization of everything **observable on the shared free-tier SaaS**, on a clean 3-axis
harness, gated by **hermetic CI** with HTML reporting + regression + trend. The only uncovered rows are the ones
that physically require a **dedicated environment** (real server capacity, 1 h soak, real high-concurrency) — and
WhiteCircle is SaaS-only with no self-host and an undocumented free-tier rate limit, so that env is unobtainable
for a take-home. Those profiles are **coded, safety-guarded, and their executors validated end-to-end against a
synthetic target** — i.e. ready to produce real numbers the moment such an env exists. Reaching **L5** needs only
that env (+ server-side metric correlation), not more code.

## 1. Coverage by performance criterion

State legend: 🟢 **live** (reproduced on the shared API) · 🟡 **mock-validated** (executor+method proven on a
synthetic target, numbers ≠ WhiteCircle) · 🟠 **designed** (coded, needs a dedicated env) · 🔴 **blocked** (workspace gate).

| # | Criterion | Method | Target | State | Result | Confidence |
|---|---|---|---|:--:|---|:--:|
| 1 | Latency p50/p95/p99 | smoke + 1-VU curves | shared | 🟢 | warm p50 ~250 ms · p95 0.6–1.3 s · p99 1.3–1.7 s | FACT |
| 2 | Throughput / rate-limit knee | constant-arrival-rate sweep 2→8 RPS | shared | 🟢 | ≤4 RPS clean · 5 onset · 6 = 18 % · 8 = 44 % `429` | FACT |
| 3 | Error model / 429 semantics | 18-req burst + header inspect | shared | 🟢 | **no `Retry-After`/`RateLimit-*`**; recovery immediate | FACT |
| 4 | Latency vs payload size | size curve + size probe to 5 MB | shared | 🟢 | flat ≤10 k chars; ~linear >50 k (5 MB → 11.5 s) | FACT |
| 5 | Latency vs dialogue length | turns curve 1→50 | shared | 🟢 | flat (last-message-only confirmed) | FACT |
| 6 | Request-size limit enforced | growing-body probe | shared | 🟢 | **no `413` ≤5 MB** (resource-exhaustion gap) | FACT |
| 7 | Burst / spike recovery | parallel burst + post-burst probe | shared | 🟢 | immediate (short-window limiter, no lockout) | FACT |
| 8 | Read vs write path | POST→GET read-after-write | shared | 🟢 | GET ~0.28 s ≈ 2× faster than POST | FACT |
| 9 | Cold start | first-hit timing | shared | 🟢 | ~3 s first request; warm tail spikes 1.8–6 s | FACT |
| 10 | Heavy-profile executors run at scale | stress/spike/breakpoint vs stub | mock | 🟡 | all 3 executors OK; knee found at the stub's ~80 RPS | proof-of-harness |
| 11 | Server capacity **breakpoint** (real) | breakpoint (ramping-arrival-rate) | dedicated | 🟠 | needs env (limiter masks real capacity) | — |
| 12 | Soak / latency-memory drift (1 h) | soak (constant-VUs 1 h) | dedicated | 🟠 | needs env (time-only run) | — |
| 13 | High-concurrency spike (real) | spike 0→100 VU | dedicated | 🟠 | needs env (would DoS shared) | — |
| 14 | Image-content latency (text vs image) | image-parts probe | shared | 🔴 | **400 "Media uploads not allowed for this team"** | FACT (gate) |
| 15 | Distributed load generation | k6-operator / k6 Cloud | — | 🟠 | not needed at this scale; design noted | — |

## 2. Test-profile coverage

| Profile | Shape | Shared-safe? | Where it ran | State |
|---|---|:--:|---|:--:|
| **smoke** | 2 VU · 20 s | ✅ | shared (CI gate) + mock | 🟢 green gate |
| rate-limit sweep | const-arrival 2→8 RPS | ✅ (bounded) | shared | 🟢 |
| 1-VU curves (size/turns) | per-VU-iterations | ✅ (1 VU) | shared + CI(mock) | 🟢 |
| **load** | ramp→10 VU hold | ❌ on shared | (guarded) | 🟠 designed |
| **stress** | ramp 50→100 VU | ❌ on shared | mock | 🟡 executor validated |
| **spike** | 0→100→0 VU | ❌ on shared | mock | 🟡 executor validated |
| **breakpoint** | arrival-rate→N RPS | ❌ on shared | mock | 🟡 executor validated |
| **soak** | 10 VU · 1 h | ❌ on shared | — | 🟠 designed |

## 3. Coverage-state summary

- **🟢 Live on shared (9 criteria):** the full latency / throughput / error / scaling / limits / recovery surface
  that a free-tier integrator can actually observe. **100 % of the shared-observable space.**
- **🟡 Mock-validated (3 executors + knee methodology):** stress/spike/breakpoint run end-to-end and the
  saturation-finding method is proven (`MOCK_VALIDATION.md`). Numbers are synthetic, explicitly labelled.
- **🟠 Designed, dedicated-env-gated (3):** real capacity breakpoint, 1 h soak drift, real high-concurrency spike.
- **🔴 Blocked (1):** image-content latency — team feature gate, not a harness limitation.

## 4. Infrastructure coverage

| Layer | Built | State |
|---|---|:--:|
| Config-as-code | `config/` env + version matrix · SLOs (tagged) · workload model | ✅ |
| Library | `lib/` client (headers+version+429/5xx) · checks · metrics · payload | ✅ |
| Profiles + guard | `profiles/` shapes + `assertSafe()` (heavy-on-shared → 0 requests, verified) | ✅ |
| Synthetic target | `mock/server.py` capacity-limited stub | ✅ |
| Reporting | `report/` self-contained HTML (SVG) · MD · **regression gate** · **p95 trend** · **per-scenario compare** | ✅ |
| Tooling | k6 (primary) + Locust (parity, cross-validated offered-vs-accepted RPS) | ✅ |
| Docs | README · PERF_REPORT · FINDINGS · ARCHITECTURE · MOCK_VALIDATION · DEDICATED_RUNBOOK · this file | ✅ |

## 5. CI coverage (`.github/workflows/perf-smoke.yml`, hermetic)

| Job | Covers | Gate? |
|---|---|:--:|
| **validate** | `k6 inspect` all entries + `py_compile` (wiring/syntax) | ✅ blocks |
| **perf-mock** | mock → smoke (SLO) → curves → HTML report → **regression vs baseline** → p95 trend | ✅ blocks |
| pages | publish HTML report to GitHub Pages on `main` | best-effort |
| compare-mock | smoke vs breakpoint vs spike → comparison report | manual |
| smoke-live | real smoke vs shared via secret | manual |

CI is **deterministic + secret-free** (mock target) → every `load/**` PR is gated without touching the shared API.

## 6. Confidence inventory

- **FACT (reproduced live, 2026-06-04, region eu):** criteria 1–9 + 14, and findings F-PERF-1…7.
- **[ASSUMPTION] (every latency/throughput threshold):** WhiteCircle publishes **no SLO** (only
  circle-guard-bench's qualitative speed score) → all pass/fail is against industry-default thresholds, labelled
  in `slo.js`, `PERF_REPORT.md`, and the report. A documented SLO is a one-number change in `config/slo.js`.
- **Synthetic (not WhiteCircle):** all 🟡 mock numbers — stub + local machine only.

## 7. Gaps & blockers — with the reason each is uncovered

| Gap | Why uncovered | Unblock |
|---|---|---|
| Real capacity / breakpoint RPS | shared rate-limits at ~5 RPS (masks capacity); no DoS on shared | dedicated env or higher-quota key |
| Soak drift (1 h) | too long / too much traffic for the shared API | dedicated env |
| Real high-concurrency spike | would DoS the shared eval endpoint | dedicated env |
| Image-content latency | team feature gate (`400`) | enable media on a test team |
| Server-side root-cause correlation | client metrics only; no APM/CPU/queue visibility on shared | dedicated env + observability |
| Stable HTML report URL | GitHub Pages not yet enabled | Settings → Pages → GitHub Actions (one click) |

## 8. What raises coverage to L5
Exactly one thing: a **dedicated WhiteCircle environment** (or a higher-quota key + written load permission),
ideally with **server-side metrics** shared during the run. Then the three 🟠 rows produce real numbers via the
already-built, already-validated profiles (`run_dedicated.sh` + `DEDICATED_RUNBOOK.md`) — no new code. Everything
else is done and CI-gated.
