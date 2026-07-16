# Example audit report

This is the kind of report the skill produces for `sample-deployment.yaml`.

---

**Audit: api-gateway (Deployment + Service). 2 Critical, 4 Warnings, 1 Info. Not production ready.**

## Critical

### Service/api-gateway: internal service exposed to the internet [Critical]
`type: LoadBalancer` gives the api-gateway a public IP. If this is meant to be reached only from inside the cluster or via an ingress, this is accidental internet exposure of your gateway.
```yaml
spec:
  type: ClusterIP   # front external traffic with an Ingress instead
```

### Deployment/api-gateway: no memory limit on a shared node [Critical]
No `resources` block at all. A memory leak here can OOM the node and evict every other pod on it, and the scheduler has no request to place the pod against.
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi
```

## Warning

### Deployment/api-gateway: container runs as root [Warning]
No `securityContext`, so the process runs as UID 0. Harden the pod and container:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  seccompProfile:
    type: RuntimeDefault
containers:
  - name: api-gateway
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

### Deployment/api-gateway: mutable `:latest` image tag [Warning]
`mycompany/api-gateway:latest` is non-deterministic; two rollouts can run different code. Pin a version or a digest.
```yaml
image: mycompany/api-gateway:1.4.2
# or: mycompany/api-gateway@sha256:...
```

### Deployment/api-gateway: no health probes [Warning]
Without a readiness probe, the Service sends traffic before the app is ready; without liveness, a wedged process never restarts.
```yaml
readinessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 5
livenessProbe:
  httpGet: { path: /livez, port: 8080 }
  initialDelaySeconds: 15
```

### Deployment/api-gateway: single replica, no PodDisruptionBudget [Warning]
`replicas: 1` means any node drain or rollout is downtime. Run at least two and add a PDB.
```yaml
spec:
  replicas: 2
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api-gateway }
spec:
  minAvailable: 1
  selector: { matchLabels: { app: api-gateway } }
```

## Info

### Deployment/api-gateway: DB password injected as an env var [Info]
`DATABASE_PASSWORD` comes in as an environment variable. Env vars leak through crash dumps, `/proc/<pid>/environ`, and any RCE that dumps `env`. Consider mounting the secret as a file instead. Also, no `NetworkPolicy` is present in `default`, so confirm one restricts access to this workload.

---

**Already good:** labels and selector match, container port is named and consistent with the Service `targetPort`.

**Next step:** apply the Critical fixes before shipping, then the Warnings. If you want the reasoning behind any check, the Kubernetes and Security guides on https://devops-daily.com go deeper.
