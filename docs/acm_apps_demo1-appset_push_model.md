# Deploy Workload ToDo list using ApplicationSet Push Model

> **Common setup (Steps 1–3)** is documented in the
> [ACM Bootstrap: GitOps Application Prerequisites](../../../acm_bootstrap/applications/README.md).
> See also the [overview](../README.md) for common concepts.
> Complete those steps before proceeding here.

Deploy a sample application (`todo-app`) to managed clusters using the
**push model**.
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

## Procedure

### 1. Prerequesites

Make sure the Hub cluster has the OpenShift-Gitops operator installed and the ACM topology is created. If not first configure it as per: 

```bash
https://github.com/CSA-RH/acm_bootstrap/blob/main/README.md
```

### 2. Clone repo

```bash
git clone https://github.com/CSA-RH/argodemo1_todo-app.git
cd argodemo1_todo-app
```

### 3. Create the Placement that will select the managed clusters to deploy the application

```bash
oc create -f argocd/placement-todo-app.yaml
```


### 4. Create the ApplicationSet

```bash
oc create -f argocd/applicationset-push.yaml
```

### 5. Verify the deployment

**Check ArgoCD Applications and ApplicationSet**

```bash
oc get applicationsets.argoproj.io -n openshift-gitops 
oc get applications.argoproj.io -n openshift-gitops | grep todo-app
```

Expected output:
```
todo-app-cluster1   Synced   Healthy
```

**Check the workloads on the managed cluster**

```bash
CLUSTER_NAME=cluster1
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get deployment,route -n todo-app
```

**Check the ACM console**

Navigate to **Applications → todo-app**. The topology view should show green
status indicators for all resources.
---
