---
name: kubernetes-operations
description: Load when deploying to, debugging, or designing workloads for Kubernetes — writing manifests, choosing workload types, diagnosing pod failures (Pending/CrashLoopBackOff/OOMKilled), setting resource requests/limits, configuring probes or autoscaling, or deciding whether something belongs on k8s at all.
---

# Kubernetes Operations

## Core mental model

- **Kubernetes is a reconciliation engine.** Every mystery resolves to "which controller owns this object, and what does it think desired state is?" — why `apply` returns before anything happens, why deleted pods come back, why fixes go to the owner (edit the Deployment; the ReplicaSet stomps pod edits).
- **The scheduler bets on requests; the kernel enforces limits.** A cluster can be 30% utilized and unschedulable (requests over-provisioned) or 95% and healthy — different axes, reason separately.
- **Pods are cattle by construction**; if the app can't tolerate a random pod death right now, the fix is app architecture (graceful shutdown, PDB, StatefulSet if identity matters), not "don't touch my pod."
- **Labels/selectors are the only glue**; `kubectl get endpointslices` is the truth about what a Service routes to.

## Workload selection

Runs to completion → Job/CronJob (never a Deployment — `restartPolicy: Always` restarts *successful* exits into CrashLoopBackOff by design). Replicas non-interchangeable (own PV, stable DNS, ordered start) → StatefulSet — the test is interchangeability, not "has state"; local disk cache is `emptyDir`, still a Deployment. Every node → DaemonSet. Otherwise → Deployment (~90%). Databases: prefer managed or a mature operator (CloudNativePG, Strimzi) over hand-rolled StatefulSets.

## Debugging: describe + events first

`kubectl describe pod` (Events = the controller's diary) + `kubectl get pod -o wide` resolve 80% of cases. Branch: **Pending** = scheduler (insufficient resources per node allocatable, taints/affinity, unbound PVC/zone mismatch, or autoscaler can't fit the shape — check its logs). **CrashLoopBackOff** = the app; `kubectl logs --previous` (the most-forgotten flag — restarts >0 → `--previous` first), exit codes 137=SIGKILL (OOM or liveness), 139=segfault, 127/126=bad entrypoint; empty logs + missing ConfigMap/Secret = `CreateContainerConfigError`. **Running-not-Ready** = readiness probe; empty endpointslices + existing pods = selector mismatch. **Terminating forever** = a finalizer nobody processes. If the story doesn't match `kubectl get events --sort-by=.lastTimestamp`, it's a guess — selectors, resources, and probes are common; the CNI is rare.

## Requests and limits

Memory: request = limit (incompressible; overcommit resolves via someone's OOM kill). CPU: request yes, **limit usually no** — CFS quota stalls the pod for the rest of each 100ms period even on an idle node; `container_cpu_cfs_throttled_periods_total` is the smoking gun for mystery tail latency. CPU limits are justified only for static-CPU-manager pinning, strict multi-tenancy, or spin-loop protection. Requests come from measured p99 over a representative week — **in-place pod resize went GA in Kubernetes 1.35 (late 2025), not 1.34 as commonly misremembered**, and VPA's `InPlaceOrRecreate` mode now applies recommendations without restarts; VPA-in-recommendation-mode remains the expert default measuring instrument.

## Probes

Readiness = "route to me?"; be careful checking *shared* dependencies — a DB blip taking every pod unready converts partial degradation into total outage; prefer failing requests fast. Liveness = "restart cures me" — most apps shouldn't have one; a liveness probe with dependencies turns every DB blip into a fleet-wide restart storm, and one that fails under load (GC, thread exhaustion) turns "slow" into "restart storm → colder → slower." Startup probe replaces giant `initialDelaySeconds` for slow boots. Shutdown: SIGTERM and endpoint removal race *in parallel* — serve while draining, or `preStop: sleep 5–10` so endpoint propagation wins.

## Autoscaling layers

HPA scales count on metrics relative to *requests* (no requests → `<unknown>` and nothing happens; requests far below baseline → permanently >100% → pinned at maxReplicas). Don't run HPA and VPA on the same metric. Node layer: Karpenter is the EKS default (and underlies AKS Node Auto Provisioning) — right-sized nodes in ~1 min and active *consolidation*, so your PDBs and graceful shutdown had better work (`karpenter.sh/do-not-disrupt` for the exceptions); Cluster Autoscaler is the slower, no-surprises multi-cloud choice. HPA reacts in seconds, nodes in minutes — overprovisioning placeholder pods (low `priorityClassName`) if spike latency matters.

## Networking

**ingress-nginx maintenance ended March 2026 — running it now means unpatched CVEs** (post-IngressNightmare); the Ingress API itself is frozen. New work: Gateway API (Envoy Gateway, Cilium, cloud-native implementations); `ingress2gateway` converts; the role split (infra owns Gateway, app teams own HTTPRoutes) is the design win — use it that way. Cross-namespace DNS needs the namespace qualifier ("works in staging, fails in prod" is often a missing `.namespace` in a URL). Default network policy is allow-everything.

## Operators, and what NOT to run on k8s

Use mature community operators freely; *writing* one is a last resort (you're maintaining a distributed-systems controller). Red flag: a CRD whose controller just templates objects with no ongoing reconciliation — a Helm chart in a trench coat. Keep off k8s: databases you can't operate, uncheckpointable never-interrupt work (upgrades and consolidation *will* interrupt it), singleton legacy apps, and tiny total footprints — under the N-2/14-month support policy you're upgrading every ~4 months; if the company fits in 4 VMs, the control-plane tax exceeds the benefit.

## Failure modes & pitfalls (checklist)

- Editing the pod instead of its owner; `kubectl apply` fighting GitOps/an operator (find the other writer via `--show-managed-fields` before "fixing" harder).
- One-shot script in a Deployment → CrashLoopBackOff with no error in sight.
- **Same-tag image pushes don't roll out** (spec unchanged; `IfNotPresent` serves stale cache even on reschedule) — unique tags or digests, never mutated tags.
- Deployment `spec.selector` is immutable — plan labels before first apply; fix = delete/recreate with `--cascade=orphan`.
- Env vars from ConfigMaps/Secrets never update running pods; mounted files do (~1 min) **except `subPath` mounts, which never update**; rotate via checksum annotation on the pod template or a reloader.
- PDB `maxUnavailable: 0` (or `minAvailable` = replicas) makes eviction impossible: drains hang, upgrades stall, Karpenter can't consolidate — and it's the *platform team* that gets paged. Single-replica workloads get no restrictive PDB.
- Cargo-cult `{cpu: 100m, memory: 128Mi}` on a JVM = startup OOM or 20× under-request; JVM/Node must know the cgroup budget (`MaxRAMPercentage`, `--max-old-space-size`).
- `ephemeral-storage` unset: a chatty container triggers node DiskPressure and evicts *neighbors*.
- CronJob default `concurrencyPolicy: Allow` overlaps slow runs — set `Forbid`/`Replace` + `startingDeadlineSeconds`.
- Sidecars as bare extra containers block Jobs and race the app at startup — native sidecars (init container + `restartPolicy: Always`, GA 1.33) fix ordering and Job completion.
- `kubectl top` lies about OOM proximity — use `container_memory_working_set_bytes` max-over-time; a kill *under* the limit means node-level pressure (different fix: evictions, system-reserved).

## Worked micro-example — the production-shaped Deployment (fields that matter)

```yaml
spec:
  selector: { matchLabels: { app: api } }        # immutable — choose once
  template:
    spec:
      terminationGracePeriodSeconds: 45           # > preStop + drain
      containers:
      - name: api
        image: registry.example.com/api@sha256:9f8e...   # digest, not tag
        resources:
          requests: { cpu: 250m, memory: 512Mi }  # measured p99
          limits: { memory: 512Mi }               # memory only; no CPU limit
        startupProbe: { httpGet: { path: /healthz, port: 8080 }, failureThreshold: 30, periodSeconds: 5 }
        readinessProbe: { httpGet: { path: /ready, port: 8080 }, periodSeconds: 5 }
        # no livenessProbe until someone can justify one aloud
        lifecycle: { preStop: { exec: { command: ["sleep", "8"] } } }
---
kind: PodDisruptionBudget
spec: { maxUnavailable: 1 }                       # evictable, so drains/consolidation work
```

## How an expert thinks through it: "deploy went out, p99 tripled"

Diff the *manifests*, not just app code: the "hardening" commit added a CPU limit and a liveness probe on `/health`. p50 unchanged → deprioritize "new code is slower." `container_cpu_cfs_throttled_periods_total` spiking = mechanism one. Restarts 3–7 = liveness `/health` hits the DB with a 1s timeout; throttle-induced slowness kills pods and dumps load on neighbors = mechanism two. Fix: drop the CPU limit (raise the request to measured p99), liveness to a no-dependency ping or delete it, readiness returns 200-degraded on DB slowness. Rejected: scaling replicas (throttling is per-pod), raising the limit (throttles later, still throttles), removing probes wholesale (readiness is load-bearing for rollouts).

## Verification / self-check

- `kubectl apply --dry-run=server` + `kubectl diff` before anything shared.
- Every Deployment: memory request=limit, CPU request no limit, readiness probe, SIGTERM handling, PDB if replicas ≥2, no unjustified liveness.
- Kill one pod on purpose and watch traffic: zero errors = shutdown and probes are right. If you haven't done this, you don't know.
- A diagnosis names the controller whose desired/observed mismatch explains the symptom, and the Events timeline agrees.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 12 baseline (cut/compressed), 2 partial (sharpened), 0 delta.
- Opus cold nails: no-CPU-limits position + throttling mechanism, liveness anti-patterns, SIGTERM/endpoint race, same-tag no-rollout, ConfigMap/subPath propagation, PDB drain-blocking, Karpenter consolidation, native sidecars (GA 1.33), CronJob overlap, exit codes and `--previous`.
- Sharpened: in-place pod resize **GA version is 1.35, late 2025** (Opus confidently says GA 1.34 — wrong release), and ingress-nginx's concrete **March 2026 end-of-maintenance** date (Opus knows "retired," not that running it today is an unpatched-CVE liability).
