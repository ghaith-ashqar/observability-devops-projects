# Observability & DevOps Projects

Two self-contained DevOps/SRE take-home projects by **Ghaith Ashqar**. Each option lives in
its own folder with its own README — start there.

| Option | Project | What it is |
|---|---|---|
| **[Option 1](./option-1)** | **FWaaS Kubernetes Load Testing Framework** | A distributed load-testing framework where each Kubernetes pod behaves like a real Firewall-as-a-Service client — it **enrolls, authenticates, then runs a k6 scenario** — scaled `10 → 10k` pods via Terraform, with Prometheus/Loki/Grafana observability to find where the platform breaks and *why*. Presentation deck. |
| **[Option 2](./option-2)** | **Kubernetes + Java Observability Proposal** | An open-source observability proposal for a Java (JRE21) app on Kubernetes that today has **logs only**. Completes the Grafana **LGTM** stack — infra + JVM metrics via Prometheus/Micrometer, a **non-intrusive Micrometer replacement for the nightly MongoDB job**, >1yr storage on S3-compatible object storage (Mimir), and distributed tracing (Tempo). See the full `PROPOSAL.md`. |

---

## Option 1 — FWaaS Kubernetes Load Testing Framework

FWaaS access is gated: a client must enroll and authenticate before it can send traffic, so
a raw HTTP generator would hide the real bottlenecks. This framework simulates the **full
client lifecycle** — Terraform provisions a Kubernetes-based distributed test runner, each
pod acts as an independent client (enroll → auth → k6), and load is ramped gradually with
readiness gates so *application* limits are separated from *Kubernetes* limits. The value
comes from correlating load with platform and application signals in Grafana.

→ **[`option-1/`](./option-1)** (deck + README)

## Option 2 — Kubernetes + Java Observability Proposal

Adds metrics and distributed tracing to a Java/Kubernetes environment that currently ships
only logs (Promtail → Loki), stores everything long-term in cheap S3-compatible object
storage, and **replaces a risky nightly MongoDB polling job** with real-time,
application-exposed business metrics — all on the Grafana LGTM ecosystem so it extends the
existing Loki setup instead of introducing a competing toolchain.

→ **[`option-2/`](./option-2)** (`PROPOSAL.md`)
