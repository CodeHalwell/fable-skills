---
name: containerization-docker
description: Load when writing or reviewing Dockerfiles, debugging slow or bloated image builds, choosing base images, containerizing an app for production, setting up docker-compose for local dev, or handling container security (secrets, non-root, signal handling, registries, multi-arch).
---

# Containerization & Docker

## Core mental model

- **An image is a stack of immutable tarballs** — deleting a file in a later layer doesn't remove it from an earlier one (secrets in any pushed layer = rotate the credential, not just rebuild), and the build cache is a prefix match: one changed byte invalidates everything after it. The entire art of fast builds is ordering least- to most-frequently-changing.
- **A container is a process with namespaces, not a VM** — everything about signals, zombies, and "why won't it stop" follows from your process being PID 1.
- Put `# syntax=docker/dockerfile:1` on line 1 — it pins the current stable Dockerfile frontend independent of the local Docker version (cache/secret mounts, heredocs everywhere).
- **Ship digests, not tags.** Tags are mutable pointers for humans; production pins `@sha256:...`, and Renovate/Dependabot rolls the digest — reproducibility *and* patch cadence rather than one or the other.

## The load-bearing Dockerfile pattern

Manifests → install (with cache mount) → source → build; multi-stage so the runtime has no toolchain; secrets via secret mounts only:

```dockerfile
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build
```

- Cache-mount paths: npm `/root/.npm` · pip/uv `/root/.cache/{pip,uv}` · Go `/go/pkg/mod` + `/root/.cache/go-build` · apt `/var/cache/apt` with `sharing=locked`.
- **Cache mounts live on the builder — ephemeral CI runners start empty.** Pair with `--cache-to/--cache-from` (`type=gha` or a registry ref) **and `mode=max`** (the default `min` exports only final-stage layers), or the mounts silently do nothing across runs.
- Secrets: `RUN --mount=type=secret,...` only. `ARG` lands in history/inspect; `COPY`-then-`rm` lives forever in the COPY layer.
- `apt-get update && install && rm -rf /var/lib/apt/lists/*` in *one* instruction, or the cached update layer serves a stale/404ing index months later.

## Base images and size

Distroless/Wolfi `:nonroot` for compiled binaries; `<lang>-slim` for interpreted stacks; **alpine only after verifying the dependency set is musl-clean** (glibc wheels don't apply → source builds needing gcc, DNS behavioral differences, occasional native-module segfaults — exit 139 on alpine ⇒ suspect musl first). Full images are for build stages only. Size matters via pull time, CVE surface, and registry cost — in that order; remove whole categories (multi-stage, distroless) rather than golfing `rm -rf` lines, and stop when each remaining 100MB has a named owner.

## Security posture (review checklist)

Non-root with a **numeric** UID (k8s `runAsNonRoot` can't verify a named user) · read-only rootfs + `--tmpfs /tmp` · runtime secrets as mounted files over env vars (env leaks into inspect/child processes/crash dumps) · `--cap-drop=ALL` · digest-pinned bases · `.dockerignore` present (minimum: `.git`, dep dirs, `.env*`; it lives next to the *context* root, supports per-Dockerfile `Dockerfile.dockerignore`, and often wants allowlist style: `*` then `!src/`) · `COPY --chown` instead of `chown -R` after COPY (which duplicates the whole tree into a new layer) · no `ADD` for URLs · scan and triage criticals in *your* layers; scanner-zero obsession on distroless leftovers is past the stopping point.

## PID 1

Exec-form always (`CMD ["node","server.js"]`); shell form makes `sh` PID 1 → SIGTERM ignored → every stop is a 10s SIGKILL (an idle app that takes exactly 10s to stop is the diagnostic). Wrapper scripts end in `exec "$@"`. Spawns children → `--init`/tini, or `<defunct>` zombies accumulate. ENTRYPOINT = the executable, CMD = default args; keep the command overridable when the same image legitimately runs server and migrations.

## Runtime contract

One image per commit promoted across environments — config enters at runtime; `ARG ENVIRONMENT` baked in means you tested something other than what ships. Log to stdout unbuffered (`PYTHONUNBUFFERED=1`). `EXPOSE` is documentation; `VOLUME` in a Dockerfile forces anonymous volumes and silently discards later build-stage writes under that path — declare volumes at runtime. `HEALTHCHECK` is used by Docker/compose, **ignored by Kubernetes** (probes rule there), and impossible in distroless (`no shell, no curl`); its *absence* makes compose `condition: service_healthy` a silent no-op.

## Multi-arch

`docker buildx build --platform linux/amd64,linux/arm64 --push` with a `docker-container` driver; current Docker's containerd image store holds multi-platform images locally. Cross-compile via `FROM --platform=$BUILDPLATFORM` + `TARGETARCH` (near-native) or use native ARM runners (`ubuntu-24.04-arm`); QEMU emulation is 5–20× slower — last resort. "exec format error" in prod = laptop-built arm64 image; prod images come from CI with explicit `--platform`, never laptops.

## Compose for local dev

Bind-mount source but **volume-mask dependency dirs** (`[".:/app", "/app/node_modules"]` — host's wrong-arch modules must never shadow the container's); `depends_on` with `condition: service_healthy` + a real healthcheck on the db (default depends_on orders *start*, not readiness); one committed `compose.yaml`, personal tweaks in the override file. Compose approximates prod topology; it doesn't replicate k8s behavior — test probes/limits in a real cluster.

## Debugging fast-reference

Distroless: `kubectl debug`/ephemeral containers, not `exec sh`. Won't start: `docker run -it --entrypoint sh` + `docker inspect --format '{{json .Config}}'`. "Works locally, dies in k8s": reproduce the runtime contract locally — `docker run --read-only --memory=512m --user 10001`. Exit codes: 125 docker itself · 126 not executable · 127 not found (typo'd entrypoint or missing shared lib — `ldd`) · 137 SIGKILL/OOM · 139 segfault (alpine ⇒ musl). `docker commit` as a build strategy means the Dockerfile is now a lie — rebuild from source.

## How an expert thinks through it: "2.8GB Python image, 15-minute builds"

Measure first: `docker history --no-trunc` / `dive`, `--progress=plain` step timings, context-transfer size. Typical findings and fix order: (1) `.dockerignore` (kills the 900MB `.git`+data COPY layer and spurious cache busts); (2) reorder manifests-then-source with a pip cache mount (code edits rebuild in seconds); (3) `-slim` base + build stage for anything needing gcc. Rejected: alpine (musl source-builds make it slower and not smaller); squashing layers (hides bloat, destroys caching); bigger CI runners (money to avoid a 10-line fix). Expected: ~1.2GB (torch is irreducible), <1 min warm builds.

## Verification / self-check

- `docker history` — name why each layer >50MB exists; any secret in any layer ⇒ revoke the credential.
- Touch one source file, rebuild: everything before `COPY . .` says CACHED, or ordering is wrong.
- `docker stop` <1s idle; `ps` for zombies; `whoami` non-root; still works `--read-only`.
- `docker inspect --format '{{.Architecture}}'` matches prod before shipping.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 14 baseline, 0 partial, 0 delta — Opus cold reproduced secret mounts (incl. why ARG and COPY+rm fail), cache ordering, CI cache-export pairing (incl. `mode=max`, which the old file lacked and has been added), musl/alpine reasoning, PID-1/10s-stop diagnosis, `--chown`, HEALTHCHECK-vs-k8s, multi-arch/QEMU costs, VOLUME/EXPOSE semantics, exit codes, and the full 2.8GB walkthrough with the same rejected alternatives.
- File restructured to a compact reference sheet; all explanatory prose cut. Residual value: the review checklists and a handful of narrow specifics (per-Dockerfile `Dockerfile.dockerignore`, allowlist ignore style, `sharing=locked` apt mounts, containerd image store default) rather than any concept Opus lacks.
