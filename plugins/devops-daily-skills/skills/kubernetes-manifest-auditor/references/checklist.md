# Kubernetes Manifest Audit Checklist

The rubric the auditor works through. Each item lists what to look for, why it matters, a default severity, and the fix. Severity is a starting point; adjust for the workload's real context and say why when you do.

---

## 1. Security context

### Runs as root
- **Check:** `securityContext.runAsNonRoot: true` is set (pod or container), and `runAsUser` is a non-zero UID. Absent means the container likely runs as UID 0.
- **Why:** A container escape or app RCE lands as root on the node. Non-root shrinks that blast radius dramatically.
- **Severity:** Critical if also privileged or mounting host paths; otherwise Warning.
- **Fix:**
  ```yaml
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
  ```

### Privilege escalation allowed
- **Check:** `allowPrivilegeEscalation: false` on every container. Default is `true`.
- **Why:** Blocks `setuid` binaries and similar from gaining more privileges than the parent process.
- **Severity:** Warning (Critical when combined with added capabilities).
- **Fix:** `allowPrivilegeEscalation: false`.

### Privileged container
- **Check:** `securityContext.privileged: true`.
- **Why:** A privileged container is effectively root on the host with all capabilities and device access. Almost never needed by application workloads.
- **Severity:** Critical unless it is a known node-agent (CNI, CSI, monitoring) that documents the need.
- **Fix:** Remove `privileged: true`; grant only the specific capability required.

### Capabilities not dropped
- **Check:** `securityContext.capabilities.drop: ["ALL"]`, with only the specific caps added back.
- **Why:** Containers inherit a default capability set most apps never use (`NET_RAW`, `SETUID`, etc.). Dropping all and adding back the few you need is least privilege.
- **Severity:** Warning.
- **Fix:**
  ```yaml
  securityContext:
    capabilities:
      drop: ["ALL"]
      # add: ["NET_BIND_SERVICE"]  # only if binding a low port
  ```

### Writable root filesystem
- **Check:** `readOnlyRootFilesystem: true`, with `emptyDir` volumes for the paths that genuinely need writes (e.g. `/tmp`).
- **Why:** Stops an attacker from writing tools or modifying binaries inside the container.
- **Severity:** Warning.
- **Fix:** set `readOnlyRootFilesystem: true` and mount writable dirs explicitly.

### seccomp not set
- **Check:** `seccompProfile.type: RuntimeDefault` (pod or container).
- **Why:** Restricts the syscalls the container can make, removing whole classes of kernel-level attack surface.
- **Severity:** Warning.
- **Fix:** `seccompProfile: { type: RuntimeDefault }`.

### Host namespaces
- **Check:** `hostNetwork`, `hostPID`, `hostIPC` are not `true`.
- **Why:** Sharing host namespaces breaks isolation: `hostPID` exposes every process on the node, `hostNetwork` exposes the node's interfaces and skips NetworkPolicy.
- **Severity:** Critical unless it is a node-level agent that needs it.
- **Fix:** remove the host namespace flags.

### hostPath volumes
- **Check:** any `hostPath` volume, especially to sensitive paths (`/`, `/var/run/docker.sock`, `/etc`, `/var/lib/kubelet`).
- **Why:** A hostPath mount is a direct line to the node filesystem; `/var/run/docker.sock` or the containerd socket is instant node takeover.
- **Severity:** Critical for sensitive paths, Warning otherwise.
- **Fix:** replace with a proper volume type (PVC, `emptyDir`, `configMap`, CSI) or remove.

---

## 2. Supply chain

### Mutable image tag
- **Check:** images pinned by digest (`@sha256:...`) or at least a specific version tag, never `:latest` or no tag.
- **Why:** `:latest` is non-deterministic: two rollouts can run different code, and you cannot reason about what is deployed. Digests are immutable.
- **Severity:** Warning (Critical for anything security-sensitive where reproducibility matters).
- **Fix:** `image: myapp:1.7.3` or `image: myapp@sha256:...`.

### imagePullPolicy with mutable tags
- **Check:** if using a tag like `:latest`, `imagePullPolicy: Always`; if pinned by digest, `IfNotPresent` is fine.
- **Why:** avoids running a stale cached image while thinking you shipped a new one.
- **Severity:** Info.

### Untrusted or implicit registry
- **Check:** images come from a known registry, not an unpinned public image where a typo or takeover could substitute malware.
- **Why:** supply-chain substitution. Prefer your own registry or a trusted mirror with digest pinning.
- **Severity:** Info to Warning depending on exposure.

---

## 3. Resource management

### No resource requests/limits
- **Check:** every container sets `resources.requests` and `resources.limits` for CPU and memory.
- **Why:** No memory limit means one leaking pod can OOM the whole node and take neighbors with it. No requests means the scheduler cannot place the pod sensibly and it has no guaranteed share.
- **Severity:** Critical for memory limits on a shared node; Warning for CPU.
- **Fix:**
  ```yaml
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      memory: 256Mi   # cap memory to protect the node
      # cpu limits are optional and often better left unset to avoid throttling
  ```

### QoS class
- **Check:** understand the resulting QoS. Requests == limits gives `Guaranteed`; requests only gives `Burstable`; nothing gives `BestEffort` (first to be evicted).
- **Why:** `BestEffort` pods are killed first under node pressure. Critical workloads should be `Burstable` or `Guaranteed`.
- **Severity:** Info, escalate for critical services.

---

## 4. Reliability

### Missing probes
- **Check:** `livenessProbe` and `readinessProbe` (and `startupProbe` for slow starters).
- **Why:** Without a readiness probe, traffic hits a pod before it is ready. Without a liveness probe, a wedged process never restarts. Startup probes stop liveness from killing slow-booting apps.
- **Severity:** Warning.
- **Fix:**
  ```yaml
  readinessProbe:
    httpGet: { path: /healthz, port: 8080 }
    initialDelaySeconds: 5
    periodSeconds: 10
  livenessProbe:
    httpGet: { path: /livez, port: 8080 }
    initialDelaySeconds: 15
    periodSeconds: 20
  ```

### Single replica for a serving workload
- **Check:** `replicas >= 2` for anything that serves traffic; note that a bare Pod has no controller at all.
- **Why:** One replica means any node drain, crash, or rollout is downtime.
- **Severity:** Warning (Info for genuinely single-instance or batch work).
- **Fix:** `replicas: 2` or more, plus anti-affinity below.

### No PodDisruptionBudget
- **Check:** a `PodDisruptionBudget` exists for multi-replica serving workloads.
- **Why:** Without a PDB, a node drain can evict all replicas at once. A PDB keeps a minimum available during voluntary disruptions.
- **Severity:** Warning.
- **Fix:**
  ```yaml
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata: { name: api-gateway }
  spec:
    minAvailable: 1
    selector: { matchLabels: { app: api-gateway } }
  ```

### No anti-affinity / topology spread
- **Check:** `topologySpreadConstraints` or pod anti-affinity so replicas do not all land on one node/zone.
- **Why:** Three replicas on one node is one node failure away from full outage.
- **Severity:** Warning.
- **Fix:** a `topologySpreadConstraints` block over `kubernetes.io/hostname` (and `topology.kubernetes.io/zone`).

### Rollout strategy / grace period
- **Check:** `strategy.rollingUpdate` with sane `maxUnavailable`/`maxSurge`, and a `terminationGracePeriodSeconds` that fits the app's shutdown.
- **Why:** Aggressive rollouts drop capacity; too-short grace periods cut connections mid-flight.
- **Severity:** Info.

---

## 5. Networking and access

### No NetworkPolicy
- **Check:** a `NetworkPolicy` (default-deny plus explicit allows) exists for the namespace/workload. Remember policies are additive and only enforced if the CNI supports them.
- **Why:** With no policy, every pod can talk to every other pod. One compromised pod reaches your databases, internal APIs, and control-plane-adjacent services. (See the Argo CD repo-server RCE for a real example of this default biting hard.)
- **Severity:** Warning (Critical for namespaces with sensitive services).
- **Fix:** a default-deny ingress policy plus targeted allows.
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata: { name: default-deny-ingress }
  spec:
    podSelector: {}
    policyTypes: ["Ingress"]
  ```

### Service exposure
- **Check:** `Service` type. A `LoadBalancer` or `NodePort` on an internal service exposes it publicly.
- **Why:** Accidental internet exposure of an admin or database service.
- **Severity:** Critical if it exposes something sensitive, Info otherwise.
- **Fix:** use `ClusterIP` for internal services; front public traffic with an Ingress.

### Ingress without TLS
- **Check:** `Ingress` defines a `tls` block.
- **Why:** plaintext traffic to a public endpoint.
- **Severity:** Warning.

### Over-broad RBAC
- **Check:** `Role`/`ClusterRole` rules. Flag `verbs: ["*"]`, `resources: ["*"]`, `apiGroups: ["*"]`, and any binding to `cluster-admin`.
- **Why:** A compromised ServiceAccount inherits these rights. Wildcards and cluster-admin turn a small foothold into cluster takeover.
- **Severity:** Critical for cluster-admin/wildcards, Warning for broad-but-scoped.
- **Fix:** enumerate the specific `apiGroups`, `resources`, and `verbs` the workload actually needs.

### Automounted ServiceAccount token
- **Check:** `automountServiceAccountToken: false` on pods/ServiceAccounts that do not call the Kubernetes API.
- **Why:** A mounted token is a credential an attacker can use against the API server. Most app pods never need it.
- **Severity:** Warning.
- **Fix:** `automountServiceAccountToken: false`.

---

## 6. Secrets and config

### Secret values in plaintext
- **Check:** `Secret` objects with real-looking values in `stringData`/`data`, or secrets committed alongside manifests.
- **Why:** Secrets in Git or manifests are leaked the moment the repo is shared or cloned.
- **Severity:** Critical.
- **Fix:** use a secrets manager (External Secrets Operator, Sealed Secrets, SOPS, cloud secret stores) and reference, never embed.

### Secrets injected as env vars
- **Check:** sensitive values via `env`/`envFrom` from a Secret, versus mounted as files.
- **Why:** Env vars leak easily: a crash dump, a `/proc/<pid>/environ` read, a logging middleware, or an RCE that dumps `env` hands them over. (Exactly the first step in the Argo CD repo-server exploit.) Mounted files are harder to exfiltrate wholesale and can be rotated.
- **Severity:** Warning.
- **Fix:** mount the Secret as a volume and read from the file path.

### Config in the image
- **Check:** environment-specific config baked into the image instead of `ConfigMap`/env.
- **Why:** forces a rebuild per environment and encourages drift.
- **Severity:** Info.

---

## Cross-cutting checks

- **Labels and selectors:** recommended labels (`app.kubernetes.io/name`, `.../instance`, `.../version`) present; selectors actually match pod labels (a mismatch means the Service routes to nothing).
- **Namespace:** workloads are not defaulting into `default`; sensitive workloads are isolated.
- **Deprecated APIs:** flag removed/deprecated `apiVersion`s (e.g. old `extensions/v1beta1`, `policy/v1beta1` PDBs).
- **Immutable field foot-guns:** note changes that would require object recreation (e.g. `Service.spec.clusterIP`, some `selector` changes).
