# ACM ApplicationSet Push Model

> **Common setup (Steps 1–3)** is documented in the
> [ACM Bootstrap: GitOps Application Prerequisites](../../../acm_bootstrap/applications/README.md).
> See also the [overview](../README.md) for common concepts.
> Complete those steps before proceeding here.

Deploy a sample application (`todo-app`) to managed clusters using the
**push model**. Hub ArgoCD deploys directly to managed clusters via the
cluster-proxy-addon or API server URL.

Repository: [argodemo1_todo-app](https://github.com/CSA-RH/argodemo1_todo-app)

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Hub Cluster (openshift-gitops namespace)                        │
│                                                                  │
│  ManagedClusterSetBinding ──► Placement ──► PlacementDecision    │
│        (bind sets)           (select clusters)   (output)        │
│                                    │                             │
│                                    ▼                             │
│                            GitOpsCluster                         │
│                   (register clusters in ArgoCD as                │
│                    cluster secrets + creates the                 │
│                    acm-placement ConfigMap)                      │
│                                    │                             │
│                                    ▼                             │
│  ApplicationSet (clusterDecisionResource generator)              │
│        │  reads ConfigMap ──► knows how to parse                 │
│        │                      PlacementDecision                  │
│        │  reads PlacementDecision ──► gets cluster names         │
│        │  reads ArgoCD cluster secrets ──► gets server URLs      │
│        │                                                         │
│        └──► Application: todo-app-cluster1                       │
│              destination: {{server}} (cluster1 API URL)          │
└──────────────────────────────────────────────────────────────────┘
          │
          ▼
   ┌──────────────┐
   │   cluster1   │
   │ (env: prod)  │
   │  replicas: 3 │
   │  TLS edge    │
   └──────────────┘
```

---

## Prerequesites

Make sure the Hub cluster has the OpenShift-Gitops operator installed and the ACM topology is created. If not first configure it as per: 

```text
https://github.com/CSA-RH/acm_bootstrap/blob/main/README.md
```

---

## Step 4: Create the ApplicationSet (Push Model)

The ApplicationSet uses the `clusterDecisionResource` generator to read
the `PlacementDecision` output and create one ArgoCD Application per cluster.

```bash
oc create -f ../argocd/applicationset-push.yaml
```

---

## Step 5: Verify the deployment

### Check ArgoCD Applications

```bash
oc get applications.argoproj.io -n openshift-gitops | grep todo-app
```

Expected output:

```
todo-app-cluster1   Synced   Healthy
```

### Check the workloads on the managed cluster

```bash
CLUSTER_NAME=cluster1
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get deployment,route -n todo-app
```

### Check the ACM console

Navigate to **Applications → todo-app**. The topology view should show green
status indicators for all resources.

> **Note:** The ACM console relies on the Search API to display push model
> application status. If the search-collector on a managed cluster has stale
> API discovery (visible as `stale GroupVersion discovery` in the collector
> logs), restart the collector pod on the affected cluster to fix missing
> resource types (e.g. Routes).

---

## How the push model flow works

1. **ManagedClusterSetBinding** grants the `openshift-gitops` namespace access
   to the cluster sets.

2. **Placement** evaluates cluster labels and writes the selected clusters to
   `PlacementDecision` resources.

3. **GitOpsCluster** watches the Placement and:
   - Creates ArgoCD cluster **Secrets** for each selected cluster (name,
     server URL, credentials, cluster labels).
   - Creates the **`acm-placement` ConfigMap** that maps the PlacementDecision
     GVK to the fields ArgoCD needs to read.

4. **ApplicationSet** `clusterDecisionResource` generator:
   - Reads the `acm-placement` ConfigMap → learns to read `placementdecisions`
     and extract `status.decisions[].clusterName`.
   - Lists `PlacementDecision` resources matching the `labelSelector`.
   - For each cluster name in the decisions, looks up the corresponding ArgoCD
     cluster secret to get the `server` URL and labels.
   - Generates one ArgoCD Application per matched cluster using the template.

5. **ArgoCD** syncs each Application, deploying the kustomize overlay to the
   target cluster via the cluster-proxy-addon.

When a new cluster is added to a ManagedClusterSet and matches the Placement
predicates, the entire chain fires automatically: ACM creates the ArgoCD
cluster secret and the ApplicationSet creates a new Application for it.

---

## Troubleshooting

### ACM console shows "missing" or "Unknown" status

- Verify the Application uses `source` (singular), not `sources` (plural).
- Check the Search API indexes the Application correctly:

```bash
TOKEN=$(oc whoami -t)
oc exec -n open-cluster-management $(oc get pod -n open-cluster-management -l app=search -l name=search-api -o name | head -1) -- \
  curl -sk -X POST https://localhost:4010/searchapi/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"query":"{ search(input:[{filters:[{property:\"kind\",values:[\"Application\"]},{property:\"apigroup\",values:[\"argoproj.io\"]},{property:\"name\",values:[\"todo-app-cluster1\"]}]}]) { items } }"}'
```

Verify `path`, `repoURL`, and `targetRevision` are not empty in the result.

### Route shows "Not Deployed" in ACM topology

The search-collector on the managed cluster may have stale API discovery.
Check the collector logs:

```bash
CLUSTER_NAME=cluster1
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig logs -n open-cluster-management-agent-addon \
  -l component=search-collector --tail=20
```

If you see `stale GroupVersion discovery: route.openshift.io/v1`, restart the
collector pod:

```bash
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig delete pod -n open-cluster-management-agent-addon \
  -l component=search-collector
```

### ApplicationSet RBAC error for PlacementDecisions

In ACM 2.16/2.17 with OpenShift GitOps 1.17+, the Role
`openshift-gitops-applicationset-controller-placement` may only have `list`
permission for `placementdecisions`. Patch it to add `get` and `watch`:

```bash
oc patch role openshift-gitops-applicationset-controller-placement \
  -n openshift-gitops --type=json \
  -p '[{"op":"replace","path":"/rules/0/verbs","value":["get","list","watch"]}]'
```

Then restart the ApplicationSet controller:

```bash
oc delete pod -n openshift-gitops -l app.kubernetes.io/name=argocd-applicationset-controller
```
