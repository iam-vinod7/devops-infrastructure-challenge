# Deployment Log — DevOps Infrastructure Challenge

## Architecture
GitHub repo → Docker build → k3s (containerd) → Kubernetes (namespace: `devops-demo`)

Flask backend (NodePort 30080) → PostgreSQL (ClusterIP 5432)

## Deployment Steps

1. Cloned repo to `/root/devops-challenge/app`
2. Installed k3s: `curl -sfL https://get.k3s.io | sh -`
3. Started Docker daemon: `systemctl start docker && systemctl enable docker`
4. Built Docker image: `docker build -t devops-backend:1.0 ./app`
5. Imported image into k3s's containerd (k3s does not share Docker's image store):
6. Applied manifests in order: `namespace.yaml` → `postgres.yaml` → `backend.yaml`
7. Verified via `kubectl get pods -n devops-demo` — both pods `1/1 Running`

## Issues Encountered & Root Cause Analysis

### 1. Docker build failed — daemon not running
**Error:** `failed to connect to the docker API at unix:///var/run/docker.sock`
**Root cause:** Docker CLI was installed but `docker.service` was never started.
**Fix:** `systemctl start docker && systemctl enable docker`
**Lesson:** Installed ≠ running. Always verify service state before assuming a tool works.

### 2. Image visibility gap — Docker vs containerd
**Root cause:** k3s runs its own embedded containerd, which does not share Docker's local image cache. `backend.yaml` uses `imagePullPolicy: Never`, so the image had to exist directly in containerd's store.
**Fix:** `docker save` → `k3s ctr images import` to move the image from Docker's store into containerd's.
**Lesson:** Different container runtimes maintain separate image stores — a common gap between local Docker workflows and Kubernetes clusters using containerd/CRI-O.

### 3. kubectl / API server unresponsive (TLS handshake timeout) — first attempt
**Error:** `kubectl apply` failed with `context deadline exceeded` / `TLS handshake timeout`
**Root cause:** VM had ~1GB RAM total, no swap. `free -h` showed ~15Mi available; `top` showed load average of 27+ — the k3s API server became briefly unresponsive under memory pressure, compounded by Docker daemon and k3s running simultaneously.
**Fix:** Stopped Docker daemon once no longer needed, added a 2GB swapfile as a safety net.
**Lesson:** Kubernetes control planes need real memory headroom. Under-provisioned nodes fail intermittently under load rather than crashing outright — a harder class of bug to diagnose than a clean error.

### 4. Ephemeral lab environment — session expiry
**Context:** Lab VM (escbash.com) auto-stops after 1 hour, purging all data. Had to redo the entire deployment from scratch on a new VM mid-session.
**Lesson:** Always document and version-control infrastructure work immediately — don't rely on any single environment persisting. This is also the entire reason IaC and reproducible deployments matter in real DevOps work: the second deployment took minutes, not an hour, because every step was already known and scripted.

## Verification
$ kubectl get pods -n devops-demo -o wide
NAME READY STATUS RESTARTS AGE
backend-68f489dbc8-h468x 1/1 Running 0 15s
postgres-79b668cb9b-4z6ll 1/1 Running 0 15s

$ curl -s http://localhost:30080/
{"application":"DevOps Infrastructure Challenge","status":"running"}

$ curl -s http://localhost:30080/health
{"status":"healthy"}

$ curl -s http://localhost:30080/db-health
{"database":"connected"}
## Notes on External Access
This deployment ran on an ephemeral, single-node lab VM (escbash.com) without a public port-forwarding mechanism. All verification was performed via `curl` against `localhost:30080` from within the VM, confirming the full request path: client → NodePort Service → backend Pod → ClusterIP Service → Postgres Pod.
