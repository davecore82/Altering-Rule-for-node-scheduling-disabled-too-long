# Cordoned Nodes Can Silently Stall `MachineConfigPool` Rollouts on OpenShift/ARO

A deep-dive into a reproducible OpenShift/ARO failure mode: a single cordoned
(`SchedulingDisabled`) node can block an entire `MachineConfigPool` rollout
cluster-wide — including trivial, no-reboot changes like a pull-secret update —
with **no `Degraded` condition and no clear error message**. This repo
documents the root cause, a full lab reproduction, why the default monitoring
stack doesn't catch it, and two tested alerting options to close the gap.

Everything here was built and verified against a live OpenShift 4.21 ARO
cluster. Nothing is theoretical — every claim below was confirmed by directly
observing cluster state, controller logs, and live Prometheus/Alertmanager
behavior.

## TL;DR

- A cordoned node — for *any* reason, even one unrelated to the Machine
  Config Operator (MCO) — counts against a `MachineConfigPool`'s
  `maxUnavailable` budget (default `1`). One cordoned node is enough to
  freeze the entire pool.
- "Uncordon it and the problem clears, then it comes back a minute later" is
  **not** one controller flip-flopping. It's two independent controllers: MCO
  reacts correctly to the uncordon and finishes the rollout; a separate,
  still-pending `Machine` deletion (e.g. from a blocked cluster-autoscaler
  scale-down) re-cordons the node on its own retry schedule.
- The default OpenShift monitoring stack has **no alert that catches this in
  a useful timeframe**. The closest built-in alerts have 60-minute to
  6-hour `for:` thresholds.
- There **is** a fully supported way to add your own alert for this —
  `AlertingRule` (`monitoring.openshift.io/v1`) — which is different from,
  and more capable than, a raw `PrometheusRule`. Two working versions are
  included below, tested live.

## The failure mode

1. The cluster-wide pull secret (or any lightweight, no-reboot `MachineConfig`
   change) is updated. MCO renders a new `MachineConfig` for the affected
   pool as expected.
2. One node in the pool is cordoned (`spec.unschedulable: true`) for a reason
   that has nothing to do with this rollout — e.g. the cluster-autoscaler
   tried to scale it down but couldn't fully drain it because of a
   restrictive `PodDisruptionBudget` on a workload there.
3. The pool's `maxUnavailable` is unset (defaults to `1`). The cordoned node
   alone consumes that entire budget.
4. Result: `oc get mcp` shows `UPDATING=True`, `0` ready, `0` updated,
   `1` unavailable — indefinitely. **No node is ever selected for the new
   config, not even nodes that need nothing more than a file write.**
   There's no `Degraded` condition because nothing is actually failing — the
   controller is correctly waiting for capacity it will never get.

## Root cause, confirmed against controller source and live logs

`machine-config-controller`'s node controller treats *any* `Unschedulable`
node as "unavailable" for budget purposes, regardless of whether MCO itself
put it there:

```
node_controller.go: Pool worker[zone=...]: node X: Reporting unready: node X is reporting Unschedulable
```

With `maxUnavailable: 1`, once one node is counted as unavailable, the
candidate-selection logic computes zero remaining capacity and **never even
logs a candidate-selection attempt** for the pool — compare this to a
healthy pool, which logs `Pool X: N candidate nodes ... capacity: 1` on every
reconcile.

### Why "uncordon, wait, it comes back" isn't one controller being flaky

Live timeline from the lab reproduction (a pod with a `minAvailable: 1` PDB
pinned to a node whose backing `Machine` was marked for deletion):

| Time (relative to uncordon) | Event |
|---|---|
| T+0s | `oc adm uncordon` issued |
| T+~5s | MCO's node controller: *"Reporting ready"* — capacity freed instantly |
| T+~65s | MCP fully updated, `Updating=False` — **"cleared right away"** |
| T+~99s | Independently, `machine-api`'s `machine-drain-controller` fires its next exponential-backoff retry (observed backoff: 10s → 20s → 40s → 80s → minutes), re-cordons the node as a precondition of trying the drain again, and fails on the same PDB |
| T+~99s | MCO's node controller immediately reports *"Reporting unready... is reporting Unschedulable"* again — **"back to `SchedulingDisabled`"** |

Two separate controllers, two separate reasons, both legitimately reacting to
real (if stale) state. The underlying `Machine` deletion stays stuck in this
retry loop — with growing backoff, observed up to ~9 minutes between
attempts — until whatever is blocking eviction (the PDB, in this repro) is
actually removed.

### Durable fix vs. band-aid

- **Fixing the real blocker** (removing/loosening the PDB, or whatever is
  stopping eviction) lets the pending `Machine` deletion finally complete.
  Confirmed: VM deleted → disk deleted → NIC deleted → node removed →
  `machine deletion successful`, no further flapping.
- **Manually setting `machineconfiguration.openshift.io/desiredConfig`** on
  individual nodes (a workaround some have used to unblock a stuck pool) is
  not documented anywhere as a supported recovery procedure. It works
  because the per-node `machine-config-daemon` acts on that annotation
  directly, bypassing the pool-level `maxUnavailable` gate — but it's editing
  a controller-owned field out from under the controller. Treat it as a
  break-glass step, not a standard one. The lower-risk equivalent is
  resolving the actual blocker, or temporarily raising `maxUnavailable` on
  the pool.

## Why the default monitoring stack doesn't catch this

Checked empirically — the cluster was left in the stuck state for 44+
minutes while watching Prometheus's `ALERTS` metric directly, not just
reading rule definitions:

| Alert | Would it catch this? |
|---|---|
| `MachineWithNoRunningPhase`, `MachineWithoutValidNode`, `PodDisruptionBudgetAtLimit` | Went `pending` (confirming they're watching the right area) but have **60-minute** `for:` thresholds — deliberately tuned to tolerate normal node replacement, too slow to be "early" |
| `MachineNotYetDeleted` | **6-hour** `for:` threshold |
| `KubePodNotScheduled` (30m), `KubeDeploymentRolloutStuck` (15m) | Never fired — the affected pods kept getting evicted and recreated, resetting each alert's pending timer before it could accumulate enough continuous duration |
| `KubeNodeNotReady`, `KubeNodePressure`, `KubeNodeReadinessFlapping` | Explicitly **exclude** cordoned nodes by design (`kube_node_spec_unschedulable == 0` baked into the expression), so a cordoned node is invisible to them no matter how long it's been there |
| `MCCPoolAlert` (the one MCO-emitted alert that sounds related) | Fires on a completely different condition — conflicting pool label selectors on a node, not a stalled rollout |

**There is no `MachineConfigPool`-status metric exposed anywhere** (no
`kube_machineconfigpool_status_*` style metric), so you also can't directly
alert on "pool stuck" — only on proxies for it.

### Two dead ends worth knowing about before you try them

- **Labeling a namespace `openshift.io/cluster-monitoring: "true"`** to get
  your own `PrometheusRule` picked up by the platform Prometheus *works*,
  but is explicitly documented as **unsupported** — that label is reserved
  for Red Hat-shipped namespaces, and manually deploying monitoring
  resources into a namespace with that label is called out by name as
  unsupported.
- **User Workload Monitoring** (the supported way to add your own
  `PrometheusRule`) automatically injects a `namespace="<your-namespace>"`
  matcher into every query your rule evaluates — a hard multi-tenancy
  boundary. Since `kube_node_spec_unschedulable` is a cluster-wide platform
  metric with no meaningful namespace of its own, the injected filter makes
  the query match nothing. Confirmed live: the rule loaded fine, but sat at
  `inactive` forever.

## The supported fix: `AlertingRule`

`AlertingRule` (`apiVersion: monitoring.openshift.io/v1`) is a
purpose-built, documented, supported CRD — distinct from the Prometheus
Operator's `PrometheusRule` — specifically for cluster admins to add custom
alerts based on core platform metrics. It must live in `openshift-monitoring`;
the Cluster Monitoring Operator validates it and generates the actual
`PrometheusRule` for you, which is exactly why it's supported where
hand-placing one yourself isn't.

### Option A — simple, cheap, a bit noisy

```yaml
apiVersion: monitoring.openshift.io/v1
kind: AlertingRule
metadata:
  name: node-scheduling-disabled-too-long
  namespace: openshift-monitoring
spec:
  groups:
  - name: custom-mco-rollout.rules
    rules:
    - alert: NodeSchedulingDisabledTooLong
      for: 20m
      expr: kube_node_spec_unschedulable == 1
      labels:
        severity: warning
      annotations:
        summary: "Node {{ $labels.node }} has been SchedulingDisabled for more than 20 minutes"
        description: "A cordoned node can silently consume a MachineConfigPool's maxUnavailable budget and block rollouts cluster-wide without the pool reporting Degraded. Check 'oc get mcp' and the node's machineconfiguration.openshift.io/state annotation before escalating — this alert also fires on legitimate maintenance or a slow-but-healthy node reboot."
```

**Honest limitation:** a cordoned node is a normal, routine thing — manual
maintenance, or even a completely healthy MCO rollout, cordons a node for the
duration of drain + reboot + rejoin, which can take well over 15–20 minutes
on a slower node. This alert will also fire on clusters doing exactly what
they're supposed to. Tune the threshold to your own environment's typical
node-update duration, and treat a firing alert as "go check `oc get mcp`,"
not as a confirmed diagnosis.

### Option B — precise: only fires when MCO *isn't* the one working on the node

The real discriminator isn't "node is cordoned" — it's "node is cordoned
**and** the machine-config-daemon's own state annotation says it isn't
actively working on it" (`machineconfiguration.openshift.io/state` is
`Working` only while MCO is actively draining/updating a node, and `Done`
otherwise). That detail isn't exposed by the platform's kube-state-metrics
(node annotations are explicitly denylisted there), so this option adds a
small, self-hosted, stock `kube-state-metrics` instance via User Workload
Monitoring — no custom code, just a config flag:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-state-exporter
  namespace: your-namespace
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-state-exporter
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-state-exporter
subjects:
- kind: ServiceAccount
  name: node-state-exporter
  namespace: your-namespace
roleRef:
  kind: ClusterRole
  name: node-state-exporter
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-state-exporter
  namespace: your-namespace
  labels:
    app: node-state-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node-state-exporter
  template:
    metadata:
      labels:
        app: node-state-exporter
    spec:
      serviceAccountName: node-state-exporter
      containers:
      - name: kube-state-metrics
        image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.13.0
        args:
        - --resources=nodes
        - --metric-annotations-allowlist=nodes=[machineconfiguration.openshift.io/state]
        - --port=8080
        - --telemetry-port=8081
        ports:
        - containerPort: 8080
          name: http-metrics
        - containerPort: 8081
          name: telemetry
---
apiVersion: v1
kind: Service
metadata:
  name: node-state-exporter
  namespace: your-namespace
  labels:
    app: node-state-exporter
spec:
  selector:
    app: node-state-exporter
  ports:
  - name: http-metrics
    port: 8080
    targetPort: 8080
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: node-state-exporter
  namespace: your-namespace
spec:
  selector:
    matchLabels:
      app: node-state-exporter
  endpoints:
  - port: http-metrics
    interval: 30s
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-cordoned-not-mco-managed
  namespace: your-namespace
spec:
  groups:
  - name: node-cordoned-not-mco.rules
    rules:
    - alert: NodeCordonedNotManagedByMCO
      for: 15m
      expr: kube_node_spec_unschedulable == 1 and on(node) kube_node_annotations{annotation_machineconfiguration_openshift_io_state!="Working"}
      labels:
        severity: warning
      annotations:
        summary: "Node {{ $labels.node }} is cordoned but MCO is not actively working on it"
        description: "This node is SchedulingDisabled while the Machine Config Daemon reports its state as something other than Working, meaning MCO is not the cause. Something external (manual cordon, stuck Machine deletion, autoscaler) is holding this node unschedulable and may be silently blocking any MachineConfigPool rollout that targets it."
```

Prerequisites: User Workload Monitoring enabled
(`enableUserWorkload: true` in the `cluster-monitoring-config` ConfigMap in
`openshift-monitoring`) — this one *does* work under UWM, because the
exporter and the alert live in the same namespace, so UWM's automatic
namespace-matcher injection matches the metric's real namespace instead of
excluding it.

**Tradeoff:** more moving parts than Option A (a second kube-state-metrics
deployment, RBAC, a ServiceMonitor) for a materially more precise signal.
Worth it if Option A turns out to be noisy in your environment; probably
overkill if it isn't.

## Validation: both options tested live, including the negative case

It's easy to build an alert that *fires* on the bad scenario. The harder,
more important test is proving it *doesn't* fire on a healthy one. Both were
confirmed directly against a live cluster, not inferred:

**Positive case** — a node cordoned by a stuck `Machine` deletion
(`state=Done`, genuinely stuck): both alerts correctly transitioned
`pending → firing` and showed up in Alertmanager.

**Negative case** — a node legitimately cordoned by MCO itself mid-reboot
(`state=Working`), captured live with a simultaneous snapshot:

| Signal, same instant | Value |
|---|---|
| `kube_node_spec_unschedulable{node="..."}` | `1` (genuinely cordoned) |
| `kube_node_annotations{...state="Working"}` | `1` (genuinely MCO-managed) |
| Option A (simple) | would go pending/fire — it only checks cordon state |
| Option B (precise) | **not present at all** — correctly excluded by the `!="Working"` filter |

That's the core value proposition of Option B in one table: at the exact
moment a healthy, in-progress MCO reboot has a node cordoned, the precise
alert stays completely silent while the simple one would start counting down
toward firing.

## What this doesn't cover

- No existing Red Hat Insights Advisor rule appears to cover this specific
  scenario either (the KB articles found cover different `MachineConfigPool`
  degraded causes — on-disk state drift, paused pools, deleted rendered
  configs — not "cordoned node silently consumes `maxUnavailable`").
- If you have an external observability platform (Dynatrace, Datadog, etc.)
  already watching this cluster, alerting directly on the underlying
  Kubernetes events (`DrainRequeued`, `FailedDelete` in
  `openshift-machine-api`) as they occur, or using built-in anomaly-baselining
  instead of a fixed threshold, would likely be more precise than either
  option here — not verified hands-on, worth checking independently.
- None of this is official Red Hat guidance or a supported troubleshooting
  procedure — it's an independent lab reproduction and the alerting options
  are offered as-is.

## References

- [Machine configuration overview — Red Hat OpenShift docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/machine_configuration/machine-config-index)
- [Using node disruption policies to minimize disruption from machine config changes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/machine_configuration/machine-config-node-disruption_machine-configs-configure)
- [Managing alerting rules for core platform monitoring (`AlertingRule` / `AlertRelabelConfig`)](https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.21/html/managing_alerts/managing-alerts-as-an-administrator)
- [Support for monitoring — unsupported configurations](https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.21/html/support_for_monitoring/support-for-monitoring)
- [A Guide to Troubleshooting OpenShift Updates — Red Hat blog](https://www.redhat.com/en/blog/a-guide-to-troubleshooting-openshift-updates-1)
- [`machine-config-operator` source — `node_controller.go`, `status.go`](https://github.com/openshift/machine-config-operator)
- [OCPBUGS-33397 — `maxUnavailable` + cordoned node interaction fix](https://github.com/openshift/machine-config-operator/pull/4475)
- [OCPBUGS-60537 — MCO re-cordons a node if externally uncordoned mid-drain](https://github.com/openshift/machine-config-operator/pull/5276)
