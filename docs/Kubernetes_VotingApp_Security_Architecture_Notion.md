
# ☸️ Kubernetes Voting App — Complete Breakdown for Security Architect Interviews

## 🧱 Part 1: Kubernetes Fundamentals (Voting App Example)

<details>
<summary>🐳 1. Docker — Container Basics</summary>

**Concept:**  
Docker packages apps and dependencies into lightweight, isolated containers that run anywhere.

**In this app:**  
- `voting-app` → Flask (Python)  
- `result-app` → Node.js  
- `redis`, `postgres`, `worker` → official images  

**Why it matters:**  
✅ Reproducibility  
✅ Speed  
✅ Portability  
✅ Simplified Dev-to-Prod migration

</details>

---

<details>
<summary>☸️ 2. Kubernetes — The Orchestrator</summary>

Manages, scales, and heals containers automatically.  
Think of it as the **operating system for containers**.

Handles:
- Scheduling pods
- Restarting failed pods
- Load balancing
- Rolling updates

</details>

---

<details>
<summary>📦 3. Pods — Smallest Deployable Unit</summary>

- Wraps **one or more containers** that share storage and network.
- Example in this app:
  - `voting-app`
  - `result-app`
  - `redis`
  - `postgres`
  - `worker`
- If one crashes, only that pod restarts — not the whole app.

</details>

---

<details>
<summary>🧱 4. Nodes — The Workers</summary>

Each **Node** runs:
- `kubelet` → talks to control plane  
- `containerd` / `docker` → runs containers  
- `kube-proxy` → handles networking

Pods are **scheduled to nodes** automatically.

</details>

---

<details>
<summary>🧠 5. Cluster — The Whole System</summary>

**Control Plane:**  
- API Server  
- Scheduler  
- Controller Manager  
- etcd  

**Worker Nodes:**  
- Run workloads (Pods)  

Together they form your **Kubernetes Cluster**.

</details>

---

<details>
<summary>🚀 6. Deployments — Declarative Management</summary>

Define desired state → Kubernetes ensures it.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: voting-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: voting-app
  template:
    metadata:
      labels:
        app: voting-app
    spec:
      containers:
      - name: voting-app
        image: voting-app:v1
        ports:
        - containerPort: 80
```

If a pod crashes, Deployment recreates it automatically.

</details>

---

<details>
<summary>🔁 7. Services — Stable Networking</summary>

Pods’ IPs are ephemeral → Services provide stable access.

| Type | Scope | Example Use |
|------|--------|-------------|
| **ClusterIP** | Internal only | Redis, Postgres |
| **NodePort** | Exposed externally via node IP | Voting-app |
| **LoadBalancer** | Cloud-managed external IP | Production |

```yaml
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
  type: ClusterIP
```

</details>

---

<details>
<summary>🔄 8. Overall Flow (Voting App)</summary>

1️⃣ User votes → request hits **Voting-App (Flask)**  
2️⃣ Flask writes vote to **Redis** (in-memory)  
3️⃣ **Worker** picks from Redis → stores in **Postgres**  
4️⃣ **Result-App** reads from Postgres → displays results  

➡️ **Flow:**  
`User → Voting-App → Redis → Worker → Postgres → Result-App → User`

</details>

---

<details>
<summary>🛡️ 9. Security Essentials (Baseline)</summary>

- Image scanning (Trivy, Clair)  
- Run as non-root  
- Drop capabilities  
- Store secrets in **KMS or Kubernetes Secrets**  
- Restrict pod-to-pod communication via **NetworkPolicies**  
- Implement **RBAC least privilege**  
- Use **Prometheus, Falco** for visibility

</details>

---

## 🔐 Part 2: Security Architecture View (Security Architect Focus)

<details>
<summary>0️⃣ System Model & Threat Boundaries</summary>

**Data flow:**  
`Internet → Ingress → Frontend Pods → Internal Tiers (Redis/Postgres)`

**Boundaries:**
1. Internet ↔ Ingress  
2. Frontend ↔ Backend  
3. Control plane ↔ Workloads  

**Goal:** Isolate, minimize privileges, enforce default-deny.

</details>

---

<details>
<summary>1️⃣ Supply Chain Security</summary>

**Threats:**  
- Unscanned or malicious base images  
- Secrets baked into image  

**Controls:**
- Distroless base images  
- Scan with Trivy  
- Enforce **signed images** (Cosign + Kyverno)  
- Generate SBOMs  
- Block `:latest` tags  

</details>

---

<details>
<summary>2️⃣ Pod Hardening</summary>

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop: ["ALL"]
```

- Apply **Pod Security Standards (Restricted)**  
- Use resource limits, health probes  
- One pod per function (micro-isolation)

</details>

---

<details>
<summary>3️⃣ RBAC & Service Accounts</summary>

- One ServiceAccount per app  
- Disable token auto-mount  
- Grant minimal verbs (`get`, `list`)  
- Bind via Role/RoleBinding  

**Cluster Admin access** via OIDC/SSO only.

</details>

---

<details>
<summary>4️⃣ Secrets Management</summary>

- Integrate with **External Secrets Operator**  
- Pull from **AWS Secrets Manager / Azure Key Vault**  
- Encrypt at rest + rotate regularly  
- Avoid plain env variables  

</details>

---

<details>
<summary>5️⃣ Network Isolation & Policies</summary>

- Enforce **default-deny** NetworkPolicies  
- Allow specific flows:
  - voting → redis  
  - worker → redis/postgres  
  - result → postgres  

```yaml
kind: NetworkPolicy
metadata:
  name: allow-worker-to-postgres
spec:
  podSelector:
    matchLabels:
      app: postgres
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: worker
    ports:
    - port: 5432
```

</details>

---

<details>
<summary>6️⃣ Ingress, WAF, and TLS</summary>

- TLS ≥ 1.2 (cert-manager or ACM)  
- Enable HSTS, CSP, XSS protection headers  
- Apply **rate limiting**  
- Use WAF or ModSecurity CRS  

</details>

---

<details>
<summary>7️⃣ Data Tier Security (Redis & Postgres)</summary>

- Redis:
  - Enable `requirepass`
  - Encrypted PVs  
- Postgres:
  - Strong password/roles  
  - Encrypted storage class  
  - ClusterIP only  

</details>

---

<details>
<summary>8️⃣ Observability & Runtime Protection</summary>

- **Prometheus/Grafana** for metrics  
- **Falco** / **Cilium Tetragon** for runtime threats  
- **Central logging** (Fluent Bit → SIEM)  
- **Drift detection:** compare running vs desired digests  

</details>

---

<details>
<summary>9️⃣ Threat Chain (MITRE ATT&CK)</summary>

| Phase | Example | Defense |
|--------|----------|----------|
| Initial Access | NodePort vuln | Restrict NodePorts |
| Execution | Malicious image | Signed + scanned |
| Persistence | CronJob backdoor | RBAC, audit |
| Priv Esc | Privileged pod | PSS restricted |
| Lateral Move | Pod scanning | NetworkPolicy |
| Cred Access | SA token theft | Disable automount |
| Exfiltration | Open egress | Restrict egress |
| Impact | Crypto mining | Quotas, runtime alerts |

</details>

---

<details>
<summary>🔒 10️⃣ Secure-by-Default Blueprint</summary>

✅ Namespaces (frontend/data) with **PSS=restricted**  
✅ Hardened Deployments with probes & limits  
✅ ClusterIP for internal, NodePort/Ingress for external  
✅ Default deny network  
✅ RBAC least privilege  
✅ Secrets via KMS  
✅ Kyverno policies  
✅ TLS ingress  
✅ Observability baseline  

</details>

---

<details>
<summary>⚙️ 11️⃣ Day-2 Security Operations</summary>

- Patch cadence: weekly images, monthly cluster  
- Simulate pod compromise quarterly  
- Backup Postgres + etcd  
- Posture scan: kube-bench, Kubescape  
- Monitor:  
  - 5xx spikes  
  - Pod restarts  
  - Unsigned image denials  
  - Falco alerts  

</details>

---

<details>
<summary>🎯 12️⃣ Interview-Grade Takeaways</summary>

💡 “Default-deny east-west” = must, not nice-to-have  
💡 Admission control > documentation  
💡 Service Mesh adds value **only** with enforced mTLS  
💡 Secrets belong in KMS, not YAML  
💡 Observability is part of security  
💡 Redis/Postgres = fortress — encrypt, isolate, backup  

</details>

---

<details>
<summary>🧩 13️⃣ Verification Checklist</summary>

```bash
# Test isolation
kubectl -n frontend exec deploy/voting-app -- nc -zv redis.data.svc.cluster.local 6379
kubectl -n frontend exec deploy/voting-app -- nc -zv postgres.data.svc.cluster.local 5432  # should fail

# Privileged pods check
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" "}{.spec.containers[*].securityContext.privileged}{"\n"}{end}'

# Signed image admission
kubectl describe events -A | grep -i kyverno

# Token mounts
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" "}{.spec.automountServiceAccountToken}{"\n"}{end}'
```

</details>
