# Load-Testing Infrastructure — Architecture

Design for a full-fledged, maintainable, **safe** load-testing infrastructure for WhiteCircle `check-session`.
This is the *target*; §5 maps the migration from what already exists. Written to be extensible beyond one
endpoint and to catch perf regressions over time — not just to run a one-off test.

## 1. Goals & non-goals
- **Goals:** (a) characterize latency/throughput/errors; (b) find true capacity boundaries on a dedicated env;
  (c) gate CI on perf regressions; (d) make adding an endpoint or scenario *config + a small module*, not a rewrite;
  (e) make it impossible to accidentally DoS the shared/prod API.
- **Non-goals:** functional/contract/quality testing (owned by the backend & quality tracks); load on the shared
  eval endpoint (smoke-only, by policy).

## 2. Core principle — three orthogonal axes
The single most important decision. Amateur setups fuse "10 VUs → /endpoint → assert p95<500ms" into one script.
Separate them so any combination composes:

| Axis | Answers | Lives in | Examples |
|---|---|---|---|
| **Workload** | *what* requests | `config/workload.js` + `scenarios/` | endpoint mix, payload-size & turn distributions, content-type ratio, think-time, data |
| **Profile** | *how much / when* | `profiles/profiles.js` | smoke · load · stress · spike · soak · breakpoint (k6 executors) |
| **SLO** | *pass/fail* | `config/slo.js` | p95<…, p99<…, error<…, throughput≥… — each tagged `docs`\|`assumption` |

One workload definition then drives **smoke in CI** and **breakpoint on dedicated**; SLOs are defined once and
reused. This is the backbone everything else hangs off.

## 3. Pillars

**3.1 Config-as-code & environments.** One `config/environments.js`: each env = `{ baseUrl, authRef,
versionMatrix, deploymentId, isShared, maxVUs, maxRPS }`. Scripts pick via `ENV=smoke|dedicated|ci|local-mock`.
The **version-per-endpoint matrix** is encoded here (session=`2026-04-15`; REST=`2025-12-01`) so the
`2025-06-15` trap can't recur. Secrets only via env vars / gitignored `.env.*` / CI secret store — never in code.

**3.2 Library layer (DRY).** `lib/`: `payload.js` (builders), `client.js` (request wrappers: headers,
version-per-endpoint, tagging, **429-vs-5xx-vs-other classification**), `checks.js` (envelope/schema validators),
`metrics.js` (custom Trends/Counters). Scenarios stay thin; cross-cutting logic lives once.

**3.3 Scenarios (workload modules).** One module per logical flow: `session_check` (POST), `session_mixed`
(POST→GET read-after-write), `curve` (size/turn characterization), `ratelimit` (arrival-rate knee). New endpoint
= new module reusing `lib/`. Realistic **mix** (read/write ratio, payload distribution) lives in `config/workload.js`,
not hardcoded.

**3.4 Profiles (load shapes).** `profiles/profiles.js` exports the executor+stages per scenario class, decoupled
from workload. Includes **abort conditions** (k6 `abortOnFail` on a hard threshold) as a kill-switch.

**3.5 Metrics & tagging taxonomy.** Standard (latency p50/p95/p99, RPS, `http_req_failed`) + custom
(`server_rate_limited_429`, `server_errors_5xx`, per-endpoint sub-metrics, per-size buckets, business: flagged-rate).
Consistent tags `{env, scenario, profile, endpoint, size_bucket, content_type}` enable slicing in one query.

**3.6 Reporting & trend tracking.** Per run: JSON (machine) + MD/HTML (human) with **run metadata** (git sha, env,
timestamp, k6 version). Time-series via k6 → **Prometheus remote-write / InfluxDB → Grafana** for live + historical
dashboards. A **baseline store** + regression comparison: fail when p95 regresses > X % vs the stored baseline.

**3.7 CI/CD.** PR gate = smoke (shared-safe, fast, thresholds) → blocks merge on perf regression. Scheduled =
nightly load + weekly soak on the dedicated env (k6-operator / k6 Cloud). Results published as artifacts + trend
dashboard; regression → alert.

**3.8 Observability correlation (the piece most setups skip).** Client metrics (k6/Locust) show *symptoms*
(latency, errors). Root cause + **true capacity** need *server-side*: APM traces, CPU/mem/GC, connection-pool &
queue depth, DB latency, autoscaling events, rate-limiter internals. Inject correlation IDs / trace headers so a
k6 run maps to server traces. **For WhiteCircle specifically:** the ~5 RPS client-observed 429 is the *rate
limiter* — the real model-inference saturation point is hidden behind it and only visible with server metrics on
a dedicated env (no limiter / higher quota). Without this correlation, "breakpoint" just re-measures the limiter.

**3.9 Scale & topology.** One generator is CPU/network-bound → for high RPS use **distributed** (k6-operator on
k8s, or k6 Cloud; Locust master/workers). **Co-locate generators in the target's region** to separate app latency
from network RTT, and **monitor the generator itself** so you never measure your own saturation.

**3.10 Safety & governance.** Env guards (heavy profiles refuse `isShared` envs — already implemented), per-env
blast-radius caps (`maxVUs`/`maxRPS`), approvals for prod, cost caps for cloud runs, threshold kill-switch, and a
**benign-data rule** (only synthetic/labeled content — no real PII, no harmful payloads).

## 4. Directory layout (as implemented — Phase 1)
Entry points (`*_k6.js`) **stay at `load/` root** so every documented command, the CI gate, and the runbook keep
working; the logic is extracted into `config/` + `lib/` + `profiles/` and the entries are thin orchestrators.
```
load/
├─ config/
│  ├─ environments.js     # env profiles + version-per-endpoint matrix; isShared guard           ✅
│  ├─ slo.js              # SLO/threshold defs, each tagged docs|assumption                     ✅
│  └─ workload.js         # traffic model (msgLen, numMsgs, content, mix, think-time)           ✅
├─ lib/
│  ├─ payload.js          # benign payload builders                                             ✅
│  ├─ client.js           # postSession/getSession: headers + version + tagging + 429/5xx       ✅
│  ├─ checks.js           # envelope validators                                                 ✅
│  └─ metrics.js          # cross-cutting custom counters (server_429 / server_5xx)             ✅
├─ profiles/profiles.js   # load shapes (executors) + assertSafe() guard, decoupled             ✅
├─ mock/server.py         # capacity-limited stub target for harness validation (no shared API)  ✅
├─ check_session_k6.js    # ENTRY: session check (smoke + heavy profiles) — thin orchestrator   ✅
├─ curve_k6.js            # ENTRY: size/turn latency curve                                       ✅
├─ ratelimit_k6.js        # ENTRY: arrival-rate knee finder                                      ✅
├─ locustfile.py          # Locust parity / distributed                                         ✅
├─ run_dedicated.sh       # full-suite runner (dedicated env)                                    ✅
├─ report/                # build_report.py · check_regression.py · baseline.json · sample_report ✅
├─ ci/perf-smoke.yml      # hermetic mock CI gate (+ optional live) → .github/workflows/          ✅
├─ results/               # gitignored raw outputs (json/csv)
└─ ARCHITECTURE.md  DEDICATED_RUNBOOK.md  FINDINGS.md  PERF_REPORT.md  .env.dedicated(gitignored)
```
**Deferred (optional, Phase 2+):** relocating entries under `scenarios/` (with root shims so commands still
resolve) + `reports/`, `runners/`, `locust/` subdirs. Skipped now because it would churn the documented command
surface for zero functional gain — the layering (the actual best-practice win) is already in place above.

## 5. Migration status
| Item | Outcome | Status |
|---|---|---|
| `check_session_k6.js` (workload+profile+SLO fused) | thin orchestrator over `config/` + `lib/` + `profiles/` | ✅ done |
| `curve_k6.js`, `ratelimit_k6.js` | use `config/environments` + `lib/client` (no duplicated host/version) | ✅ done |
| inline headers/version/429-logic | centralized in `lib/client.js` (kills the version trap) | ✅ done |
| `.env` + ad-hoc env reads | `config/environments.js` env profiles + version-per-endpoint matrix | ✅ done |
| safety guard | `profiles/assertSafe()` (verified: heavy-on-shared → 0 requests) | ✅ done |
| SLOs scattered in thresholds | `config/slo.js`, each tagged `docs\|assumption` | ✅ done |
| stdout + JSON summaries | `report/build_report.py` → HTML + MD; `check_regression.py` baseline gate | ✅ done |
| manual runs | `ci/perf-smoke.yml` hermetic mock gate (+ optional live job) | ✅ done |
| client-only metrics | Prometheus/InfluxDB → Grafana + server correlation | ⏳ Phase 3 |

Behavior-preserving: every prior command still works (smoke re-run green, guard verified); we relayered, not rewrote.

## 6. Tooling decisions
- **k6 = primary.** Scriptable JS, first-class thresholds/metrics/executors, low overhead, CI-native, k6-operator/Cloud for scale.
- **Locust = parity / Python-native complex flows** (stateful user journeys), and when the team is Python-first. Kept for cross-validation (already caught the offered-vs-accepted-RPS difference).
- **Grafana + Prometheus/InfluxDB** for dashboards & trends; **k6 Cloud / k6-operator** for distributed generation.

## 7. Maturity roadmap
- **Phase 0 — done:** scripts, green smoke gate, shared-safe characterization (latency, rate-limit knee, payload limit, recovery, scaling), env safety guard, runbook.
- **Phase 1 — done:** refactored to the 3-axis layout — `config/` (environments·slo·workload) + `lib/` (client·checks·metrics·payload) + `profiles/`; entries are thin orchestrators. Pure structure & DRY, behavior-preserving (smoke green, guard verified).
- **Phase 2 — done:** `report/build_report.py` (HTML+MD report) + `check_regression.py` baseline gate (catches *and* passes — both demonstrated) + `ci/perf-smoke.yml` hermetic mock CI gate (+ optional live job). Sample: `report/sample_report.html`.
- **Phase 3 — observability & scale:** k6→Prometheus→Grafana, server-side metric correlation, distributed generation (k6-operator/Cloud), trend alerts.

## 8. Take-home scope vs production
For the take-home, **Phase 0–1 + a documented design of 2–3** is a complete, senior-level deliverable: the infra
is clean, safe, and extensible, and the report is honest about what ran vs what needs a dedicated env. Phases 2–3
are the production build-out, switched on once a dedicated env + observability stack exist.
