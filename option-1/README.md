# Option 1 — FWaaS Kubernetes Load Testing Framework

A DevOps/SRE project that answers one question for a **Firewall-as-a-Service (FWaaS)**
platform:

> **How many independent clients can the platform support before latency, errors, or
> infrastructure pressure become visible?**

FWaaS access is *gated* — a client cannot just send traffic. It must first **enroll**,
**authenticate**, and **receive a usable identity** before it can talk to the firewall. A
naive HTTP request generator would skip that journey and hide the real bottlenecks. So this
framework simulates the **complete client lifecycle at scale**: each Kubernetes pod behaves
like a real FWaaS client — it enrolls, authenticates, gets credentials, then runs a
[k6](https://k6.io/) load scenario against the firewall.

| | |
|---|---|
| **Deck** | [`fwaas-k8s-load-testing.pptx`](./fwaas-k8s-load-testing.pptx) — 18 slides, 16:9 |
| **Stack** | Terraform · Kubernetes (Jobs) · Docker · k6 · Prometheus · Loki · Grafana |
| **Scale target** | 10 → 100 → 1k → 5k → **10k** simulated clients |

---

## The idea in one picture

```
Terraform (IaC)                 Kubernetes                       FWaaS
──────────────                  ──────────                       ─────
namespace + RBAC   ──provision─►  N Load-Test Pods  ──enroll───►  Enrollment API
secrets / config                  (each = 1 client)  ◄─identity──  (token / cert)
job + scale params                       │
                                         └──k6 run──►  FWaaS control/data plane
                                              │
                                         metrics · logs
                                              │
                                   Prometheus · Loki ─► Grafana (single view)
```

Each pod runs the same `entrypoint.sh` flow: `enroll.sh` → `export FW_TOKEN` →
`k6 run firewall_test.js` → return an exit code. Because every phase is separated
(startup → enrollment → authentication → traffic → platform limits), a failure can be
attributed to the *stage* it happened in rather than lumped together as "the test failed."

## Why it's built this way

- **Realistic user journey, not synthetic traffic.** The enrollment + auth path is part of
  the test, so bottlenecks *before* the firewall (identity issuance, rate limits) surface
  instead of being skipped.
- **Kubernetes pods as independent clients.** Each pod has its own lifecycle and exit
  status, so thousands of concurrent, isolated clients can be simulated and individually
  diagnosed.
- **Infrastructure as Code.** Terraform makes the whole environment — namespace, RBAC,
  secrets, jobs, and scale parameters — repeatable and reviewable.
- **Controlled scaling.** Load is ramped `10 → 100 → 1k → 5k → 10k` with readiness gates
  between steps, so *application* limits are separated from *Kubernetes* limits (scheduling,
  image pulls, DNS/network, node pressure) instead of jumping straight to 10k and hiding the
  real bottleneck.
- **Observability first.** Metrics (Prometheus / k6 output: request rate, p95/p99, error
  rate) and logs (Loki/Promtail: enrollment errors, TLS/DNS) are correlated in Grafana.
  Core lesson: *load testing without observability shows that something failed, but not why.*

## Reference repository layout (as presented)

```
fwaas-load-testing-repo/
├── docker/          # test-runner image (k6 + enrollment automation + scripts)
├── scripts/         # enrollment + entrypoint
├── tests/           # k6 scenarios (thresholds, latency targets, error budgets)
├── terraform/       # Kubernetes resources as code
├── k8s/             # example Job manifest
├── observability/   # dashboard / monitoring notes
└── docs/            # architecture + interview track
```

## Security & production hardening

The prototype packages scripts and assets into a self-contained image for easy
reproducibility. For production the deck recommends **not** baking secrets/certs into
images and instead injecting them at runtime, along a hardening path:

`Kubernetes Secrets` (baseline) → `External Secrets Operator` → `HashiCorp Vault`
(dynamic, short-lived credentials), plus **least-privilege RBAC** and **NetworkPolicy** to
restrict the test namespace. Execution moves from a raw image to **Kubernetes Jobs / the
k6 Operator** for clean lifecycle and reporting.

## How it works — step by step

1. **Provision the environment (Terraform).** Terraform creates the Kubernetes namespace,
   RBAC, secrets/config, and the Job definition with the target number of pods. Because the
   environment is defined as code, it is repeatable and reviewable — the same test can be
   re-run identically.
2. **Start the load-test pods (Kubernetes).** Kubernetes launches *N* Job pods, each one an
   independent FWaaS client with its own lifecycle and exit status. Pod readiness is watched
   before doing anything else, so cluster startup noise is not mistaken for application
   failure.
3. **Enroll (per pod).** Each pod runs `enroll.sh`, which calls the **Enrollment API** to
   register itself as a new client — exactly what a real client must do before it is allowed
   to use the firewall.
4. **Authenticate (per pod).** The Enrollment API returns a token or certificate. The pod
   stores it as its identity (`export FW_TOKEN`) and uses it for all subsequent traffic.
5. **Generate load (k6).** With real credentials, the pod runs `k6 run firewall_test.js`
   against the FWaaS control/data plane, applying SLO-style thresholds: request failure
   rate, p95/p99 latency, and timeout behaviour.
6. **Emit signals.** Each pod exposes metrics (k6 / Prometheus: request rate, p95/p99, error
   rate), writes logs (Loki / Promtail: enrollment errors, TLS/DNS), and returns an exit
   code — so a failure can be attributed to a specific **client** and a specific **phase**
   (startup, enrollment, authentication, traffic, or platform limits).
7. **Observe & correlate (Grafana).** Metrics and logs are brought together in Grafana to
   see where latency or errors first appear, and whether the limit is on the **application**
   side or the **Kubernetes** side (scheduling, image pulls, DNS/network, node pressure).
8. **Ramp gradually.** Repeat steps 2–7 at `10 → 100 → 1k → 5k → 10k` pods, advancing only
   when pods are Ready, enrollment success is healthy, and error rate / latency are within
   threshold. The goal is to capture the **first meaningful error pattern** and the load
   level at which the platform becomes unreliable — not simply to reach 10k pods.

## Open the deck

Open [`fwaas-k8s-load-testing.pptx`](./fwaas-k8s-load-testing.pptx) in PowerPoint,
Keynote, or Google Slides (or LibreOffice Impress).
