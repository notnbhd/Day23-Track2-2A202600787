# Day 23 Lab Reflection

**Student:** Nguyen Bach Hai Dang  
**Submission date:** 2026-06-29  
**Lab repo URL:** https://github.com/notnbhd/Day23-Track2-2A202600787

---

## 1. Hardware + setup output

Output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.0)
Compose v2:    OK  (5.1.4)
RAM available: 14.98 GB (OK)
Ports free:    BOUND: [8000, 3000]  (occupied by other lab services; freed before make up)
Report written: 00-setup/setup-report.json
```

Setup report summary:

```json
{
  "docker": { "ok": true, "version": "29.4.0" },
  "compose_v2": { "ok": true, "version": "5.1.4" },
  "ram_gb_available": 14.98,
  "ram_ok": true,
  "required_ports": [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888],
  "bound_ports": [8000, 3000],
  "all_ports_free": false
}
```

Machine: Linux 6.19.11-arch1-1, AMD CPU, 16 GB RAM. The 7-service stack runs comfortably within the available memory; no OOM events were observed during the 60-second load test.

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels

The **AI Service Overview** dashboard (`day23-ai-overview`) contains six panels wired to live Prometheus data:

1. **Request Rate (RPS) by status** — `rate(inference_requests_total[1m])` split on `status` label
2. **Latency P50 / P95 / P99** — `histogram_quantile()` over `inference_latency_seconds_bucket`
3. **Error Rate (last 5m)** — stat panel with red/yellow/green thresholds at 1% and 5%
4. **Active Requests (gauge)** — `inference_active_gauge` live value
5. **Token Throughput** — `rate(inference_tokens_total)` split by `direction` (input/output)
6. **Quality Score** — `inference_quality_score` gauge by model

Screenshot: `submission/screenshots/dashboard-overview.png`

### Burn-rate panel

The **SLO Burn Rate** dashboard (`day23-slo`) implements the Google SRE Workbook multi-window multi-burn-rate pattern against a 99.5% availability SLO (error budget = 0.5%).

- **Error Budget Remaining %** stat panel with red below 25%, yellow below 50%
- **Burn rate (multiple windows)** timeseries showing 5m, 30m, 1h, 6h ratios relative to budget
- **SLOFastBurn** fires when both 5m and 1h windows exceed 14.4× the normal burn rate
- **SLOSlowBurn** fires when both 30m and 6h windows exceed 6× normal

Screenshot: `submission/screenshots/slo-burn-rate.png`

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | Stopped `day23-app` container (`docker stop day23-app`) | — |
| T0+60s | `ServiceDown` alert entered FIRING state in Alertmanager | `submission/screenshots/alertmanager-firing.png` |
| T0+90s | Slack `#oncall` received 🔥 fire message | `submission/screenshots/slack-firing.png` |
| T1 | Restored app (`docker start day23-app`) | — |
| T1+60s | Alert resolved; Slack `#oncall` received ✅ resolve message | `submission/screenshots/slack-resolved.png` |

The `send_resolved: true` flag in `alertmanager.yml` ensures the resolve notification is sent automatically once `up{job="inference-api"}` returns to 1.

### One thing surprised me about Prometheus / Grafana

The **multi-window burn-rate** alert logic surprised me most. Intuitively one might fire an alert on a single high error-rate window, but the AND of fast (5m+1h) and slow (30m+6h) windows dramatically reduces false positives caused by short spikes. A 30-second transient blip at 14× burn rate fires the 5m leg but not the 1h leg — so no page. This is the SRE Workbook's core insight: the cost of a missed page is lower than the cost of unnecessary on-call fatigue.

---

## 3. Track 03 — Tracing & Logs

### One trace from Jaeger

After running `make trace`, the Jaeger UI at `http://localhost:16686` shows a root span `POST /predict` with three child spans in sequence:

```
inference-api  POST /predict  (total ~35ms)
├── embed-text          ~5ms    gen_ai.* attributes
├── vector-search       ~10ms   k=5
└── generate-tokens     ~20ms   gen_ai.usage.input_tokens, gen_ai.usage.output_tokens
```

All spans carry the `deployment.environment=lab` resource attribute injected by the OTel Collector `resource` processor. Span attributes follow the OpenTelemetry GenAI semantic conventions (`gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reason`).

Screenshot: `submission/screenshots/jaeger-trace.png`

### Log line correlated to trace

Structured JSON log emitted by structlog on a successful prediction:

```json
{
  "event": "prediction served",
  "model": "llama3-mock",
  "input_tokens": 12,
  "output_tokens": 47,
  "quality": 0.8731,
  "duration_seconds": 0.0347,
  "trace_id": "e681fb85345d6a397707137335410e8c",
  "level": "info",
  "timestamp": "2026-06-29T10:15:42.381Z"
}
```

The `trace_id` field links directly to the Jaeger trace, enabling log-to-trace correlation in Grafana Explore (Loki → Jaeger via derived field).

### Tail-sampling math

OTel Collector policy: `decision_wait=30s`, three policies evaluated in order:
1. **keep-errors** — `status_code=ERROR` → 100% retention
2. **keep-slow** — `latency > 2000ms` → 100% retention  
3. **probabilistic-1pct** — remaining healthy traces → 1% retained

At 10 RPS (60-second load test, concurrency 10), with ~0.5% error rate:
- Error traces: 10 × 0.005 = 0.05 tps → **100% kept** = 0.05 tps retained
- Healthy+fast traces: 10 × 0.995 ≈ 9.95 tps → **1% kept** = ~0.0995 tps retained
- Overall retention: (0.05 + 0.0995) / 10 ≈ **1.5% of all traces stored**

This means for a forced-error request (`fail=true`), the trace is guaranteed to appear in Jaeger. A healthy request has a 99% chance of being dropped — verifiable by sending 200 healthy requests and confirming ~2 appear in Jaeger.

---

## 4. Track 04 — Drift Detection

### PSI scores

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

`prompt_length` (shifted from N(50,15) → N(85,20)) shows PSI=3.46, far above the 0.2 threshold — severe input distribution shift. `response_quality` (shifted from Beta(8,2) → Beta(2,6)) shows PSI=8.85, an extreme quality degradation. `embedding_norm` and `response_length` are stable (PSI < 0.02).

### Which test fits which feature?

| Feature | Best test in production | Reason |
|---|---|---|
| `prompt_length` | **PSI** | Continuous, approximately normal. PSI gives an intuitive "population stability" score familiar to credit-risk teams; threshold 0.1/0.2 maps directly to "monitor/act." Best for input features monitored by non-ML stakeholders. |
| `embedding_norm` | **KS (Kolmogorov-Smirnov)** | Stable, nearly Gaussian. KS is parameter-free and sensitive to *any* distributional difference (location or shape), making it the safest choice when you don't know how drift will manifest. Its p-value (0.13 here) correctly confirms no drift. |
| `response_length` | **KS** | Same rationale as `embedding_norm` — symmetric Gaussian, no strong prior on where shift will occur. KS p-value = 0.087, borderline but not drifted. |
| `response_quality` | **KL divergence** | Beta-distributed (bounded [0,1]), so PSI and KS can understate the magnitude of a shape change (from right-skewed to left-skewed). KL captures the *information-theoretic* distance between the reference and current PDFs, penalizing tails heavily — KL=13.5 here is enormous, correctly flagging a complete quality inversion. For multivariate embeddings, **MMD** (Maximum Mean Discrepancy) would be preferred over per-feature univariate tests. |

**Summary rule of thumb:**  
- **PSI**: interpretable for business-facing continuous inputs (lengths, costs, counts)  
- **KS**: safe default for continuous features with unknown shift pattern; returns a p-value  
- **KL**: captures shape/tail changes in bounded or multi-modal distributions  
- **MMD**: use when comparing full embedding vectors (avoids the curse of dimensionality of per-feature tests)

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose?

The **Day 18 (lakehouse / Spark)** metric would be hardest to expose in production. Spark's internal metrics API (`/metrics/json`) returns a flat JSON blob that requires a custom `jmx_exporter` or `spark-metrics-exporter` sidecar to convert into Prometheus-scrapeable format. Unlike Qdrant (Day 19) which ships a native `/metrics` Prometheus endpoint, or llama.cpp (Day 20) which exposes OTel-compatible metrics, Spark's telemetry was designed for JMX first and Prometheus second. Getting DAG-level granularity (not just executor-level) further requires patching the Airflow StatsD emitter (Day 17) to emit histogram-compatible metrics.

The cross-day dashboard (`full-stack-dashboard.json`) is provisioned with all 6 panels. Panels for Days 16–22 gracefully display "No Data (Day X not running)" when the respective services are not reachable — no panel breaks the dashboard layout.

Screenshot: `submission/screenshots/cross-day-dashboard.png`

---

## 6. The single change that mattered most

**Adding the `inference_active_gauge` Gauge as the fourth metric alongside the standard RED triple (requests, errors, duration) made the biggest practical difference in the stack.**

Without the active-gauge, the dashboard could tell you the *throughput* and *error rate* after the fact, but not *right now, how much load is the system under?* The Gauge — incremented at the start of each `predict()` call and decremented in the `finally` block — turns the service into something you can *feel* during a load test: you see it climb from 0 to ~8 concurrent requests, plateau, then drain back to 0 after locust stops. This transforms the "AI Service Overview" panel from a historical summary into an operational radar.

The deck's §2 (LLM-Native Signals / USE method) distinguishes *Utilization* (how saturated is the resource?) from *Saturation* (is work piling up?). The active gauge is exactly the Utilization signal for our inference thread pool. Without it you can only infer utilization post hoc from throughput × average latency (Little's Law), which introduces a lag and loses the instantaneous peak. With it, a sudden spike to 50 concurrent requests — even if P99 latency looks fine in the 5-minute window — immediately stands out as a saturation signal worth investigating before it becomes an SLO violation. In practice, this single gauge is what an on-call SRE would glance at first when a `ServiceDown` alert resolves but the system still feels slow.
