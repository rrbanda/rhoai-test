# JobSet & LeaderWorkerSet — CPU-Only Test Cases on OpenShift

Verified test cases for validating **JobSet Operator** and **LeaderWorkerSet Operator** on **CPU-only worker nodes** on OpenShift Container Platform.

> **Last verified:** 2026-10-22 on OCP 4.20.33, JobSet 1.0.1, LeaderWorkerSet Operator, cert-manager 1.20.1

## Why This Exists

The OCP AI Workloads documentation shows GPU-flavored examples for both operators, leading customers to believe GPUs are required. **They are not.** Both operators are general-purpose Kubernetes pod-group orchestrators. GPU allocation only occurs when a container explicitly requests `nvidia.com/gpu` in its resource limits.

This repo provides tested, reproducible test cases using **Red Hat registry images only** (no `docker.io`, no `gcr.io`) on clusters with **CPU-only worker nodes**.

## Prerequisites

| Requirement | Minimum Version |
|---|---|
| OpenShift Container Platform | 4.19+ |
| cert-manager Operator for Red Hat OpenShift | 1.17+ |
| JobSet Operator | 1.0.0+ |
| LeaderWorkerSet Operator | 1.0.0+ |
| GPU Operator | **Not required** |
| Node Feature Discovery | **Not required** |

### Verify operators are installed

```bash
# cert-manager
oc get csv -n cert-manager-operator | grep cert-manager

# JobSet
oc get csv -n openshift-jobset-operator | grep jobset
oc get jobsetoperators cluster -o jsonpath='{.status.conditions[?(@.type=="Available")].status}'
# Expected: True

# LeaderWorkerSet
oc get crd leaderworkersets.leaderworkerset.x-k8s.io
oc get leaderworkersetoperators cluster -o jsonpath='{.status.conditions[?(@.type=="Available")].status}'
# Expected: True
```

## Images Used

| Image | Auth Required | Purpose |
|---|---|---|
| `registry.access.redhat.com/ubi9/ubi-minimal:9.6` | No | Basic sanity tests |
| `registry.redhat.io/rhoai/odh-th06-cpu-torch210-py312-rhel9@sha256:4b762d...` | Yes (RHOAI entitlement) | CPU-only distributed PyTorch training |
| `registry.redhat.io/rhel10/nginx-126` | Yes (Red Hat registry) | HTTP connectivity tests |
| `registry.access.redhat.com/ubi9/python-312:9.6` | No | Python-based distributed work |

> **Key image:** `odh-th06-cpu-torch210-py312-rhel9` is Red Hat's **CPU-only** Training Hub image from RHOAI 3.5. It contains PyTorch 2.10, Python 3.12, and the Gloo distributed backend — **no CUDA, no GPU dependency**.

## Quick Start

```bash
# Clone this repo
git clone https://github.com/rrbanda/rhoai-test.git
cd rhoai-test

# Create a test namespace
oc new-project cpu-test-cases
```

Then follow the test cases below in order.

---

## JobSet Test Cases

### Test 1: Basic CPU Sanity

Validates the JobSet controller creates, coordinates, and completes a leader + workers job group.

```bash
oc apply -f jobset/01-basic-sanity.yaml

# Watch pods (expect 3: 1 leader + 2 workers)
oc get pods -l jobset.sigs.k8s.io/jobset-name=jobset-cpu-sanity -w

# Verify completion
oc get jobset jobset-cpu-sanity -o jsonpath='{.status.conditions[?(@.type=="Completed")].status}'
# Expected: True

# Check logs
oc logs -l jobset.sigs.k8s.io/jobset-name=jobset-cpu-sanity

# Cleanup
oc delete jobset jobset-cpu-sanity
```

**Pass criteria:**
- All 3 pods reach `Completed` status
- JobSet condition `Completed` is `True`
- No pods scheduled on GPU-tainted nodes

### Test 2: Distributed PyTorch Training on CPU (Gloo Backend) ⭐

This is the critical test. Validates JobSet orchestrating real distributed DDP training across 2 pods using the `gloo` backend (not `nccl` which requires GPUs).

> **Requires:** RHOAI entitlement for the `registry.redhat.io` pull. Image is ~3GB — first pull takes 1-2 minutes.

```bash
oc apply -f jobset/02-pytorch-distributed-cpu.yaml

# Watch pods (expect 2 worker pods)
oc get pods -l jobset.sigs.k8s.io/jobset-name=jobset-pytorch-cpu -w

# Wait for both to complete (2-3 minutes including image pull)
oc wait --for=condition=Completed jobset/jobset-pytorch-cpu --timeout=300s

# Check training logs — Rank 0 (master)
oc logs $(oc get pods -l jobset.sigs.k8s.io/jobset-name=jobset-pytorch-cpu -o name | head -1)

# Check training logs — Rank 1 (worker)
oc logs $(oc get pods -l jobset.sigs.k8s.io/jobset-name=jobset-pytorch-cpu -o name | tail -1)

# Cleanup
oc delete jobset jobset-pytorch-cpu
```

**Expected output (verified 2026-10-22):**

```
=== Distributed PyTorch CPU Training ===
Hostname: jobset-pytorch-cpu-workers-0-0
Rank: 0, World Size: 2
Master: jobset-pytorch-cpu-workers-0-0.jobset-pytorch-cpu:29500
PyTorch: 2.10.0
CUDA available: False
[Rank 0] Initializing gloo backend...
[Gloo] Rank 0 is connected to 1 peer ranks. Expected number of connected peer ranks is : 1
[Rank 0] Process group initialized!
[Rank 0] Epoch 1/3 - avg loss: 2.3188
[Rank 0] Epoch 2/3 - avg loss: 2.3259
[Rank 0] Epoch 3/3 - avg loss: 2.3264
[Rank 0] Training COMPLETE - all ranks synchronized
[Rank 0] Done.
```

**Pass criteria:**
- Both pods initialize with `backend="gloo"` (not `nccl`)
- `CUDA available: False` on both ranks
- Both ranks print loss values for all 3 epochs
- `dist.barrier()` completes (proves inter-pod communication via Gloo)
- Both pods print "Training COMPLETE - all ranks synchronized"
- JobSet condition `Completed` is `True`

### Test 3: Parallel Batch Processing (HPC Pattern)

Validates JobSet for a non-AI HPC-style workload — proves the operator is not GPU-specific.

```bash
oc apply -f jobset/03-batch-processing.yaml

# Watch (expect 6 pods: 3 replicas × 2 completions)
oc get pods -l jobset.sigs.k8s.io/jobset-name=jobset-batch-cpu -w

# Verify
oc get jobset jobset-batch-cpu -o jsonpath='{.status.conditions[?(@.type=="Completed")].status}'
# Expected: True

# Cleanup
oc delete jobset jobset-batch-cpu
```

---

## LeaderWorkerSet Test Cases

### Test 1: Basic CPU Sanity

Validates LWS creates and maintains leader-worker groups on CPU nodes.

```bash
oc apply -f leaderworkerset/01-basic-sanity.yaml

# Watch pods (expect 6: 2 groups × 3 pods)
oc get pods -l leaderworkerset.sigs.k8s.io/name=lws-cpu-sanity -w

# Verify StatefulSets
oc get statefulsets
# Expected: lws-cpu-sanity 2/2, lws-cpu-sanity-0 2/2, lws-cpu-sanity-1 2/2

# Check leader log
oc logs lws-cpu-sanity-0

# Cleanup
oc delete lws lws-cpu-sanity
```

**Pass criteria:**
- 6 pods Running (2 leaders + 4 workers)
- 3 StatefulSets all show Ready
- Leaders start before workers (`startupPolicy: LeaderCreated`)

### Test 2: HTTP Connectivity (nginx)

Validates inter-pod network connectivity via HTTP using a Red Hat nginx image.

> **Note:** `registry.redhat.io/rhel10/nginx-126` is an S2I builder image. We override the command to start nginx directly. The default docroot returns 403 but the test validates DNS + TCP + HTTP connectivity, not content serving.

```bash
oc apply -f leaderworkerset/02-nginx-http-connectivity.yaml

# Wait for pods
oc get pods -l leaderworkerset.sigs.k8s.io/name=lws-nginx-cpu -w

# Test: Leader 0 → Worker 0-1 (same group)
oc exec lws-nginx-cpu-0 -- curl -sI http://lws-nginx-cpu-0-1.lws-nginx-cpu:8080
# Expected: HTTP/1.1 403 Forbidden  (nginx responds = connectivity works)

# Test: Worker → Leader (reverse)
oc exec lws-nginx-cpu-0-1 -- curl -sI http://lws-nginx-cpu-0.lws-nginx-cpu:8080
# Expected: HTTP/1.1 403 Forbidden

# Test: Cross-group connectivity (Group 0 → Group 1)
oc exec lws-nginx-cpu-0 -- curl -sI http://lws-nginx-cpu-1.lws-nginx-cpu:8080
# Expected: HTTP/1.1 403 Forbidden

# Cleanup
oc delete lws lws-nginx-cpu
```

**Pass criteria:**
- All 6 pods Running
- `curl` between pods returns HTTP response (403 proves nginx is serving)
- DNS resolves across pods in both directions
- Cross-group communication works

### Test 3: Distributed Work Simulation (Python)

Validates a leader-worker coordination pattern: leader distributes tasks via HTTP, workers process and return results. Simulates distributed inference without GPUs.

```bash
oc apply -f leaderworkerset/03-distributed-work-python.yaml

# Watch pods (expect 3: 1 leader + 2 workers)
oc get pods -l leaderworkerset.sigs.k8s.io/name=lws-distributed-cpu -w

# Wait ~60s for workers to process all tasks, then check leader log
oc logs lws-distributed-cpu-0
# Expected: "[Leader] Result from lws-distributed-cpu-0-1: task-0" ... through task-19

# Check worker logs
oc logs lws-distributed-cpu-0-1
oc logs lws-distributed-cpu-0-2

# Verify all tasks processed
oc exec lws-distributed-cpu-0 -- curl -s http://localhost:8080/status
# Expected: {"total": 20, "distributed": 20, "results": 20}

# Cleanup
oc delete lws lws-distributed-cpu
```

**Pass criteria:**
- Workers discover leader via pod DNS
- Workers pull tasks from leader over HTTP
- Workers submit results back to leader
- All 20 tasks distributed and completed
- `/status` shows `"results": 20`

---

## Full Cleanup

```bash
oc delete project cpu-test-cases
```

## Summary Matrix

| # | Operator | Test | Image | Auth | Validates |
|---|---|---|---|---|---|
| J1 | JobSet | Basic Sanity | `ubi9/ubi-minimal:9.6` | No | Job creation, coordination, completion |
| J2 | JobSet | **PyTorch CPU (Gloo)** | `odh-th06-cpu-torch210-py312-rhel9` | RHOAI | Distributed DDP training, inter-pod Gloo comms |
| J3 | JobSet | Batch HPC | `ubi9/ubi-minimal:9.6` | No | Parallel batch processing (non-AI) |
| L1 | LWS | Basic Sanity | `ubi9/ubi-minimal:9.6` | No | Leader/worker groups, StatefulSets, lifecycle |
| L2 | LWS | nginx HTTP | `rhel10/nginx-126` | RHEL | Inter-pod HTTP, DNS, TCP connectivity |
| L3 | LWS | Distributed Work | `ubi9/python-312:9.6` | No | Leader-worker task distribution over HTTP |

## FAQ

**Q: Why does the PyTorch test use `gloo` instead of `nccl`?**
A: `nccl` (NVIDIA Collective Communications Library) requires GPUs. `gloo` is PyTorch's CPU-compatible distributed backend. Both are first-class backends in PyTorch — `gloo` is the default for CPU, `nccl` for GPU.

**Q: Why does the nginx test return 403?**
A: The `rhel10/nginx-126` image is an S2I (Source-to-Image) builder. Its default document root is empty, so nginx returns 403 Forbidden. The point of the test is validating HTTP connectivity between pods, not content serving — and 403 proves nginx is running and reachable.

**Q: Can I use the CUDA training image on CPU-only nodes?**
A: Technically yes — PyTorch falls back to CPU when no GPU is present. However, `odh-training-cuda128-torch28-py312-rhel9` is ~8GB vs ~3GB for the CPU-only image. Use `odh-th06-cpu-torch210-py312-rhel9` instead.

**Q: What about the `docker.io/nginxinc/nginx-unprivileged:1.27` image in the OCP docs?**
A: It has **CVE-2023-44487** (HTTP/2 Rapid Reset). Use `registry.redhat.io/rhel10/nginx-126` from Red Hat instead, which is patched and maintained.

## References

- [OCP 4.20 — JobSet Operator documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/ai_workloads/index#jobset-operator)
- [OCP 4.20 — LeaderWorkerSet Operator documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/ai_workloads/index#leader-worker-set-operator)
- [Upstream JobSet project (kubernetes-sigs/jobset)](https://github.com/kubernetes-sigs/jobset)
- [Upstream LWS project (kubernetes-sigs/lws)](https://github.com/kubernetes-sigs/lws)
- [Red Hat Container Catalog](https://catalog.redhat.com/en/search?searchType=Containers)
