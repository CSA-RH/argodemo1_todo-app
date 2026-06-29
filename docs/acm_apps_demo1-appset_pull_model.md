# ACM ApplicationSet Pull Model

> **Common setup (Steps 1–3)** is documented in the
> [ACM Bootstrap: GitOps Application Prerequisites](../../../acm_bootstrap/applications/README.md).
> See also the [overview](../README.md) for common concepts.
> Complete those steps before proceeding here.

Deploy a sample application (`todo-app`) to managed clusters using the
**pull model**. Unlike the push model (where hub ArgoCD deploys directly),
the pull model creates Application resources on the hub that are propagated
via `ManifestWork` to each managed cluster. The local ArgoCD instance on each
managed cluster then reconciles the application against its own
`https://kubernetes.default.svc`.

Repository: [argodemo1_todo-app](https://github.com/CSA-RH/argodemo1_todo-app)
(same repo: only the ApplicationSet definition changes between push and pull)

---

## Additional prerequisites (pull model only)

In addition to the [common prerequisites](README.md#prerequisites):

- **OpenShift GitOps operator installed on every target managed cluster** in the
  `openshift-gitops` namespace. The version on managed clusters must be equal
  to or older than the hub version.

Verify OpenShift GitOps is installed on managed clusters:

```bash
CLUSTER_NAME=cluster1
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get csv -n openshift-gitops-operator
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Hub Cluster (openshift-gitops namespace)                               │
│                                                                         │
│  ManagedClusterSetBinding ──► Placement ──► PlacementDecision           │
│        (bind sets)           (select clusters)   (output)               │
│                                    │                                    │
│                                    ▼                                    │
│                            GitOpsCluster                                │
│                   (register clusters in ArgoCD +                        │
│                    create acm-placement ConfigMap)                      │
│                                    │                                    │
│                                    ▼                                    │
│  ApplicationSet (clusterDecisionResource generator)                     │
│        │  With pull model annotations:                                  │
│        │    skip-reconcile: "true"                                      │
│        │    ocm-managed-cluster: '{{name}}'                             │
│        │    pull-to-ocm-managed-cluster: "true"                         │
│        │                                                                │
│        └──► Application: todo-app-cluster1                              │
│              (NOT reconciled on hub: just a template)                  │
│                       │                                                 │
│                       ▼                                                 │
│              Pull Model Controller                                      │
│              (wraps Application in ManifestWork)                        │
└─────────────────────────────────────────────────────────────────────────┘
                        │ ManifestWork via OCM agent
                        ▼
                 ┌──────────────┐
                 │   cluster1   │
                 │ (env: prod)  │
                 │              │
                 │  local ArgoCD│──► deploys todo-app
                 │  reconciles  │    to kubernetes.default.svc
                 │  the app     │
                 │  replicas: 3 │
                 │  TLS edge    │
                 └──────────────┘
```

---

## Key annotations and labels for pull model

The following must be present in the ApplicationSet template `metadata`:

| Key | Type | Value | Purpose |
|---|---|---|---|
| `apps.open-cluster-management.io/ocm-managed-cluster` | annotation | `'{{name}}'` | Identifies which managed cluster this Application targets |
| `argocd.argoproj.io/skip-reconcile` | annotation | `"true"` | Prevents the hub ArgoCD from reconciling this Application |
| `apps.open-cluster-management.io/pull-to-ocm-managed-cluster` | label | `"true"` | Enables the pull model controller to select and propagate this Application |

---

## Step 4: Create the ApplicationSet (Pull Model)

> **Manual step**: this is where the pull model differs from the push model.
>
> After this step, the pull model controller propagates the Application to
> managed clusters via ManifestWork. The local ArgoCD on each managed cluster
> then reconciles the application.

```bash
cat <<'EOF' | oc apply -f -
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: todo-app
  namespace: openshift-gitops
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: acm-placement
        labelSelector:
          matchLabels:
            cluster.open-cluster-management.io/placement: todo-app-placement
        requeueAfterSeconds: 180
  template:
    metadata:
      name: todo-app-{{name}}
      annotations:
        apps.open-cluster-management.io/ocm-managed-cluster: '{{name}}'
        argocd.argoproj.io/skip-reconcile: "true"
      labels:
        apps.open-cluster-management.io/pull-to-ocm-managed-cluster: "true"
    spec:
      project: default
      source:
        repoURL: https://github.com/CSA-RH/argodemo1_todo-app.git
        targetRevision: main
        path: kustomize/overlays/production
      destination:
        server: https://kubernetes.default.svc
        namespace: todo-app
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true
EOF
```

### Differences from push model

| Field | Push model | Pull model |
|---|---|---|
| `template.metadata.annotations` | (none required) | `ocm-managed-cluster: '{{name}}'` + `skip-reconcile: "true"` |
| `template.metadata.labels` | (none required) | `pull-to-ocm-managed-cluster: "true"` |
| `destination.server` | `'{{server}}'` (remote cluster URL) | `https://kubernetes.default.svc` (always local) |

---

## Step 5: Verify the deployment

### Check the Application on the hub

The Application should exist on the hub but **not be reconciled** (status will
show as Unknown since hub ArgoCD ignores it):

```bash
oc get applications.argoproj.io -n openshift-gitops | grep todo-app
```

### Check ManifestWork propagation

The pull model controller wraps the Application in a ManifestWork for each
target cluster:

```bash
oc get manifestwork -n cluster1 | grep todo-app
```

### Check the application on the managed cluster

On the managed cluster, the local ArgoCD reconciles the application:

```bash
CLUSTER_NAME=cluster1
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get applications.argoproj.io -n openshift-gitops
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get deployment,route -n todo-app
```

### Check the ACM console

The pull model has native ACM console integration. Navigate to:

**Applications → todo-app**

The topology view should show green status indicators for all resources
(Deployment, Service, Route, ReplicaSet, Pods).

---

## How the pull model flow works

1. **ManagedClusterSetBinding** grants the `openshift-gitops` namespace access
   to the cluster sets.

2. **Placement** evaluates cluster labels and writes the selected clusters to
   `PlacementDecision` resources.

3. **GitOpsCluster** watches the Placement and creates ArgoCD cluster secrets
   and the `acm-placement` ConfigMap.

4. **ApplicationSet** `clusterDecisionResource` generator reads the
   PlacementDecision and generates one Application per matched cluster.
   The Applications have `skip-reconcile` so hub ArgoCD ignores them.

5. **Pull Model Controller** (running on the hub in `open-cluster-management`)
   detects Applications labelled with `pull-to-ocm-managed-cluster: "true"`,
   wraps each one in a **ManifestWork**, and sends it to the target managed
   cluster via the OCM agent channel.

6. **ManifestWork agent** on the managed cluster receives the Application
   resource and creates it in the local `openshift-gitops` namespace.

7. **Local ArgoCD** on the managed cluster reconciles the Application against
   `https://kubernetes.default.svc`, deploying the kustomize overlay locally.

8. **Status feedback** flows back: the managed cluster reports Application
   health/sync status via ManifestWork status, and ACM aggregates this into
   `MulticlusterApplicationSetReport` for display in the ACM console.

---

## Troubleshooting

### Application shows Unknown status on the hub

This is expected. The `argocd.argoproj.io/skip-reconcile: "true"` annotation
tells hub ArgoCD to ignore the Application. Check the managed cluster's local
ArgoCD for the real status.

### ManifestWork not created

Verify the pull model controller is running:

```bash
oc get pods -n open-cluster-management | grep multicluster-integrations
```

Check its logs:

```bash
oc logs -n open-cluster-management -l app=multicluster-integrations --tail=50
```

### Application not appearing on managed cluster

Verify the ManifestWork was applied:

```bash
oc get manifestwork -n cluster1 -o yaml | grep todo-app
```

Verify OpenShift GitOps is running on the managed cluster:

```bash
CLUSTER_NAME=cluster1
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get pods -n openshift-gitops
```

### Managed cluster ArgoCD cannot reach the Git repo

The managed cluster's ArgoCD needs network access to the Git repository.
Check ArgoCD application status on the managed cluster:

```bash
CLUSTER_NAME=cluster1
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get applications.argoproj.io -n openshift-gitops -o yaml | grep -A 5 'status:'
```

---

## References

- [Deploying applications with the Pull model (ACM 2.16)](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html/gitops/gitops-overview)
- [Introducing the Argo CD Application Pull Controller (Red Hat blog)](https://www.redhat.com/en/blog/introducing-the-argo-cd-application-pull-controller-for-red-hat-advanced-cluster-management)
- [ApplicationSet ClusterDecisionResource generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster-Decision-Resource/)
- [Common setup and push model](README.md)
