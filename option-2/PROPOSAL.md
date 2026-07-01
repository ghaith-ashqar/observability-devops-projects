# Observability Proposal: Metrics, Tracing, and Long-Term Storage for the Java/Kubernetes Platform

## 1. Executive Summary

Our core Java application runs on Kubernetes today with only logging in place; we have no metrics and no distributed tracing, which leaves us blind to infrastructure health, application performance, and the root cause of incidents. Compounding this, a nightly job queries the production MongoDB directly to extract business KPIs, and it is the likely cause of the latency spikes our customers experience around 1:00 AM. This proposal recommends building a complete open-source observability stack — Prometheus for collection, **Grafana Mimir** for cost-efficient long-term storage on S3-compatible object storage, **Grafana Tempo** for tracing, and **Grafana** as the single pane of glass alongside our existing Loki logs. It also replaces the risky nightly database job with lightweight, application-exposed business metrics, eliminating the load on MongoDB and the associated customer impact. The result is full visibility across metrics, logs, and traces, more than a year of retained history at low cost, and the removal of a known reliability risk — all built on tools the team can realistically operate.

---

## 2. Architecture Overview

The proposed stack is built around the **Grafana "LGTM" ecosystem** (Loki, Grafana, Tempo, Mimir) deliberately, because we already run Loki and Promtail. Extending into the same family means one query layer, one dashboarding tool, one set of operational patterns, and native cross-correlation between logs, metrics, and traces.

The data flow works as follows. Inside the application cluster, a **Prometheus agent (or Grafana Agent / Alloy in agent mode)** is responsible for *collection only*. It scrapes three categories of targets: (1) infrastructure metrics from **kube-state-metrics** and **node-exporter**, deployed via the `kube-prometheus-stack` Helm chart; (2) JVM and application metrics exposed by the Java app through **Micrometer's Prometheus endpoint**; and (3) the new business-KPI metrics exposed by the application itself. Rather than storing data locally, the agent uses **remote_write** to ship all scraped samples out of the application cluster to **Grafana Mimir** running on the shared infrastructure cluster.

Mimir is the metrics storage and query engine. It buffers recent data and continuously flushes compacted blocks to **S3-compatible object storage**, where data lives cheaply for well over a year. Mimir exposes a Prometheus-compatible query API that Grafana uses as a datasource.

For tracing, the Java application is instrumented with the **OpenTelemetry Java agent**, which auto-instruments common frameworks (HTTP servers, JDBC/Mongo drivers, messaging). Spans are exported via OTLP to an **OpenTelemetry Collector**, which forwards them to **Grafana Tempo**. Tempo, like Mimir and Loki, stores its data in the same object storage backend — keeping the storage model uniform and cheap.

**Grafana** sits on top of everything as the unified visualization layer, with four datasources configured: Mimir (metrics), Loki (logs), Tempo (traces), and the ability to pivot between them — e.g., jump from a latency spike on a metrics dashboard, to the traces during that window, to the logs for a specific failing request, using trace and label correlation.

Textual component flow:

```
[App Cluster]
  Java App (Micrometer /actuator/prometheus, business metrics)
  Java App (OTel Java agent) ──OTLP──► OTel Collector ──► (remote)
  kube-state-metrics ─┐
  node-exporter ──────┤
  cAdvisor (kubelet) ─┴─► Prometheus Agent ──remote_write──► (remote)
  Promtail (existing) ──────────────────────────────────────► Loki

                                  │ (network to shared infra cluster)
                                  ▼
[Shared Infrastructure Cluster]
  Grafana Mimir  ◄── metrics      ──► S3 object storage
  Grafana Tempo  ◄── traces       ──► S3 object storage
  Loki (existing)◄── logs         ──► S3 object storage
        ▲ ▲ ▲
        └─┴─┴──── Grafana (Mimir + Loki + Tempo datasources) ──► Engineers
```

---

## 3. Selected Tools and Justification

**Kubernetes and infrastructure metrics collection — `kube-prometheus-stack` (Prometheus + kube-state-metrics + node-exporter).**
This Helm chart is the de facto standard for Kubernetes metrics and bundles node-exporter (host CPU/memory/disk/network), kube-state-metrics (object-level state: deployments, pods, restarts), and cAdvisor scraping (per-container usage) with sane default dashboards and alerts. We deploy Prometheus in *agent mode* rather than as a full TSDB, because storage and querying are delegated to Mimir; this keeps the in-cluster footprint small. It is chosen over a DIY exporter setup because the curated rules and ServiceMonitor CRDs save weeks of work for a small team.

**JVM/application metrics exposure — Micrometer (with the Prometheus registry).**
For JRE21 applications (especially Spring Boot, but Micrometer works standalone too), Micrometer is the native instrumentation facade and exposes heap, GC pauses, thread counts, class loading, and HTTP server metrics out of the box at a `/actuator/prometheus` endpoint. It is preferred over the standalone JMX Exporter because it requires no sidecar or javaagent for JMX scraping, produces clean Prometheus-formatted metrics directly, and is the same library we will use to expose custom business metrics — one instrumentation approach for both technical and business KPIs.

**Long-term metrics storage backend — Grafana Mimir.**
Mimir stores metrics as compacted blocks in S3-compatible object storage, which makes multi-year retention dramatically cheaper than block-storage-backed Prometheus, and it horizontally scales to tens of millions of active series. We choose it over Thanos because Mimir offers a simpler, more integrated deployment (a single Helm chart, monolithic or microservices mode) with better out-of-the-box multitenancy and compaction, and over Cortex because Mimir is its actively maintained successor. Crucially it belongs to the same Grafana family as our existing Loki, so it reuses the same object-storage and operational mental model.

**Visualization and dashboarding — Grafana.**
Grafana is the only realistic choice: it is open-source, already implied by our Loki deployment, and is the native query/visualization front-end for Mimir, Tempo, and Loki simultaneously. Its built-in correlation features (exemplars linking metrics to traces, "Logs to Trace" and "Trace to Logs" links) let an engineer move between the three signals without switching tools, which is the core value of consolidating on this ecosystem.

**Distributed tracing — OpenTelemetry (instrumentation) + Grafana Tempo (backend).**
The OpenTelemetry Java agent provides vendor-neutral, zero-code auto-instrumentation for HTTP, JDBC, and the MongoDB driver, so we get request-level traces without rewriting the application. Tempo is the backend because it stores traces directly in the same S3 object storage (no Elasticsearch/Cassandra to operate, unlike Jaeger's heavier backends), making it both cheaper and operationally lighter for a small team. Tempo integrates natively with Grafana and supports trace-to-logs/metrics correlation.

**Business metrics exposure (MongoDB replacement) — application-side Micrometer counters/gauges, scraped by Prometheus.**
Instead of an external job querying MongoDB, the application increments business counters as events naturally occur (a transaction completes, a user logs in) and exposes them on its existing Prometheus endpoint. This is non-intrusive by construction — it adds no query load to MongoDB — and is detailed in Section 5.

---

## 4. Implementation Plan

### Phase 1 — Metrics Foundation
Deploy the `kube-prometheus-stack` Helm chart into the application cluster, but configure Prometheus in **agent mode** (`prometheus.prometheusSpec` with remote-write only, minimal local retention such as 2h). This brings up node-exporter, kube-state-metrics, and cAdvisor scraping immediately. In parallel, enable Micrometer's Prometheus registry in the Java app and expose `/actuator/prometheus`; add a `ServiceMonitor` (or `PodMonitor`) so Prometheus discovers and scrapes the JVM endpoint.
*Gotchas:* Set scrape intervals to 30s (not 15s) to control cardinality and cost; ensure the app's metrics endpoint is on a separate management port not exposed publicly; watch for high-cardinality labels (avoid putting user IDs or request IDs as metric labels).

### Phase 2 — Long-Term Storage
Deploy **Grafana Mimir** on the shared infrastructure cluster (the same place Loki lives), pointed at an S3-compatible bucket. Start with the monolithic deployment mode for simplicity at our scale (30–50 pods is small for Mimir), with the option to split into microservices later. Configure the Phase-1 Prometheus agent's `remote_write` to target Mimir's distributor endpoint. Set the compactor's retention to **at least 400 days** (or longer per the >1-year requirement) and configure lifecycle/retention on the bucket.
*Gotchas:* Provision Mimir's ingesters with adequate memory and persistent volumes for the write-ahead log; secure the remote_write path (auth token / mTLS) since it crosses cluster boundaries; set a sensible `X-Scope-OrgID` tenant header even if single-tenant.

### Phase 3 — Distributed Tracing
Deploy the **OpenTelemetry Collector** (gateway mode) and **Grafana Tempo** (object-storage backend) on the infrastructure cluster. Attach the **OpenTelemetry Java agent** to the application via `JAVA_TOOL_OPTIONS=-javaagent:/otel/opentelemetry-javaagent.jar` and configure the OTLP exporter endpoint to the collector. Begin with **probabilistic sampling** (e.g., 10%) to manage volume, plus tail-based sampling later for errors/slow traces.
*Gotchas:* The OTel agent adds minor startup and per-request overhead — validate in staging; ensure trace context propagation is enabled across service boundaries; configure exemplars in Micrometer so metrics can link to traces.

### Phase 4 — Business Metrics (MongoDB Job Replacement)
Instrument business events in the application with Micrometer counters/gauges/timers (see Section 5). Expose them on the same Prometheus endpoint already scraped in Phase 1. Build Mimir **recording rules** for the daily/aggregate KPIs that the nightly job used to compute. Run the new metrics in parallel with the old nightly job for one to two weeks, validate the numbers match, then **decommission the nightly MongoDB job**.
*Gotchas:* Counters reset on pod restart — always query with `increase()`/`rate()` and `sum` across pods, never raw counter values; for KPIs that are inherently snapshot/stateful (e.g., total registered users), see the strategy in Section 5.

### Phase 5 — Visualization and Alerting
In Grafana, add **Mimir** and **Tempo** as datasources (Loki already exists). Import the standard kube-prometheus and JVM (Micrometer) dashboards, build a dedicated **Business KPI dashboard** from the new metrics, and configure trace-to-logs and metrics-to-traces correlations. Define alerting rules (via Mimir's ruler or Grafana alerting) for infrastructure saturation, JVM health (GC pressure, heap), SLOs, and notably an alert for the latency window we previously suspected.
*Gotchas:* Use Grafana provisioning (datasources and dashboards as code) so the setup is reproducible; lock down Grafana with SSO/RBAC.

---

## 5. Database Polling Replacement — Deep Dive

### The problem with the current approach
The nightly job opens a direct connection to the production MongoDB and runs heavy aggregation pipelines (counts, sums, daily rollups) over large collections at 1:00 AM. These aggregations compete with live customer traffic for the same database resources — working set memory, IOPS, CPU — which is the most plausible explanation for the observed latency spikes. The approach is also fragile: it couples an external job to the database schema, it produces KPIs only once per day with no real-time visibility, and any change to collections risks breaking it.

### Recommended approach: expose business metrics from the application itself
The correct pattern is to move measurement to **where the events happen** — inside the application — rather than reconstructing them after the fact by scanning the database. When a business event occurs (a transaction is processed, a user logs in, an order is placed), the application already knows about it in the normal course of handling the request. We attach a Micrometer instrument at that point:

```java
// Counter: incremented at the moment a transaction completes
Counter transactions = Counter.builder("business_transactions_total")
    .description("Total business transactions processed")
    .tag("type", "payment")
    .register(meterRegistry);
// ... in the transaction handler:
transactions.increment();

// Timer / amount tracking for transaction totals
DistributionSummary amount = DistributionSummary.builder("business_transaction_amount")
    .baseUnit("currency")
    .register(meterRegistry);
amount.record(transaction.getAmount());
```

These metrics are exposed on the same `/actuator/prometheus` endpoint already scraped in Phase 1, so no new pipeline is needed.

### Why this is non-intrusive
The application increments an in-memory counter as part of work it is **already doing** — there is no extra database query, no aggregation pipeline, and no additional connection or load on MongoDB. The marginal cost is a few CPU instructions per event and a small amount of memory per metric series. There is no schema dependency: the business logic, not the database structure, defines the metric. And because Prometheus scrapes the endpoint every 30 seconds, KPIs become **near-real-time** instead of once-nightly, which is strictly better for the business.

### How it avoids the latency-spike risk
By eliminating the direct database aggregation entirely, the 1:00 AM resource contention disappears. There is simply no longer a heavy read workload hitting MongoDB at night. The metrics are computed incrementally throughout the day at negligible cost, spread evenly rather than concentrated in a single damaging window.

### Handling aggregations and "stateful" KPIs
Most KPIs the nightly job produced are **flow** metrics (events per day, transaction totals) and map perfectly to counters; daily/weekly rollups are then computed cheaply at query time in Mimir using `increase(business_transactions_total[1d])` or precomputed **recording rules**. For **stock**/snapshot KPIs that are not naturally event-driven (e.g., "total active subscriptions right now"), there are two non-intrusive options: (a) update a Micrometer **gauge** in the code path that changes the underlying state, or (b) if a value genuinely must be read from the database, expose it via a **lightweight, single indexed-count query on a read replica** at a low frequency (e.g., every few minutes) — never a heavy aggregation against the primary. The default should always be event-driven counters.

### Scraping and visualization
The business metrics are scraped by the same Prometheus agent via the existing ServiceMonitor, remote-written to Mimir, retained for >1 year alongside everything else, and surfaced on a dedicated **Business KPI dashboard** in Grafana. Because they share the time-series store with infrastructure and JVM metrics, we can finally correlate business outcomes with system behavior — e.g., overlay transaction throughput against GC pauses or pod restarts on a single dashboard.

---

## 6. Summary Table

| Component | Selected Tool | Purpose | Integration Point |
|---|---|---|---|
| Infra metrics collection | kube-prometheus-stack (Prometheus agent, kube-state-metrics, node-exporter) | Scrape node, pod, and cluster-object metrics | ServiceMonitors in app cluster; remote_write to Mimir |
| JVM/app metrics | Micrometer (Prometheus registry) | Expose heap, GC, threads, HTTP metrics | `/actuator/prometheus` scraped by Prometheus agent |
| Business metrics | Micrometer counters/gauges in app code | Real-time business KPIs replacing nightly job | Same Prometheus endpoint; recording rules in Mimir |
| Long-term storage | Grafana Mimir + S3-compatible object storage | Scalable, cheap >1-year metric retention | Receives remote_write; queried by Grafana |
| Log storage (existing) | Loki + Promtail | Log aggregation | Already in place; Grafana datasource |
| Tracing instrumentation | OpenTelemetry Java agent | Auto-instrument requests, JDBC, Mongo | OTLP export to OTel Collector |
| Trace backend | Grafana Tempo + S3 object storage | Store and query distributed traces | OTel Collector → Tempo; Grafana datasource |
| Trace pipeline | OpenTelemetry Collector | Receive, batch, sample, route spans | OTLP in → Tempo out |
| Visualization | Grafana | Unified dashboards for metrics, logs, traces | Datasources: Mimir, Loki, Tempo |
| Alerting | Mimir ruler / Grafana Alerting | SLO and saturation alerts | Rules over Mimir metrics |

---

## 7. Risks and Mitigations

1. **Metric cardinality explosion.** High-cardinality labels (user IDs, request IDs, unbounded tag values) can blow up Mimir's memory and cost. *Mitigation:* enforce label hygiene in code review, set Mimir per-tenant series limits, and monitor active series as a first-class metric.

2. **Cross-cluster remote_write reliability and security.** Metrics traverse the network from the app cluster to the infra cluster; an outage or open endpoint is a risk. *Mitigation:* secure the path with auth tokens/mTLS, rely on the Prometheus agent's WAL and retry/backoff to buffer during brief outages, and alert on remote_write failures.

3. **Business KPI miscount during the cutover.** New event-driven counters could disagree with the old nightly aggregates due to logic differences or counter resets. *Mitigation:* run both in parallel for one to two weeks, reconcile the numbers, and always query counters with `increase()`/`rate()` summed across pods rather than raw values.

4. **Tracing overhead and data volume.** Full tracing adds per-request overhead and can generate large data volumes. *Mitigation:* start with probabilistic sampling (~10%), validate overhead in staging, and adopt tail-based sampling to keep error/slow traces while discarding routine ones.

5. **Operational burden on a small team.** Running Mimir, Tempo, and the OTel pipeline is more to maintain than logs alone. *Mitigation:* deploy monolithic/simple modes initially (our scale doesn't need microservices), manage everything as code via Helm and Grafana provisioning, and reuse the existing object-storage and Grafana operational knowledge from the Loki setup to flatten the learning curve.

---

This proposal keeps the team entirely within open-source, Grafana-native tooling that builds on the Loki investment already made, removes the known 1:00 AM reliability risk at its root, and gives us cost-efficient multi-year retention with a single correlated view across metrics, logs, and traces.
