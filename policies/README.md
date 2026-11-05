# ACM 5.0 custom observability policies

First draft. ACM 5.0 installs default observability (the `MultiClusterObservability`, the default `PrometheusAgent`, the default platform `ScrapeConfig` and `PrometheusRule`, and the `ClusterManagementAddOn` placement entries). This repo does not policy that default.

Policies here are only for custom configuration:

- OpenShift `cluster-monitoring-config` and `user-workload-monitoring-config` on managed clusters, adapted from [bry-acm-policy-samples `policies/cluster-configs/monitoring`](https://github.com/brian-jarvis/bry-acm-policy-samples/tree/implement-perses-dashboards/policies/cluster-configs/monitoring) (MCOA variants).
- One example hub `ScrapeConfig`. MCOA picks it up from annotations. There is no policy that rewrites `ClusterManagementAddOn` placement configs.
- A patch of the existing hub `MultiClusterObservability` `observability`: metric capabilities, right-sizing analytics, and infra-or-worker placement.

Metrics collection through MCOA is GA. This draft does not enable logs, traces, or OpenTelemetry.

## What is not generated

- No `MultiClusterObservability` install. `policy-patch-multiclusterobservability` patches the existing object and does not set `storageConfig`. If `observability` is missing, that template emits nothing.
- No `observability-addon-config` policy. That sample kept the add-on placement list in sync by hand. In current MCOA a custom `ScrapeConfig` is selected with labels and `observability.open-cluster-management.io/placements`.
- No `open-cluster-management-agent-addon` monitoring-label policy. MCOA sets `openshift.io/cluster-monitoring: "true"` on the metrics namespace (ACM-35500).
- No `dependencies` or `extraDependencies`. Apply order is the setup sequence below. The GitOps shape follows [policydependencydemo](https://github.com/ch-stark/policydependencydemo): one OpenShift GitOps instance, Policy Generator on that instance, generated Policies in namespace `policies`.

## Layout

```
setup/                  # apply on the hub once; not synced by the Application
  gitops/               # Subscription for the OpenShift GitOps operator, local-cluster only
  plugin/               # Policy Generator plugin and Policy RBAC on ArgoCD/openshift-gitops
  applications/         # AppProject + Application in openshift-gitops
policies/               # PolicyGenerator; Argo CD renders Policies into namespace policies
  monitoring/           # spoke ConfigMaps
  observability/        # hub ScrapeConfig and MultiClusterObservability patch
```

`policy-custom-cluster-monitoring` is bound to every cluster in the `global` set. `policy-custom-metrics-scrapeconfig` and `policy-patch-multiclusterobservability` are bound to `local-cluster` only. The `ScrapeConfig` and `MultiClusterObservability` both live on the hub.

## MultiClusterObservability patch

`policies/observability/multiclusterobservability.yaml` uses `musthave` on the existing `observability` object:

- `capabilities.platform.metrics` and `capabilities.userWorkloads.metrics` (collection, alerts, and the metrics UI)
- namespace and virtualization right-sizing analytics
- `nodeSelector` on infra nodes when the hub has that role, otherwise worker, plus the infra toleration

Logs, traces, and OpenTelemetry stay unset. `storageConfig` stays whatever ACM created. The toleration list in the template is the desired list for that field.

## Custom ScrapeConfig

Create the object in `open-cluster-management-observability` with:

```yaml
metadata:
  labels:
    app.kubernetes.io/part-of: multicluster-observability-addon
    app.kubernetes.io/component: platform-metrics-collector
  annotations:
    observability.open-cluster-management.io/placements: open-cluster-management-observability/global
```

`app.kubernetes.io/component` is `platform-metrics-collector` or `user-workload-metrics-collector`. The annotation value is a comma-separated list of `<placement namespace>/<placement name>`. The default fleet placement is `open-cluster-management-observability/global`.

MCOA adds the object to that placement. For platform collectors it also server-side-applies `scheme: HTTPS`, `scrapeClass: not-configurable`, static target `not-configurable`, and the backup label. Leave those fields unset in Git and put the custom `match[]` list in `spec.params`.

Edit `policies/observability/custom-scrapeconfig.yaml` before relying on it. The match list is a placeholder.

## Monitoring config

`policies/monitoring/*.yaml` merge a custom baseline over the live ConfigMap (retention, PVC size, requests, external labels, infra-or-worker placement). `storageClassName` is `gp3-csi` from the sample. Change it for the platform you run.

Enable default ACM observability before these policies. They do not create it.

## Apply

Prerequisite: ACM hub, `ManagedClusterSet` `global`, and namespace `open-cluster-management-global-set` already bound to that set.

```bash
kubectl apply -k setup/
kubectl get policy -n open-cluster-management-global-set openshift-gitops-operator
# wait until Compliant, and until ArgoCD/openshift-gitops exists
kubectl apply -k setup/plugin/
kubectl apply -k setup/applications/
```

The Application `acm-observability-policies` in `openshift-gitops` syncs this repo path `policies` and writes into namespace `policies`.

Preview without the cluster, with the [Policy Generator plugin](https://github.com/open-cluster-management-io/policy-generator-plugin) installed:

```bash
kustomize build --enable-alpha-plugins --enable-exec policies/
```
