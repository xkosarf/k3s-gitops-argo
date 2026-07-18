# k3s-gitops-argo — Kubernetes Security Lab

A single-node Kubernetes security lab running on Hetzner Cloud, managed via GitOps with ArgoCD. Built to learn, test, and demonstrate cloud-native security tooling in a realistic environment.

---

## Infrastructure

| Component | Details |
|-----------|---------|
| **Provider** | Hetzner Cloud (eu-central-1 / Nuremberg) |
| **Node** | CX22 — 4 vCPU, 8GB RAM |
| **OS** | Ubuntu 24.04 |
| **Kubernetes** | K3s v1.33.5 (single node, control-plane + worker) |
| **Ingress** | Traefik (K3s built-in) |
| **TLS** | cert-manager + Let's Encrypt (production) |
| **DNS** | kosar.sk via websupport.sk |
| **Access** | SSH key auth, kubeconfig via K3s |

---

## Architecture

```
GitHub (xkosarf/k3s-gitops-argo)
        │
        │  GitOps sync (ArgoCD)
        ▼
┌─────────────────────────────────────────────┐
│           K3s Single Node (Hetzner)         │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  ArgoCD  │  │ Kyverno  │  │  Trivy   │  │
│  │  GitOps  │  │ Policies │  │ Operator │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Falco   │  │  Loki +  │  │  Policy  │  │
│  │  eBPF    │  │ Grafana  │  │ Reporter │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│  ┌──────────┐  ┌──────────┐                │
│  │ Traefik  │  │  cert-   │                │
│  │ Ingress  │  │ manager  │                │
│  └──────────┘  └──────────┘                │
└─────────────────────────────────────────────┘
```

All workloads are declaratively managed — pushing to `main` is the only way to change cluster state.

---

## GitOps Structure

```
clusters/k3s-hetzner/
├── root-app.yaml                    # App-of-Apps root — bootstraps everything
├── namespaces.yaml
├── argocd/
│   ├── argocd-app.yaml              # ArgoCD self-managed (Helm chart 8.5.10)
│   ├── ingress-argocd.yaml
│   └── project-default.yaml
├── cert-manager/
│   ├── cert-manager-app.yaml        # cert-manager (Helm)
│   └── clusterissuer-prod.yaml      # Let's Encrypt production issuer
├── traefik/
│   └── traefik-app.yaml             # Traefik ingress (Helm)
├── kyverno/
│   ├── kyverno-app.yaml             # Kyverno engine (Helm chart 3.7.0)
│   ├── kyverno-policies-app.yaml    # Kyverno policies (separate ArgoCD app)
│   └── policies/
│       ├── pss-baseline.yaml        # Pod Security Standards baseline (Audit)
│       ├── disallow-latest-tag.yaml # Block :latest images (Enforce)
│       ├── disallow-privileged.yaml # Block privileged containers (Audit)
│       └── require-resource-limits.yaml  # Require CPU/memory limits (Audit)
├── trivy-operator/
│   └── trivy-operator-app.yaml      # Trivy Operator (Helm chart 0.33.2)
├── falco/
│   └── falco-app.yaml               # Falco + Falcosidekick (Helm chart 4.21.2)
├── loki/
│   └── loki-app.yaml                # Loki + Promtail + Grafana (Helm chart 2.10.2)
└── policy-reporter/
    └── policy-reporter-app.yaml     # Policy Reporter UI (Helm chart 2.24.2)
```

The root app uses `directory.recurse: true` with exclusions for Helm values files and Kyverno policies (which are managed by their own dedicated ArgoCD application to avoid ownership conflicts).

---

## Security Tools

### Kyverno — Admission Control
**Namespace:** `kyverno` | **Chart:** `kyverno/kyverno:3.7.0`

Kyverno is a Kubernetes-native policy engine. It validates, mutates, and generates resources at admission time — before they land in the cluster.

**Policies in this lab:**

| Policy | Mode | What it does |
|--------|------|--------------|
| `pss-baseline` | Audit | Enforces Kubernetes Pod Security Standards baseline profile |
| `disallow-latest-tag` | **Enforce** | Blocks any pod using `:latest` image tag in non-system namespaces |
| `disallow-privileged-containers` | Audit | Flags containers running with `privileged: true` |
| `require-resource-limits` | Audit | Flags containers missing CPU or memory limits |

System namespaces (kube-system, cattle-system, falco, cert-manager, etc.) are excluded from all policies to avoid breaking legitimate privileged workloads.

Policy violations are visible in the **Policy Reporter UI** and written as `PolicyReport` CRDs queryable with `kubectl get policyreport -A`.

---

### Trivy Operator — Vulnerability & Configuration Scanning
**Namespace:** `trivy-system` | **Chart:** `aquasecurity/trivy-operator:0.33.2`

Trivy Operator runs continuous background scans and writes results as Kubernetes CRDs. No manual scanning needed — it automatically picks up new workloads.

**What it scans:**
- **VulnerabilityReports** — scans container images for known CVEs (using built-in Trivy server mode to avoid external DB download issues)
- **ConfigAuditReports** — checks workload configs for misconfigurations (missing securityContext, root containers, exposed ports, etc.)
- **RbacAssessmentReports** — audits RBAC roles for over-privileged bindings
- **InfraAssessmentReports** — checks cluster-level infrastructure configs

```bash
# View vulnerability findings
kubectl get vulnerabilityreports -A

# View config audit findings (with HIGH/CRITICAL counts)
kubectl get configauditreports -A
```

---

### Falco — Runtime Threat Detection
**Namespace:** `falco` | **Chart:** `falcosecurity/falco:4.21.2`

Falco uses eBPF (modern BPF probe) to hook into the Linux kernel and detect suspicious behavior at runtime — after admission. Where Kyverno stops bad deployments, Falco catches bad behavior in running containers.

**Driver:** `modern_ebpf` — no kernel module compilation needed, works directly with kernel 6.8+.

**Output pipeline:**
```
Falco (eBPF syscall events)
    → JSON output
    → Falcosidekick (HTTP forwarding)
    → Loki (storage)
    → Grafana (visualization)
```

**Custom rule overrides** — the default `Contact K8S API Server From Container` rule generates high noise in clusters where operators legitimately need API access. A local rules override whitelists known-good processes (ArgoCD, Kyverno, Fleet, cert-manager, etc.) so only genuinely unexpected API connections alert.

**Active detections include:**
- Terminal shell spawned in a container
- Unexpected connections to the Kubernetes API server
- Writes to sensitive filesystem paths
- Privilege escalation attempts

```bash
# Live Falco events
kubectl -n falco logs daemonset/falco -f

# Count alerts by rule
kubectl -n falco logs daemonset/falco --tail=1000 | python3 -c "
import sys, json
from collections import Counter
rules = Counter()
for line in sys.stdin:
    try:
        e = json.loads(line)
        rules[e.get('rule','unknown')] += 1
    except: pass
for rule, count in rules.most_common():
    print(f'{count:4d}  {rule}')
"
```

---

### Loki + Grafana — Log Aggregation & Dashboards
**Namespace:** `monitoring` | **Chart:** `grafana/loki-stack:2.10.2`

**Components:**
- **Loki** — log storage, 7-day retention, no persistence (in-memory for lab use)
- **Promtail** — DaemonSet that ships logs from all pods to Loki
- **Grafana** — visualization at `https://grafana.kosar.sk`

**Custom Falco dashboard** included — 5 panels:
1. Total Events (stat)
2. Events by Priority (pie chart — uses `priority` Loki label)
3. Top Rules Triggered (pie chart — uses `rule` Loki label)
4. Events Rate over Time (time series)
5. Raw Falco Events (log panel with formatted output)

Falco events are labeled with `source`, `priority`, `hostname`, and `rule` as Loki stream labels, making them efficiently queryable without full-text search.

```
# Explore Falco events in Grafana
{source="syscall"}
{source="syscall", priority="Warning"}
{source="syscall"} | json | line_format "[{{.priority}}] {{.rule}}"
```

---

### Policy Reporter — Kyverno UI
**Namespace:** `policy-reporter` | **Chart:** `kyverno/policy-reporter:2.24.2`

Web UI at `https://kyverno.kosar.sk` — purpose-built for visualizing Kyverno `PolicyReport` CRDs. Shows:
- Pass/fail counts per policy
- Violations by namespace
- Per-resource policy results
- Kyverno-specific plugin views

Lighter and more accurate for Kyverno data than generic Grafana dashboards.

---

## Public Endpoints

| URL | Service |
|-----|---------|
| `https://argocd.kosar.sk` | ArgoCD GitOps dashboard |
| `https://grafana.kosar.sk` | Grafana log dashboards |
| `https://kyverno.kosar.sk` | Policy Reporter (Kyverno UI) |

All endpoints use TLS certificates issued by Let's Encrypt via cert-manager + Traefik.

---

## Bootstrap (Fresh Cluster)

If starting from scratch with a clean K3s node:

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Apply the root app — ArgoCD will deploy everything else automatically
kubectl apply -f clusters/k3s-hetzner/root-app.yaml

# 3. Get initial ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Everything else is handled by ArgoCD's app-of-apps pattern — sync waves ensure correct deployment order (cert-manager → Kyverno → Trivy → Falco → Loki → Policy Reporter).

---

## Key Design Decisions

**Single-node** — simplicity over HA. This is a security testing lab, not production. The tradeoff is no node-level isolation for testing network policies or DaemonSet behavior across nodes.

**No Rancher Manager in the GitOps stack** — Rancher is present on the cluster (installed separately) to mirror a company setup, but all tooling is managed via ArgoCD + Helm directly. Rancher is not load-bearing for any security tool.

**K3s over RKE2** — K3s has a lower memory footprint and simpler startup (single binary, embedded SQLite). RKE2 was attempted first but had stability issues on 2GB nodes (embedded etcd + kube-scheduler startup race conditions).

**Falcosidekick as the forwarding layer** — Falco stdout → Falcosidekick → Loki gives proper Loki label structure (`source`, `priority`, `rule`, `hostname`) that makes log queries efficient. Direct Promtail scraping of Falco stdout works but loses structured label routing.

**Trivy built-in server mode** — avoids per-scan-job DB downloads which caused OOM kills on a single small node and were blocked by the Hetzner GCR mirror prepending `mirror.gcr.io/` to `ghcr.io` paths.

---

## Planned Additions

- [ ] Custom webapp at `lab.kosar.sk` — static site documenting the lab
- [ ] NeuVector — network segmentation and runtime protection (requires node resize to CX32)
- [ ] Wazuh or ELK — SIEM for log correlation and compliance reporting
- [ ] Second worker node — enables multi-node network policy testing
- [ ] Comprehensive cluster security audit document

---

## Repository

**GitHub:** [xkosarf/k3s-gitops-argo](https://github.com/xkosarf/k3s-gitops-argo)

All cluster state is in git. The cluster is fully reproducible from this repository.
