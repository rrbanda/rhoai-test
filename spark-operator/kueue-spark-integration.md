# Kueue + SparkApplication Integration on RHOAI 3.5

## Prerequisites

| Requirement | Minimum Version | Why |
|---|---|---|
| Red Hat build of Kueue (RHBoK) | **1.4+** | SparkApplication reconciler code does not exist in 1.3 |
| Kubeflow Spark Operator | 2.4.0 (GA in RHOAI 3.5) | Earlier versions lack Kueue-compatible suspend/resume |
| cert-manager Operator | 1.17+ | Required for Kueue webhook certificates |

> ⚠️ **RHBoK 1.3 will NOT work.** The `sparkoperator.k8s.io/sparkapplication` framework
> value is not recognized by Kueue 1.3. You will see this error:
>
> ```
> Unsupported value: "sparkoperator.k8s.io/sparkapplication": supported values:
> "BatchJob", "RayJob", "RayCluster", "JobSet", "MPIJob", "PaddleJob",
> "PyTorchJob", "TFJob", "TrainJob", "XGBoostJob", "JaxJob", "AppWrapper",
> "Pod", "Deployment", "StatefulSet", "LeaderWorkerSet"
> ```

## Check Your Kueue Version

```bash
oc get csv -n openshift-kueue-operator | grep kueue
# Must show 1.4.x — if it shows 1.3.x, upgrade first
```

## Configure Kueue for SparkApplication

Use `integrations.frameworks` — **not** `externalFrameworks`:

```bash
oc edit kueue cluster -n openshift-kueue-operator
```

```yaml
apiVersion: kueue.openshift.io/v1
kind: Kueue
metadata:
  name: cluster
  namespace: openshift-kueue-operator
spec:
  config:
    integrations:
      frameworks:
      - BatchJob
      - RayJob
      - RayCluster
      - JobSet
      - Pod
      - Deployment
      - StatefulSet
      - LeaderWorkerSet
      - sparkoperator.k8s.io/sparkapplication    # ← add this
    gangScheduling:
      policy: ByWorkload    # ← optional but recommended for Spark
```

### Why `frameworks` and not `externalFrameworks`?

SparkApplication has a **built-in** Kueue reconciler starting in RHBoK 1.4.
It's not an external/custom integration — it just needs to be listed in `frameworks`.

The `externalFrameworks` API is for truly custom CRDs that don't have a built-in
Kueue controller. Using it for SparkApplication won't work because the built-in
reconciler won't be activated.

### What about `featureGates`?

The Red Hat Kueue CR (`kueue.openshift.io/v1`) **does not expose** a `featureGates`
field. The upstream Kueue requires a `SparkApplicationIntegration` feature gate,
but the Red Hat operator enables it **automatically** when it detects
`sparkoperator.k8s.io/sparkapplication` in the `frameworks` list.

You don't need to toggle anything else.

## Verify It Worked

```bash
# Check the Kueue controller logs for SparkApplication reconciler registration
oc logs deployment/kueue-controller-manager -n openshift-kueue-operator --tail=50 \
  | grep -i spark

# Verify the webhook is watching SparkApplications
oc get mutatingwebhookconfiguration -o yaml | grep -A5 sparkapplication
```

## Create Queue Infrastructure

### ClusterQueue

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: spark-cluster-queue
spec:
  namespaceSelector: {}
  resourceGroups:
  - coveredResources: ["cpu", "memory"]
    flavors:
    - name: default-flavor
      resources:
      - name: cpu
        nominalQuota: "8"
      - name: memory
        nominalQuota: "32Gi"
```

### ResourceFlavor

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: default-flavor
```

### LocalQueue (in your target namespace)

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: spark-queue
  namespace: <your-namespace>
spec:
  clusterQueue: spark-cluster-queue
```

### Label the namespace

```bash
oc label namespace <your-namespace> kueue.openshift.io/managed=true --overwrite
```

## Submit a SparkApplication with Kueue

Add the `kueue.x-k8s.io/queue-name` label to your SparkApplication:

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: my-spark-job
  namespace: <your-namespace>
  labels:
    kueue.x-k8s.io/queue-name: spark-queue    # ← this triggers Kueue management
spec:
  type: Python
  mode: cluster
  # ... rest of your SparkApplication spec
```

When Kueue manages the job:
1. Kueue's webhook intercepts creation and sets `spec.suspend: true`
2. Kueue creates a `Workload` object for the total resource demand
3. If quota is available → unsuspends → Spark Operator creates pods
4. If quota is insufficient → stays Suspended → zero pods, zero waste

## Constraints

| Constraint | Detail |
|---|---|
| **Technology Preview** | Kueue + SparkApplication is TP in RHOAI 3.5 — no production SLA |
| **No dynamic allocation** | Kueue rejects SparkApplications with `dynamicAllocation.enabled: true` — executor count must be fixed |
| **Cluster mode only** | Kueue rejects SparkApplications in `client` mode |
| **RHBoK 1.4 breaks Trainer v2** | Known issue RHOAIENG-87625 — if you also use Trainer v2, it cannot share the same RHBoK instance |
| **Gang scheduling disabled by default** | Set `gangScheduling.policy: ByWorkload` in Kueue CR to enable all-or-nothing pod scheduling |

## Also Required

Before any of this works, the Spark Operator webhook must cover your namespace.
See [webhook-namespace-fix.md](webhook-namespace-fix.md) for the persistent fix.

## References

- [RHOAI 3.5 — Technology Preview: Kueue + SparkApplication](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/release_notes/technology-preview-features_relnotes)
- [Upstream Kueue — Run a SparkApplication](https://kueue.sigs.k8s.io/docs/tasks/run/kubeflow/sparkapplications/)
- [Upstream Kueue — Configuration API](https://kueue.sigs.k8s.io/docs/reference/kueue-config.v1beta2/)
- [OCP 4.22 — Red Hat build of Kueue](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/ai_workloads/red-hat-build-of-kueue)
