# Spark Operator Webhook — Namespace Selector Fix

## The Problem

The Spark Operator's `mutatingwebhookconfiguration` and `validatingwebhookconfiguration`
default to only watching the `default` namespace:

```yaml
namespaceSelector:
  matchExpressions:
  - key: kubernetes.io/metadata.name
    operator: In
    values:
    - default
```

Without the webhook covering your target namespace, **volumes and ConfigMaps are not
mounted** into Spark driver/executor pods. SparkApplications appear to submit, but
pods start with missing mounts and the job fails.

## Why Manual Patches Don't Stick

The Spark Operator controller runs a **reconciliation loop** that regenerates the
webhook configuration from its internal defaults. If you:

1. Scale down the operator
2. Patch the webhook
3. Scale back up

…the operator comes back, sees the webhook doesn't match its desired state, and
**overwrites your patch back to `default` only**.

This is by design — Kubernetes operators own the resources they create.

---

## Fix A — Annotation + Patch (Recommended ✅)

Tell the RHOAI/ODH controller to stop reconciling the webhook, then patch.
The `opendatahub.io/managed=false` annotation is the supported escape hatch.

```bash
# Step 1: Stop the operator from reconciling the webhooks
oc annotate mutatingwebhookconfiguration mutating-webhook-configuration \
  opendatahub.io/managed=false --overwrite

oc annotate validatingwebhookconfiguration validating-webhook-configuration \
  opendatahub.io/managed=false --overwrite

# Step 2: Patch the namespace selector — now it sticks
TARGET_NS="redhat-ods-applications"   # ← change to your namespace

oc patch mutatingwebhookconfiguration mutating-webhook-configuration --type='json' \
  -p='[
    {"op":"replace","path":"/webhooks/0/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]},
    {"op":"replace","path":"/webhooks/1/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]},
    {"op":"replace","path":"/webhooks/2/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]},
    {"op":"replace","path":"/webhooks/3/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]}
  ]'

oc patch validatingwebhookconfiguration validating-webhook-configuration --type='json' \
  -p='[
    {"op":"replace","path":"/webhooks/0/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]},
    {"op":"replace","path":"/webhooks/1/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]},
    {"op":"replace","path":"/webhooks/2/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]}
  ]'

# Step 3: Verify
oc get mutatingwebhookconfiguration mutating-webhook-configuration \
  -o jsonpath='{.webhooks[0].namespaceSelector}' | python3 -m json.tool
```

> **Trade-off:** The operator no longer auto-updates the webhook. After an
> operator upgrade, re-check the webhook and re-apply the annotation if needed.

---

## Fix B — Label-Based Selector + Annotation (Best for Multi-Namespace)

Instead of listing every namespace by name, switch to a label-based selector.
Any namespace with the label is automatically included — no webhook patches needed
when adding new namespaces.

```bash
# Step 1: Stop reconciliation
oc annotate mutatingwebhookconfiguration mutating-webhook-configuration \
  opendatahub.io/managed=false --overwrite

oc annotate validatingwebhookconfiguration validating-webhook-configuration \
  opendatahub.io/managed=false --overwrite

# Step 2: Replace the name-based selector with a label selector
oc patch mutatingwebhookconfiguration mutating-webhook-configuration --type='json' \
  -p='[
    {"op":"replace","path":"/webhooks/0/namespaceSelector","value":{"matchLabels":{"spark-webhook":"enabled"}}},
    {"op":"replace","path":"/webhooks/1/namespaceSelector","value":{"matchLabels":{"spark-webhook":"enabled"}}},
    {"op":"replace","path":"/webhooks/2/namespaceSelector","value":{"matchLabels":{"spark-webhook":"enabled"}}},
    {"op":"replace","path":"/webhooks/3/namespaceSelector","value":{"matchLabels":{"spark-webhook":"enabled"}}}
  ]'

oc patch validatingwebhookconfiguration validating-webhook-configuration --type='json' \
  -p='[
    {"op":"replace","path":"/webhooks/0/namespaceSelector","value":{"matchLabels":{"spark-webhook":"enabled"}}},
    {"op":"replace","path":"/webhooks/1/namespaceSelector","value":{"matchLabels":{"spark-webhook":"enabled"}}},
    {"op":"replace","path":"/webhooks/2/namespaceSelector","value":{"matchLabels":{"spark-webhook":"enabled"}}}
  ]'

# Step 3: Label each namespace that needs the Spark webhook
oc label namespace default spark-webhook=enabled
oc label namespace redhat-ods-applications spark-webhook=enabled
# oc label namespace <any-other-ns> spark-webhook=enabled
```

> **Advantage:** Adding a new namespace is just
> `oc label namespace <ns> spark-webhook=enabled` — no webhook patches needed.

---

## Fix C — Plain Patch (Temporary — Testing Only)

A bare patch without the annotation. **Will be overwritten** by the operator on
its next reconciliation cycle. Use only for quick tests.

```bash
TARGET_NS="redhat-ods-applications"

oc patch mutatingwebhookconfiguration mutating-webhook-configuration --type='json' \
  -p='[
    {"op":"replace","path":"/webhooks/0/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]},
    {"op":"replace","path":"/webhooks/1/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]},
    {"op":"replace","path":"/webhooks/2/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]},
    {"op":"replace","path":"/webhooks/3/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]}
  ]'

oc patch validatingwebhookconfiguration validating-webhook-configuration --type='json' \
  -p='[
    {"op":"replace","path":"/webhooks/0/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]},
    {"op":"replace","path":"/webhooks/1/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]},
    {"op":"replace","path":"/webhooks/2/namespaceSelector/matchExpressions/0/values",
     "value":["default","'"${TARGET_NS}"'"]}
  ]'
```

---

## Comparison

| Fix | Persists? | Multi-Namespace | Operator Upgrade Safe | Best For |
|-----|-----------|-----------------|----------------------|----------|
| **A — Annotation + Patch** ✅ | Yes | Manual (list names) | Re-apply annotation | Production, single namespace |
| **B — Label Selector** | Yes | Automatic (label) | Re-apply annotation | Production, multi-namespace |
| **C — Plain Patch** | **No** | Manual | N/A (overwritten) | Quick testing only |

## Verification

After applying any fix, confirm the webhook covers your namespace:

```bash
# Check mutating webhook
oc get mutatingwebhookconfiguration mutating-webhook-configuration \
  -o jsonpath='{range .webhooks[*]}{.name}: {.namespaceSelector}{"\n"}{end}'

# Check validating webhook
oc get validatingwebhookconfiguration validating-webhook-configuration \
  -o jsonpath='{range .webhooks[*]}{.name}: {.namespaceSelector}{"\n"}{end}'

# Test: submit a SparkApplication and verify volumes mount correctly
oc get pods -l sparkoperator.k8s.io/launched-by-spark-operator=true -o yaml \
  | grep -A2 volumeMounts
```

## Also Required: NetworkPolicy

If your namespace has default-deny ingress rules, Spark executor pods cannot reach
the driver without a NetworkPolicy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: spark-operator-allow-internal
spec:
  podSelector:
    matchLabels:
      sparkoperator.k8s.io/launched-by-spark-operator: "true"
  policyTypes: [Ingress]
  ingress:
  - ports:
    - { port: 7078, protocol: TCP }
    - { port: 7079, protocol: TCP }
    - { port: 4040, protocol: TCP }
    from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress
```

## References

- [OCP 4.20 — Spark Operator documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/ai_workloads/index#spark-operator)
- [RHOAI 3.5 — Spark Operator user guide](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/creating_distributed_data_processing_applications_with_the_kubeflow_spark_operator/)
- [Upstream Spark Operator webhook config](https://github.com/kubeflow/spark-operator/tree/master/config/webhook)
