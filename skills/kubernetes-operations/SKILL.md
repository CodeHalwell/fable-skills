---
name: kubernetes-operations
description: Load when deploying to, debugging, or designing workloads for Kubernetes — writing manifests, choosing workload types, diagnosing pod failures (Pending/CrashLoopBackOff/OOMKilled), setting resource requests/limits, configuring probes or autoscaling, or deciding whether something belongs on k8s at all.
---

# Kubernetes Operations

## Core mental model

- **Kubernetes is a reconciliation engine, not a deployment tool.** Every object is a *desired state* record; controllers run infinite loops comparing desired vs observed state and nudging reality toward the spec. This explains everything confusing: why `kubectl apply` returns before anything happens (you wrote a record, that's all), why deleted pods come back (a controller's desired state still says 3 replicas), why fixes must go to the *owner* (edit the Deployment, not the pod — the ReplicaSet will stomp your pod edit), and why the right debugging question is always "which controller owns this object, and what does it think the desired state is?"
- **The scheduler bets on requests; the kernel enforces limits.** `requests` are a scheduling-time claim used for bin-packing — never measured against actual usage after placement. `limits` are runtime enforcement: cgroup CPU throttling for CPU, OOM-kill for memory. A cluster can be 30% utilized and unschedulable (requests over-provisioned), or 95% utilized and healthy. These are different axes; reason about them separately.
- **Pods are cattle by construction.** Any pod can be killed at any time by node drain, eviction, preemption, or bin-packing (consolidation). If your app can't tolerate a random pod death right now, the fix is app architecture (graceful shutdown, PDBs, StatefulSet if identity matters), not "please don't touch my pod."
- **Labels and selectors are the only glue.** Service→pod, Deployment→ReplicaSet→pod, NetworkPolicy→pod: all label matching. A huge fraction of "traffic isn't reaching my pod" bugs are selector typos. `kubectl get endpointslices` is the truth about what a Service actually routes to.
- **YAML is not the interface; the API is.** `kubectl explain deployment.spec.strategy`, `kubectl get -o yaml`, and events are how you interrogate the system. Fields you didn't set have defaults that matter (e.g., `terminationGracePeriodSeconds: 30`, `restartPolicy: Always`).

## Workload resource selection — the reasoning chain

Ask in order:

1. **Does it run to completion?** → `Job` (or `CronJob` for schedules). Never a Deployment with a script that exits — you'll get CrashLoopBackOff by design, because `restartPolicy: Always` restarts *successful* exits too.
2. **Does each replica need stable identity** — its own persistent volume, a stable DNS name, or ordered startup (databases, Kafka, anything with a notion of "node 0")? → `StatefulSet`. The test isn't "has state" but "are replicas interchangeable?" A stateless API writing to RDS is a Deployment; replicas are fungible.
3. **Must it run on every node** (log shipper, node agent, CNI)? → `DaemonSet`.
4. **Otherwise** → `Deployment`. This is the default; ~90% of workloads.

What changes the answer: "our app caches to local disk" does *not* make it a StatefulSet (use `emptyDir`; cache is disposable). "Replicas coordinate via each other's hostnames" does. If you're reaching for StatefulSet for a database, first ask whether a managed DB or an operator (CloudNativePG, Strimzi) should own that complexity instead.

## The debugging decision tree

Start every pod investigation with the same two commands — they resolve 80% of cases:

```bash
kubectl describe pod <pod>        # Events section = the controller's diary
kubectl get pod <pod> -o wide     # phase, restarts, node, IP
```

Then branch on the symptom:

**Pending** — the scheduler can't place it. Nothing is wrong with your app; it never started.
- `kubectl describe pod` → Events. Look for `FailedScheduling: 0/12 nodes available: insufficient cpu` (requests too big or cluster full — check `kubectl describe nodes | grep -A5 "Allocated resources"`), `didn't match node selector/affinity`, `had untolerated taint`, or `unbound PersistentVolumeClaim` (check `kubectl get pvc` — a WaitForFirstConsumer StorageClass or a zone mismatch between the PV and schedulable nodes).
- If the cluster autoscaler/Karpenter should have added a node, check its events/logs — a pod whose requests fit no *possible* node shape stays Pending forever.

**CrashLoopBackOff** — the container starts, exits nonzero, and kubelet backs off exponentially (max 5 min). The app is the problem.
- `kubectl logs <pod> --previous` — the crashed container's logs, not the current attempt's. This is the single most-forgotten flag.
- Exit code from `kubectl describe pod` (`Last State: Terminated, Exit Code:`): 1 = app error, 137 = SIGKILL (OOM or liveness timeout), 139 = segfault, 127/126 = bad command/entrypoint.
- If logs are empty: bad command/args, missing env var making it die pre-logging, or a missing ConfigMap/Secret mount (that one shows as `CreateContainerConfigError` instead). Reproduce with `kubectl run debug -it --image=<image> -- sh` or `kubectl debug`.

**OOMKilled** — `describe` shows `Reason: OOMKilled`, exit code 137. The container exceeded its *memory limit* (or the node ran out and it lost the eviction/OOM lottery).
- Distinguish: steady growth to the limit = leak or genuinely undersized; instant kill at startup = limit below baseline footprint; kills under load spikes = right-size for peak, not average.
- Check actual usage vs limit: `kubectl top pod` (needs metrics-server) or your metrics stack (`container_memory_working_set_bytes` — that's what the OOM decision uses, not RSS).
- JVM/Node gotcha: the runtime must know its budget. Modern JVMs respect cgroup limits (`MaxRAMPercentage`); Node needs `--max-old-space-size`. A 4 GiB-default heap in a 512 MiB container is a guaranteed 137.

**Running but not Ready** — pod is up, readiness probe failing, so the Service won't route to it.
- `kubectl describe pod` → `Readiness probe failed: ...` with the actual HTTP code/connection error. Then: is the probe port/path right? Does the app listen on 0.0.0.0 or only localhost? Is the dependency the readiness check pings actually down (in which case not-ready is *correct behavior*)?
- Service-level check: `kubectl get endpointslices -l kubernetes.io/service-name=<svc>` — empty means selector mismatch or nothing ready.

**ImagePullBackOff** — wrong image name/tag, missing `imagePullSecrets`, or registry auth/network. `describe` events contain the exact registry error.

**Terminating forever** — a finalizer nobody is processing (`kubectl get pod -o yaml | grep -A3 finalizers`) or a node that died; last resort `--force --grace-period=0`, but understand a stuck finalizer means some controller is broken.

## Requests and limits — current reasoning (as of 2026)

- **Always set memory requests and limits, and set them equal.** Memory is incompressible — overcommit resolves via OOM kills of *someone*, often not the overcommitter. `requests == limits` gives predictable eviction behavior.
- **Always set CPU requests; usually omit CPU limits.** This is now the mainstream position, not a hot take. CPU is compressible: without limits, contention is resolved proportionally by CPU *requests* (cgroup cpu.weight), so a noisy neighbor can only steal *idle* cycles — your request is still guaranteed. CPU limits add hard throttling: a pod hitting its quota is stalled for the rest of the 100ms CFS period even on an idle node, which manifests as mysterious tail latency (`container_cpu_cfs_throttled_periods_total` is the smoking gun).
- When CPU limits *are* justified: Guaranteed QoS for static CPU-manager pinning (latency-critical, core-isolated workloads), strict multi-tenant platforms where predictability beats utilization, benchmarking to simulate constrained capacity, and pathological spin-loop protection.
- Setting requests: measure (p99 of actual usage over a representative week), don't guess. In-place pod resize is GA (Kubernetes 1.35, late 2025), and VPA can now apply recommendations without restarting pods (`InPlaceOrRecreate` mode) — use VPA in recommendation mode as a measuring instrument even if you never let it actuate.

## Probes done right

- **Readiness** = "send me traffic?" Checked continuously; failing removes the pod from Service endpoints but does *not* restart it. Should reflect ability to serve: app initialized, critical local resources up. Be careful checking *shared* dependencies: if the database blips and every pod's readiness check pings the DB, the whole fleet goes unready simultaneously and you convert a partial degradation into a total outage. Prefer failing requests fast over going unready for shared-dependency failures.
- **Liveness** = "am I irrecoverably wedged? kill me." A liveness restart is a kill -9 with no diagnosis. Most apps don't need one. Never make liveness check dependencies; never make it the same endpoint as readiness. A liveness probe that fails under load (heavy GC, thread-pool exhaustion) turns "slow" into "restart storm": slow pod → probe timeout → restart → cold cache → slower → more restarts. If you must have one, probe the barest "event loop responds" endpoint with generous `timeoutSeconds` and `failureThreshold`.
- **Startup** = "don't judge me while booting." For slow-starting apps (JVM warmup, large model load), a startup probe gates liveness/readiness until first success, replacing the old hack of huge `initialDelaySeconds`. `failureThreshold: 30, periodSeconds: 10` = up to 5 minutes to boot.
- Graceful shutdown is a probe-adjacent must: on pod deletion, endpoint removal and SIGTERM race *in parallel* — traffic can arrive after SIGTERM. Handle SIGTERM by continuing to serve while draining, or add a `preStop: sleep 5-10` hook to let endpoint propagation win the race.

## Autoscaling layers (as of 2026)

Four layers that must be reasoned about together:

- **HPA** scales replica *count* on metrics (CPU utilization relative to *requests* — another reason requests must be honest; or custom/external metrics like queue depth, e.g. via KEDA for scale-to-zero and event sources).
- **VPA** adjusts per-pod requests. Historically restart-on-resize made it unpopular; with in-place resize GA this is changing, but recommendation-mode-first is still the expert default. Don't run HPA and VPA on the same metric (both act on CPU → feedback loop); HPA-on-CPU + VPA-on-memory is fine.
- **Node layer:** Karpenter is the default choice on AWS EKS (and underlies AKS Node Auto Provisioning): provisions right-sized nodes directly from the cloud API in under a minute, no node-group ceremony, and actively *consolidates* (kills and repacks nodes to cut cost — your PDBs and graceful shutdown had better work). Cluster Autoscaler remains the safe, multi-cloud choice elsewhere: scales predefined node groups, slower, no repacking surprises.
- Interplay trap: HPA reacts in seconds, node provisioning in ~minutes (CAS) or ~1 min (Karpenter). Under a spike, new replicas go Pending until nodes arrive — keep headroom (overprovisioning placeholder pods with low `priorityClassName`) if spike latency matters.

## Networking model (as of 2026)

- Every pod gets a routable IP; Services are stable virtual IPs load-balancing over ready endpoints. `ClusterIP` for internal, `LoadBalancer` for direct external exposure, `NodePort` almost never directly.
- **Ingress is frozen; Gateway API is the successor.** ingress-nginx was retired (maintenance ended March 2026 — running it now means unpatched CVEs), and the Ingress API accepts no new features. For new work use Gateway API (`GatewayClass`/`Gateway`/`HTTPRoute`) with an implementation like Envoy Gateway, Cilium, or your cloud's native gateway; `ingress2gateway` converts existing manifests. Gateway API's role split (infra team owns Gateway, app teams own HTTPRoutes in their namespaces) is the design win — use it that way.
- DNS: `<svc>.<namespace>.svc.cluster.local`; cross-namespace calls need the namespace qualifier — "works in staging, fails in prod" is often a missing namespace in a URL.
- Default network policy is *allow everything*; NetworkPolicy objects are additive deny→allow and require CNI support.

## Operators and CRDs — judgment

An operator is worth it when the operational knowledge is genuinely complex, encodable, and repeated: failover choreography, backup/restore, version upgrades of stateful systems. Use mature community operators (CloudNativePG, Strimzi, cert-manager, ESO) freely. *Writing* your own is a last resort: you're signing up to maintain a distributed-systems controller with idempotent reconciliation, conflict handling, and upgrade paths. If a Helm chart + Job can do it, don't write an operator. Red flag in reviews: a CRD whose controller just templates other objects with no ongoing reconciliation logic — that's a Helm chart wearing a trench coat.

## What NOT to run on k8s

- **Databases you aren't expert in operating** — the operator helps but doesn't absolve you of understanding failover; managed services win below large scale or strong data-locality needs.
- **Anything that must never be interrupted** without checkpointing — cluster upgrades and consolidation will interrupt it.
- **Singleton legacy apps** with no health semantics: a VM is honestly simpler.
- **Tiny total footprint**: if your whole company fits in 4 VMs, the control-plane tax (upgrades every ~4 months under the N-2/14-month support policy, CNI/CSI/ingress churn) exceeds the benefit. Cloud Run / ECS / Fly-class platforms cover the middle ground.

## How an expert thinks through it: "deploy went out, latency p99 tripled"

Rollout event correlates → check `kubectl rollout history` and diff the manifests, not just app code. Diff shows someone "added best practices": CPU limit `500m` and a liveness probe on `/health`. Hypotheses: (a) new code is slower — but p50 unchanged, only tail; deprioritize. (b) CPU throttling — check `container_cpu_cfs_throttled_periods_total`: spiking during request bursts. That's mechanism one. (c) Restarts? `kubectl get pods` shows `RESTARTS: 3-7` — liveness `/health` calls the DB with a 1s timeout, and under throttle-induced slowness it times out, killing pods and dumping their load onto neighbors. Two interacting failures, both from the "hardening" commit. Fix: drop CPU limit (keep the request, raised to measured p99), point liveness at a no-dependency ping endpoint or delete it, keep readiness on `/health` but return 200-with-degraded rather than failing on DB slowness. Rejected along the way: scaling replicas (treats symptom, throttling is per-pod), raising the CPU limit to 2 cores (still throttles at bursts, just later), removing probes wholesale (readiness is load-bearing for rollouts).

## Verification / self-check

Before declaring a manifest or diagnosis done:
- `kubectl apply --dry-run=server -f .` (server-side catches admission/schema errors client-side misses); `kubectl diff` before applying to anything shared.
- For a diagnosis: can you name the *controller* whose desired/observed mismatch explains the symptom, and does the Events timeline agree? If your story doesn't match `kubectl get events --sort-by=.lastTimestamp`, it's a guess.
- Every Deployment ships with: memory request=limit, CPU request, readiness probe, graceful SIGTERM handling, PDB if replicas ≥ 2, and no liveness probe you can't justify aloud.
- Kill one pod on purpose (`kubectl delete pod`) and watch traffic: zero errors = shutdown and probes are right. If you haven't done this, you don't know.
- Stopping rule: the incident is explained when symptom, events, metrics, and the diff all tell one story. If you're on your third "maybe it's the CNI" theory without evidence, go back to `describe` and events — exotic causes are rare; selectors, resources, and probes are common.
