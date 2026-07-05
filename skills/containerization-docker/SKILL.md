---
name: containerization-docker
description: Load when writing or reviewing Dockerfiles, debugging slow or bloated image builds, choosing base images, containerizing an app for production, setting up docker-compose for local dev, or handling container security (secrets, non-root, signal handling, registries, multi-arch).
---

# Containerization & Docker

## Core mental model

- **An image is a stack of immutable tarballs plus metadata.** Each Dockerfile instruction that changes the filesystem produces a layer; a container is those layers union-mounted with a thin writable layer on top. Consequences: deleting a file in a later layer doesn't remove it from the image (it's still in the earlier layer — this is why secrets in any layer are unrecoverable mistakes), layers are shared between images (base image pulled once), and "image size" is the sum of all layers, not the final filesystem.
- **The build cache is a prefix match.** BuildKit reuses a cached layer iff the instruction and its inputs (for `COPY`/`ADD`, a content hash of the copied files) are identical *and every prior layer was cached*. One changed byte invalidates everything after it. The entire art of fast builds is ordering instructions from least- to most-frequently changing.
- **A container is a process with namespaces and cgroups, not a VM.** No init system, no daemon manager, one foreground process (PID 1) whose exit ends the container. Everything about signals, zombies, and "why won't it stop" follows from your process being PID 1.
- **BuildKit is the builder** (default since Docker Engine 23.0; invoked as `docker build`/`docker buildx build`). Put `# syntax=docker/dockerfile:1` on line 1 — it pins the latest stable Dockerfile frontend independent of the local Docker version, enabling cache mounts, secret mounts, and heredocs everywhere.
- **Ship digests, not tags.** A tag (`:v1.2.3`, worse `:latest`) is a mutable pointer; a digest (`@sha256:...`) is content-addressed truth. Production deploys pin digests; tags are for humans.

## Dockerfile design — the reasoning chain

Questions an expert asks, in order:

1. **What changes most often?** Source code. **Least often?** OS packages, then language dependencies (lockfile-driven). Order the Dockerfile accordingly: base → system packages → *dependency manifests only* → install deps → copy source → build. The load-bearing pattern:

```dockerfile
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build
```

Copying all source before `npm ci` means every code edit re-downloads the world. This single mistake dominates slow builds in the wild.

2. **What must exist at runtime?** Only the built artifact and its runtime deps. Everything else — compilers, dev dependencies, git, curl-for-downloading — belongs in an earlier *stage*. Multi-stage is the default, not an optimization:

```dockerfile
FROM golang:1.24 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build CGO_ENABLED=0 go build -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot
ENTRYPOINT ["/app"]
```

3. **What persists across builds but not in the image?** Package-manager caches. `RUN --mount=type=cache,target=...` (npm: `/root/.npm`, pip: `/root/.cache/pip`, Go: `/go/pkg/mod` + `/root/.cache/go-build`, apt: `/var/cache/apt` with `sharing=locked`) — the cache survives between builds without ever entering a layer. This is the cheapest 5–10x speedup available.

4. **What must never enter any layer?** Credentials. `RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci` + `docker build --secret id=npmrc,src=$HOME/.npmrc`. The secret exists only during that RUN, in tmpfs, in no layer. `ARG TOKEN` is **not** safe: build args are embedded in image history (`docker history`, `docker inspect`). Neither is `COPY creds && RUN use && RUN rm creds` — the file lives forever in the COPY layer.

## Base image selection

| Choice | Use when | Watch out |
|---|---|---|
| `distroless` (static/base/cc, `:nonroot` variants) or Chainguard/Wolfi | Compiled binaries (Go/Rust), production default when you can | No shell — debug with `kubectl debug`/ephemeral containers, not `exec sh` |
| `<lang>-slim` (e.g. `python:3.13-slim`, Debian-based) | Interpreted languages in production | Fine default; add only the apt packages you can name a reason for |
| `alpine` | You need a shell + small size and your stack is musl-clean | **musl gotchas**: Python wheels historically needed musl builds (source compiles when missing), glibc-linked binaries segfault, musl DNS behaves differently (historically no TCP fallback; resolv.conf edge cases), slower malloc for some workloads. Don't pick alpine for Python/Node by reflex — `-slim` is usually the better small option |
| Full `ubuntu`/`debian`/`<lang>:latest` | Build stages, dev containers | Never as a production final stage |

Size reasoning: size matters via pull time (cold-start, autoscaling latency), CVE surface (scanners flag every package present), and registry cost — in that order. But a 200 MB image that works beats a 40 MB one that segfaults on musl. Optimize size *after* correctness, and prefer removing whole categories (multi-stage, distroless) over golfing individual `RUN rm -rf` lines.

## Security posture

- **Non-root by default**: `USER` with a numeric UID (`USER 10001` or distroless `:nonroot` = 65532). Numeric matters: Kubernetes `runAsNonRoot` cannot verify a *named* user. Create the user in the Dockerfile; don't rely on runtime flags someone must remember.
- **Read-only rootfs**: design the app to write only to mounted volumes/`/tmp`, then run with `--read-only --tmpfs /tmp` (k8s: `readOnlyRootFilesystem: true`). This turns whole exploit classes (dropped binaries, modified configs) into ENOENT.
- **Secrets**: build-time → secret mounts (above). Runtime → env vars from a secret manager, or better, mounted files (env vars leak into `docker inspect`, child processes, and crash dumps). Never bake either.
- Drop capabilities (`--cap-drop=ALL` plus adds you can justify), no `--privileged` outside of genuinely node-level tooling.
- Pin base images by digest in production Dockerfiles and let a bot (Renovate/Dependabot) roll them — you get reproducibility *and* patch cadence.

## The PID-1 problem

Your process is PID 1. PID 1 has two special properties: default signal dispositions don't apply (unhandled SIGTERM is *ignored*, not fatal), and it inherits all orphaned children (must reap zombies).

- **Exec-form always**: `CMD ["node", "server.js"]`. Shell form (`CMD node server.js`) makes `/bin/sh -c` PID 1; your app becomes a child that never receives SIGTERM, so every stop is a 10-second wait then SIGKILL — dropped requests on every deploy.
- If the app spawns children or you can't audit its signal handling, add an init: `docker run --init`, or `ENTRYPOINT ["tini", "--", ...]` in the image, or Kubernetes `shareProcessNamespace`/proper handling. Symptom of missing reaping: `<defunct>` processes accumulating.
- Shell wrapper scripts must end in `exec "$@"` — without `exec`, the shell stays PID 1 and swallows signals.
- Test it: `docker stop` should return in <1s for an idle well-behaved app. If it takes exactly 10s, SIGTERM is being ignored — find out why before shipping.

## Build context hygiene and healthchecks

- `.dockerignore` is mandatory: at minimum `.git`, `node_modules`/venvs, build outputs, `.env*`, secrets. Without it, `COPY . .` invalidates cache on every `.git` change *and* can copy secrets into a layer. A multi-GB "sending build context" message means this file is missing or wrong.
- `HEALTHCHECK` in the Dockerfile is used by Docker/compose (`depends_on: condition: service_healthy`) but **ignored by Kubernetes** — k8s uses probes. Write healthchecks for compose-based dev/simple hosts; don't assume they do anything in k8s. Note distroless images can't run `CMD curl ...` — no shell, no curl; use a tiny healthcheck binary or skip it.

## Multi-arch (as of 2026)

Apple-silicon dev + x86 prod (or ARM prod for cost) makes this table stakes. `docker buildx build --platform linux/amd64,linux/arm64 --push -t reg/app:tag .` with a `docker-container` driver builds both and pushes a manifest list; clients pull their native arch automatically. Current Docker uses the containerd image store by default on new installs, which can hold multi-platform images locally. In Dockerfiles use `FROM --platform=$BUILDPLATFORM` + `TARGETOS`/`TARGETARCH` args for cross-compiling stages (Go/Rust cross-compile fast; QEMU-emulated native builds are 5–20x slower — prefer cross-compilation or native ARM CI runners, e.g. GitHub's `ubuntu-24.04-arm`). Pitfall: `docker build` on an M-series Mac produces arm64 by default; "exec format error" in prod means you shipped the wrong arch — pin `--platform` in CI and never build prod images on laptops.

## Compose for local dev

Compose (v2, `docker compose`) is the local-dev tool, not a production orchestrator. Expert setup: bind-mount source for hot reload but **volume-mask dependency dirs** (`volumes: [".:/app", "/app/node_modules"]`) so host node_modules (wrong arch/OS) never shadows the container's; `depends_on` with `condition: service_healthy` because default depends_on only orders *start*, not readiness — the classic "app crashes because postgres isn't accepting connections yet"; one `compose.yaml` committed, `compose.override.yaml` for personal tweaks. Resist parity theater: compose approximates prod topology, it doesn't replicate k8s behavior; test probes and resource limits in a real cluster.

## Registry and tagging strategy

- Tag every CI build with the immutable git SHA (`app:sha-abc1234`); deploy manifests reference the digest CI resolved (`app@sha256:...`). Human-friendly tags (`:v1.4.2`, `:main`) are aliases layered on top.
- Never re-push a changed image under an existing version tag; never deploy `:latest` anywhere that matters — you lose the ability to know what's running or to roll back to "the same thing."
- Set a registry retention policy from day one (untagged manifests, SHA tags >N months) or the registry bill becomes its own project.
- Sign/attest if your platform supports it (cosign, GitHub artifact attestations) — cheap now, painful to retrofit.

## How an expert thinks through it: "our Python image is 2.8 GB and builds take 15 minutes"

First measure, don't guess: `docker history --no-trunc app:latest` (or `dive`). Findings: base `python:3.12` (~1 GB, full Debian with gcc), a 900 MB layer from `COPY . .` (the `.git` dir plus a `data/` folder — no `.dockerignore`), and a 600 MB pip-install layer including torch. Build time: every commit invalidates from `COPY . .`, which sits *before* `pip install`, so torch re-downloads each build. Plan, in impact order: (1) `.dockerignore` with `.git`, `data/`, `.venv` — kills 900 MB and stops spurious cache busts; (2) reorder: `COPY requirements.txt` → `RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt` → `COPY . .` — code edits now rebuild in seconds; (3) switch base to `python:3.12-slim` with a build stage for anything needing gcc. Considered and rejected: alpine (torch/numpy on musl = source builds or missing wheels — worse, not better); squashing layers into one RUN (defeats caching, saves little post-cleanup); "just use a bigger CI runner" (spends money to avoid a 10-line fix). Expected result: ~1.2 GB (torch is irreducible), <1 min warm builds. Stopping rule: once the image is slim-based, cache-ordered, and each remaining 100 MB has a named owner (torch, CUDA libs), further golfing is waste.

## Verification / self-check

- `docker history app:tag` — can you name why each layer >50 MB exists? Any layer containing a secret means rebuild from scratch, revoke the credential (it's compromised, not just untidy).
- Cache check: touch one source file, rebuild — everything before `COPY . .` must say `CACHED`. If dependency install reruns, ordering is wrong.
- Runtime check: `docker run` it — then `docker stop` (fast exit?), `ps` inside for zombies if it spawns children, `whoami`/`id` (non-root?), and with `--read-only --tmpfs /tmp` (still works?).
- Arch check before shipping: `docker inspect --format '{{.Architecture}}'` matches prod.
- Scan (`trivy image`/`docker scout`) and triage: fix criticals in *your* layers by bumping deps, base-image CVEs by bumping the base digest; a scanner-zero obsession on distroless leftovers is past the stopping point.
