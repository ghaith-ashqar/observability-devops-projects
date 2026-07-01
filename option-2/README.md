# Kubernetes + Java Observability Proposal

An open-source observability proposal for a
Java (JRE21) application running on Kubernetes. It adds metrics and distributed tracing
to an environment that today has **logs only** (Promtail → Loki), stores everything
long-term in cheap S3-compatible object storage, and **replaces a risky nightly MongoDB
polling job** with real-time, application-exposed business metrics.

Built entirely on the **Grafana "LGTM" ecosystem** (Loki, Grafana, Tempo, Mimir) so it
extends the existing Loki setup instead of introducing a competing toolchain.

---

## What's in this repo

| Path | Contents |
|---|---|
| [`PROPOSAL.md`](./PROPOSAL.md) | The full technical proposal: architecture, tool selection & justification, phased implementation plan, the MongoDB-replacement deep dive, summary table, and risks/mitigations. |

---

## The problem, in one paragraph

The application runs on a 5–10 node / 30–50 pod cluster with **no metrics pipeline** (no
Prometheus/Thanos/Cortex/Mimir) and **no tracing** — only logs. Separately, a nightly
1:00 AM job connects directly to the production MongoDB and runs heavy aggregation queries
to extract business KPIs; this is the suspected cause of customer-facing latency spikes in
that window. We need infrastructure + JVM metrics, a non-intrusive replacement for the DB
job, >1 year of cost-efficient storage, distributed tracing, and a unified dashboarding
layer — all open-source.

## The solution at a glance

```
[App Cluster]                                   [Shared Infra Cluster]
  Spring Boot (Micrometer) ─┐  remote_write       ┌─ Grafana Mimir ─► S3 (metrics, >1yr)
  kube-state-metrics ───────┼─► Prometheus agent ─┤
  node-exporter / cAdvisor ─┘                      │
  OTel Java agent ──OTLP──► OTel Collector ───────┼─ Grafana Tempo ─► S3 (traces)
  Promtail (existing) ─────────────────────────── ┼─ Loki (existing) ─► S3 (logs)
                                                   │
                                          Grafana (Mimir + Loki + Tempo) ─► engineers
```

| Goal | Chosen tool | Why |
|---|---|---|
| K8s / infra metrics | **kube-prometheus-stack** (Prometheus agent, kube-state-metrics, node-exporter) | De-facto standard; curated rules & ServiceMonitors; agent mode keeps the in-cluster footprint small |
| JVM / app metrics | **Micrometer** (Prometheus registry) | Native JVM instrumentation; one library for technical *and* business metrics |
| Long-term storage | **Grafana Mimir** + S3 object storage | Cheap multi-year retention; simpler than Thanos; successor to Cortex; same family as Loki |
| Visualization | **Grafana** | Single pane over metrics + logs + traces with built-in correlation |
| Tracing | **OpenTelemetry Java agent** → **Grafana Tempo** | Zero-code auto-instrumentation; Tempo stores traces in the same object storage |
| Business metrics (DB job replacement) | **Micrometer counters/gauges** in app code | Event-driven, in-memory — zero load on MongoDB, near-real-time vs. once-nightly |

Full reasoning and the phased rollout are in **[`PROPOSAL.md`](./PROPOSAL.md)**.

## Why the MongoDB job goes away

Instead of scanning MongoDB nightly to reconstruct KPIs, the app increments an in-memory
Micrometer counter **at the moment each business event happens** (an order completes, a
user logs in). Those meters are exposed on the same `/actuator/prometheus` endpoint that
already serves JVM metrics, scraped every 30s, and stored in Mimir for >1 year. There is
**no extra DB query, no aggregation pipeline, and no schema coupling** — so the 1:00 AM
resource contention disappears, and KPIs become real-time. Daily/weekly rollups are
computed cheaply at query time (`increase(business_orders_total[1d])`) or via Mimir
recording rules. See the deep dive in [`PROPOSAL.md` §5](./PROPOSAL.md#5-database-polling-replacement--deep-dive).
