# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** _Hồ Sỹ Minh Hà_
**Submission date:** _2026-05-11_
**Lab repo URL:** _https://github.com/cafesuada24/2A202600060-HoSyMinhHa-Day23/_

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.2)
Compose v2:    OK  (5.1.3)
RAM available: 15.31 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: /home/serein/SourceCodes/Day23/00-setup/setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

I was surprised by the strict distinction between metrics and logs regarding cardinality. While it's tempting to put a `trace_id` as a label in Prometheus to see exactly which request caused a spike, doing so would cause a "cardinality explosion" and crash the database. Learning to use metrics for the "What" (aggregate rates) and Traces/Logs for the "Why" (individual request details) was a key architectural takeaway.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```
{"model": "llama3-mock", "input_tokens": 8, "output_tokens": 18, "quality": 0.873, "duration_seconds": 0.0679, "trace_id": "7e2f9c80c392843b3e05b873c30c7aa5", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T08:27:58.867907Z"}
```

### Tail-sampling math

The collector uses a composite policy: 100% of errors, 100% of slow traces (>2s), and 1% of healthy traces.
Formula: `sampled = N × (P(error) × 1.0 + P(slow) × 1.0 + P(healthy) × 0.01)`
Assuming 1% error rate and 1% slow requests:
`sampled = N × (0.01 + 0.01 + 0.98 × 0.01) = N × 0.0298`.
This means the policy retains approximately **3%** of all generated traces, providing a 97% reduction in storage costs while losing zero high-value (error/slow) data.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

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

### Which test fits which feature?

- **prompt_length**: **KS Test**. It's non-parametric and sensitive to shifts in the distribution shape (like users switching from short to long prompts) without requiring us to pre-define bins.
- **embedding_norm**: **PSI**. Ideal for numerical stability checks. It provides a single stable score that is easy to dashboard and set alerts on (e.g., > 0.2 indicates significant drift).
- **response_length**: **KL Divergence**. Useful for measuring information loss. Since LLM response lengths are often log-normal, KL helps quantify how much "surprise" is in the new distribution compared to the baseline.
- **response_quality**: **MMD (Maximum Mean Discrepancy)**. For evaluations like quality scores which can be noisy, MMD is robust because it compares the distributions in a high-dimensional kernel space rather than just comparing means.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The model-serving metrics from Day 20 (Llama.cpp) would be the hardest. Unlike our FastAPI app where we have direct source-code access for OTel SDK integration, Llama.cpp usually requires a sidecar exporter to translate its custom metrics endpoint into Prometheus-compatible format, adding complexity to the scrape configuration.

---

## 6. The single change that mattered most

The single most impactful change was fixing the OpenTelemetry middleware registration order. Initially, the code attempted to instrument the FastAPI app inside the `lifespan` manager, which triggered a `RuntimeError` because middleware cannot be added once an application has started. By moving `setup_otel(app)` to the module level immediately after instantiation, I ensured that every single request—including the very first health checks—was properly captured in the trace context.

This fix was the "keystone" for the entire observability stack. Without it, the correlation between structured logs and traces would be broken; logs would carry a `trace_id` that never appeared in Jaeger. Connecting these signals is what transforms simple "monitoring" (knowing a service is down) into "observability" (being able to traverse from a high-error-rate metric to a specific failed trace and see exactly which child span, like `generate-tokens`, was responsible).
