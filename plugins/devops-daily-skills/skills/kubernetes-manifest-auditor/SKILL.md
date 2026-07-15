---
name: kubernetes-manifest-auditor
description: Audit Kubernetes YAML manifests for security, reliability, and production-readiness issues, then report findings by severity with concrete fixes. Use when the user asks to review, audit, lint, or harden Kubernetes manifests (Deployments, StatefulSets, DaemonSets, Pods, Services, Ingress, RBAC, NetworkPolicy), asks "is this manifest production ready", "why is my pod insecure", or wants a security/best-practice check of Kubernetes YAML or Helm-rendered output.
---

# Kubernetes Manifest Auditor

Review Kubernetes manifests the way an experienced platform engineer would in a pull request: catch the security holes, the reliability gaps, and the "this will page someone at 3am" mistakes, and hand back a fix for each one.

This skill does not deploy anything and does not need cluster access. It reads YAML and reasons about it.

## When to use this

Trigger on requests like:

- "Review / audit / lint this Kubernetes manifest"
- "Is this Deployment production ready?"
- "Harden this pod spec" or "what's wrong with this YAML security-wise"
- "Check these manifests before I apply them"
- Auditing Helm-rendered output (`helm template ... | ...`) or `kustomize build` output

## How to run the audit

1. **Collect the manifests.** They may be one file, several files, or multiple documents in one file separated by `---`. Render Helm/Kustomize first if given a chart (`helm template`, `kustomize build`) so you are auditing the real objects. Parse each document and note its `kind` and `metadata.name`.

2. **Walk every workload and object against the checklist.** The full rubric lives in `references/checklist.md` (load it before auditing). It is grouped into six areas:
   - **Security context** (root user, privilege escalation, capabilities, read-only rootfs, host namespaces)
   - **Supply chain** (image tags vs digests, `imagePullPolicy`, registry trust)
   - **Resource management** (requests/limits, QoS, ephemeral storage)
   - **Reliability** (probes, replicas, `PodDisruptionBudget`, anti-affinity, `terminationGracePeriodSeconds`, rollout strategy)
   - **Networking and access** (`NetworkPolicy`, `Service` exposure, `Ingress` TLS, RBAC scope, ServiceAccount token automounting)
   - **Secrets and config** (secrets in env vs mounted, plaintext data, config drift)

3. **Judge severity honestly.** Use three levels:
   - **Critical** — exploitable or outage-causing as written (privileged container, `hostPath` to `/`, cluster-admin binding, no resource limits on a public workload, secret value in plaintext).
   - **Warning** — a real best-practice gap that will bite under load or in an incident (no probes, `:latest` tag, single replica for a stateful service, automounted SA token that is unused).
   - **Info** — a nice-to-have or a note to confirm intent (missing labels, no `PodDisruptionBudget` on a batch job where it does not matter).

   Do not inflate severity to pad the report, and do not flag something that is genuinely fine for the workload's context (a `Job` legitimately has one "replica"; a debug tool may need extra capabilities). When a finding depends on intent, say so.

4. **Give a fix for every finding.** A finding without a fix is noise. Show the corrected YAML snippet, not just prose. Keep snippets minimal and paste-ready.

## Output format

Lead with a one-line verdict and a severity tally, then the findings grouped Critical → Warning → Info. For each finding:

- **`<kind>/<name>` — short title** `[Critical|Warning|Info]`
- One or two sentences on what is wrong and why it matters.
- A fenced YAML snippet with the fix.

End with a short "looks good" list of the things the manifest already does right (so the report is fair and the reader trusts it), and a one-line next step.

### Example shape

```
Audit: api-gateway Deployment — 2 Critical, 3 Warnings, 1 Info. Not production ready.

## Critical
### Deployment/api-gateway — container runs as root [Critical]
No securityContext, so the container runs as UID 0. A container escape lands as root on the node.
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  seccompProfile:
    type: RuntimeDefault
```
...
```

A full worked example (input manifest and the report it should produce) is in `examples/`.

## Principles

- **Read the whole set, not one object in isolation.** A `Deployment` with no `NetworkPolicy` in the namespace is a finding you only see by looking across documents.
- **Prefer the fix the reader can copy.** Real field names, real values, minimal context.
- **Be specific about blast radius.** "Runs as root" matters more when paired with a `hostPath` mount or a mounted ServiceAccount token. Call out the combination.
- **Stay defensive.** This skill hardens configuration; it does not write exploits.

---

Part of [devops-daily-skills](https://github.com/The-DevOps-Daily/devops-daily-skills) by [DevOps Daily](https://devops-daily.com). For deeper background on the checks, see the Kubernetes and Security sections on [devops-daily.com](https://devops-daily.com/categories/kubernetes).
