# Troubleshooting HCP Cluster Deletion

This guide helps ARO SREs diagnose HCP cluster and node pool deletion failures using Kusto queries and break-glass remediations.

## Failure modes

1. [Cluster Stuck in Deleting](#1-cluster-stuck-in-deleting)
2. [Delete Operation Fails](#2-delete-operation-fails)
3. [Orphaned Azure Resources](#3-orphaned-azure-resources)
4. [Node Pool Delete Stuck](#4-node-pool-delete-stuck)

## Prerequisites

- Access to the regional Kusto cluster
- Databases: `ServiceLogs` (RP, CS, Maestro) and `HostedControlPlaneLogs` (HyperShift, CAPZ, CAPI)
- ARM resource ID of the affected cluster or node pool
- `hcpctl mc breakglass` / `hcpctl sc breakglass` for live checks

**Conventions:** Replace placeholders before executing:


| Placeholder                | Example                                                                                                                                                    |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `{KUSTO_CLUSTER_URI}`      | `https://hcp-prod-us.eastus2.kusto.windows.net`                                                                                                            |
| `{SUBSCRIPTION_ID}`        | `64f0619f-ebc2-4156-9d91-c4c781de7e54`                                                                                                                     |
| `{RESOURCE_GROUP}`         | `mytestcluster-net-rg-03`                                                                                                                                  |
| `{CLUSTER_NAME}`           | `mytestcluster`                                                                                                                                            |
| `{RESOURCE_ID}`            | Cluster ARM ID: `/subscriptions/{SUBSCRIPTION_ID}/resourceGroups/{RESOURCE_GROUP}/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/{CLUSTER_NAME}` |
| `{NODEPOOL_NAME}`          | `np-1`                                                                                                                                                     |
| `{NODEPOOL_RESOURCE_ID}`   | Node pool ARM ID: `{RESOURCE_ID}/nodePools/{NODEPOOL_NAME}`                                                                                                |
| `{START_TIME}`             | `datetime(2026-06-17T00:00:00Z)` or `ago(24h)`                                                                                                             |
| `{END_TIME}`               | `datetime(2026-06-18T00:00:00Z)` or `now()`                                                                                                                |
| `{CID}`                    | Clusters Service internal cluster ID, e.g. `2rns63m2qupho9757oflhqje00895crh`                                                                              |
| `{HCP_NS}`                 | Control plane namespace: `ocm-{prefix}-{cid}-{cluster_name}`                                                                                               |
| `{MANAGED_RESOURCE_GROUP}` | Managed RG name                                                                                                                                            |




## Deletion architecture

Cluster deletion flows top-down:

```
ARM DELETE → RP Frontend (Cosmos: Deleting, deletionTimestamp)
  → RP Backend (OperationClusterDelete polls CS)
    → CS state → uninstalling, destruct chain
      → Maestro ResourceBundles deleted
        → Maestro Agent removes ManifestWork (mgmt cluster)
          → HostedCluster / ManagedCluster deletion
            → HyperShift + CAPZ remove Azure resources
              → Namespace + finalizer cleanup

Clusters Service (sets state to "uninstalling")
  → Maestro Server (deletes ResourceBundle)
    → Maestro Agent (deletes ManifestWork on MGMT)
      → ManifestWork cleanup (deletes HostedCluster, ManagedCluster, etc.)
        → ManagedCluster destructor (hypershift-managed-cluster-destructor)
          → ManagedClusterAddon pre-delete hooks (finalizer: hosting-addon-pre-delete)
          → ManagedClusterAddon manifests cleanup (finalizer: hosting-manifests-cleanup)
        → HostedCluster deletion (finalizer: hypershift.openshift.io/finalizer)
          → NodePool deletion (finalizer: hypershift.openshift.io/finalizer)
          → Control Plane namespace cleanup
            → Deployments (finalizer: hypershift.openshift.io/component-finalizer)
            → Cluster CRD (finalizer: cluster.cluster.x-k8s.io)
            → HostedControlPlane (finalizer: hypershift.openshift.io/finalizer)
            → MachineDeployment / MachineSet / Machine (CAPI finalizers)
            → AzureMachine (finalizer: azuremachine.infrastructure.cluster.x-k8s.io)              
```


| Phase                 | ARM `provisioningState` | CS state                 | Signal                                   |
| --------------------- | ----------------------- | ------------------------ | ---------------------------------------- |
| Delete accepted       | `Deleting`              | `ready` → `uninstalling` | Frontend 202, Cosmos `deletionTimestamp` |
| Uninstall in progress | `Deleting`              | `uninstalling`           | Backend polling CS                       |
| CS resource gone      | `Succeeded` (operation) | 404                      | Backend completes operation              |
| ARM resource gone     | GET → 404               | 404                      | Cosmos document deleted                  |


---

## Which database/table to use

Use this table to pick the right database and table before writing KQL. 


| What you're debugging          | Database                 | Table(s)                                |
| ------------------------------ | ------------------------ | --------------------------------------- |
| RP frontend (ARM requests)     | `ServiceLogs`            | `frontendLogs`                          |
| RP backend (async ops)         | `ServiceLogs`            | `backendLogs`                           |
| Cluster Service                | `ServiceLogs`            | `clustersServiceLogs`                   |
| Maestro, general svc pods      | `ServiceLogs`            | `containerLogs`                         |
| HyperShift / HCP control plane | `HostedControlPlaneLogs` | `containerLogs` (in `ocm-*` namespaces) |
| CP-namespace Warning events    | `HostedControlPlaneLogs` | `kubernetesEvents`                      |




## Step 0: Discovery — map Resource ID to internal identifiers

- Run this before layer-specific queries (aligned with Kusto Cookbook Step 0). You need `{CID}` for HyperShift/CAPZ queries.

```kql
let resourceId = "{RESOURCE_ID}";
let start_time = ago(24h);
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp >= start_time
| where log.aro_hcp_cluster_resource_id =~ resourceId
| where tostring(cid) != ''
| sort by timestamp desc
| take 1
| project cid;
```

If CID lookup returns nothing, widen the window to `ago(7d)` or confirm the correct regional Kusto cluster.

- Or you can also find CID using kubectl commands

```
hcpctl mc breakglass <mc-name>
export KUBECONFIG=<path-from-output>
kubectl get hostedcluster -A | grep '{CLUSTER_NAME}'
```

Namespace pattern: `ocm-{prefix}-{CID}` or control plane namespace `ocm-{prefix}-{CID}-{CLUSTER_NAME}`

---



## 1. Cluster Stuck in Deleting

**Symptom:** Cluster stays in `provisioningState: Deleting` well beyond the deletion SLO or the async ARM operation stays `InProgress` with no sign of progress.

### Step 1: HyperShift operator and CAPI manager

This usually helps to rule out whether cluster is stuck in deleting due to node pool deletion not making any progress

```kql
// HyperShift operator — fleet-wide filter by CID
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').containerLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == 'hypershift'
| where container_name == 'operator'
| extend log_str = tostring(log)
| where log_str has '{CID}'
| where log_str has_any ('error', 'failed', 'delet', 'Waiting')
| project timestamp, log_str
| order by timestamp desc
| take 100
```

```kql
// CAPI manager in the hosted cluster namespace (HostedControlPlaneLogs)
cluster('{KUSTO_CLUSTER_URI}').database('HostedControlPlaneLogs').containerLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name has '{CID}'
| where container_name == 'manager'
| extend log_str = tostring(log)
| where log_str has_any ('failed', 'error', 'delet', 'Waiting')
| project timestamp, pod_name, log_str
| order by timestamp desc
| take 100
```



### Breakglass and debug

```bash
# ARM / RP view
az resource show --ids '{RESOURCE_ID}' \
  --query "{provisioningState:properties.provisioningState, deletionTimestamp:properties.serviceProviderProperties.deletionTimestamp}" -o json

# Management cluster — is HyperShift still deleting?
hcpctl mc breakglass <mc-name> && export KUBECONFIG=<path-from-output>
kubectl get hostedcluster -A | grep '{CID}'
kubectl get hostedcluster -n "ocm-${CLUSTER_PREFIX}-{CID}" '{CLUSTER_NAME}' -o jsonpath='{.status.conditions}' | jq '.[] | select(.type|test("Progressing|Degraded|CloudResourcesDestroyed|Available"))'
kubectl get nodepool -n "ocm-${CLUSTER_PREFIX}-{CID}" -o wide
kubectl logs -n hypershift deployment/operator --tail=100 --since=1h | grep -E '{CID}|{CLUSTER_NAME}'

# What is holding the control-plane namespace?
kubectl get ns "{HCP_NS}" -o json | jq '.status.conditions[]? | select(.type=="NamespaceContentRemaining")'
kubectl get azuremachines.infrastructure.cluster.x-k8s.io,machines.cluster.x-k8s.io -n "ocm-${CLUSTER_PREFIX}-{CID}-{CLUSTER_NAME}" -o custom-columns=KIND:.kind,NAME:.metadata.name,DELETION:.metadata.deletionTimestamp,FINALIZERS:.metadata.finalizers
kubectl get azuremachines.infrastructure.cluster.x-k8s.io,machines.cluster.x-k8s.io -n "ocm-${CLUSTER_PREFIX}-{CID}-{CLUSTER_NAME}" -o json
```

Example failures

- Failed to remove finalizer

```
"error":"[failed to remove finalizer from hostedcluster: Operation cannot be fulfilled on hostedclusters.hypershift.openshift.io]
```

Patch the reported resource to clear all finalizers

```
kubectl -n ocm-arohcppers-{CID} patch hostedcluster mytestclus --type=merge -p='{"metadata":{"finalizers":null}}'
```

- failed to delete nodepool: there are still Machines in for NodePool

```
{"level":"error","ts":"2026-07-23T04:40:23Z","msg":"Reconciler error","controller":"nodepool","controllerGroup":"hypershift.openshift.io","controllerKind":"NodePool","NodePool":{"name":"ykulkarn-np-1","namespace":"ocm-arohcppers-2rme6pr00q4spalp20ks1gemk80m1m4b"},"namespace":"ocm-arohcppers-2rme6pr00q4spalp20ks1gemk80m1m4b","name":"ykulkarn-np-1","reconcileID":"f7f974a0-1028-4f9f-9a25-05ad714e2651","error":"failed to delete nodepool: there are still Machines in for NodePool \"ykulkarn-np-1\"","stacktrace":"sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller[...]).reconcileHandler\n\t/hypershift/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:474\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller[...]).processNextWorkItem\n\t/hypershift/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:421\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller[...]).Start.func1.1\n\t/hypershift/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:296"}
```

For node pools stuck in deleting state jump [here](#4-node-pool-delete-stuck). 

---



## 2. Delete Operation Fails

**Symptom:** Operation ends in `Failed`; cluster resource may still exist.

### Step 1: Backend cloud errors

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where resource_group == '{RESOURCE_GROUP}'
| where resource_name == '{CLUSTER_NAME}'
| where isnotempty(cloud_error_code) or isnotempty(cloud_error_message)
| project timestamp, operation, operation_id, cloud_error_code, cloud_error_message, msg, level
| order by timestamp desc
```



### Step 2: Maestro server and agent

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').containerLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == 'maestro'
| where container_name in ('maestro-server', 'maestro-agent')
| extend logs = tostring(log)
| where logs has '{CID}'
| extend msg = extract('] "([^"]+)"', 1, logs)
| extend resource_id = extract('resource[_]?id="([^"]+)"', 1, logs)
| extend resource_name = extract('resource[_]?name="([^"]+)"', 1, logs)
| extend manifest_work = extract('manifestwork[_]?name="([^"]+)"', 1, logs)
| project timestamp, container_name, msg, resource_id, resource_name, manifest_work, logs
| order by timestamp asc
```



### Step 3: CS error state

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.aro_hcp_cluster_resource_id =~ '{RESOURCE_ID}'
| where isempty(log.aro_hcp_node_pool_resource_id)
| where log has 'state to' or log has 'now in'
| project timestamp, msg = tostring(log.msg), level
| order by timestamp asc
```

If CS is already uninstalling and those messages are looping, stop digging in CS further.

### Breakglass and debug

```bash
# Service cluster — CS state + Maestro bundles for this CID
hcpctl sc breakglass <svc-name> && export KUBECONFIG=<path-from-output>

kubectl exec -n clusters-service deployment/clusters-service -c clusters-service-server -- \
  curl -s "http://localhost:8000/api/clusters_mgmt/v1/clusters/{CID}" | \
  jq '{id, name, state, status}'

kubectl exec -n maestro deployment/maestro -c maestro-server -- sh -c \
  "curl -s 'http://localhost:8000/api/maestro/v1/resource-bundles?size=2900'" | \
  jq --arg cid '{CID}' '[.items[] | select(.manifests | tostring | test($cid)) |
    {id, name, deleted_at: .metadata.deleted_at, labels: .metadata.labels, consumer_name}]'

# Namespace-only orphans often block CS destruct
kubectl exec -n maestro deployment/maestro -c maestro-server -- sh -c \
  "curl -s 'http://localhost:8000/api/maestro/v1/resource-bundles?size=2900'" | \
  jq --arg cid '{CID}' '[.items[] | select(.manifests | tostring | test($cid)) |
    select(.metadata.labels.containsNamespaces == "true" or (.name | test("namespaces"))) |
    {id, name, deleted_at: .metadata.deleted_at}]'

# Management cluster — ManifestWork delivery vs missing CP namespace
hcpctl mc breakglass <mc-name> && export KUBECONFIG=<path-from-output>
kubectl get manifestwork -n local-cluster -l "api.openshift.com/id={CID}" -o wide
kubectl get manifestwork -n local-cluster -l "api.openshift.com/id={CID}" -o json | \
  jq '.items[] | {name: .metadata.name, deletionTimestamp: .metadata.deletionTimestamp,
      conditions: [.status.conditions[]? | {type, status, reason, message}]}'
kubectl get ns | grep '{CID}' || echo "CP / HC namespaces already gone"
kubectl logs -n maestro deployment/maestro-agent -c maestro-agent --tail=80 --since=1h | grep '{CID}'
```

Example failures

- Manifest work deletion is stuck
- CID namespace not found
  ```
  "controller failed to sync" err="namespaces \"ocm-arohcpprod-{CID}-{CLUSTER_NAME}\" not found" key="<bundle-id>"
  ```

This means the control plane namespace on the management cluster is already gone, but the Maestro server still has a ResourceBundle pointing at it. The agent keeps trying to sync into a missing namespace and will never self-resolve.

**Diagnosis:** The HostedCluster, NodePool, ManifestWork, and Machines are all gone on the management cluster. The namespace itself is deleted or terminating. CS cannot proceed because Maestro reports the bundle as not delivered.

**Remediation:** Delete the stale resource bundle from the Maestro server — see [fix-maestro-stale-resource-bundle.md](fix-maestro-stale-resource-bundle.md). Once the bundle is removed, CS completes its destruct chain and the deletion pipeline proceeds.

---



## 3. Orphaned Azure Resources

**Symptom:** VMs, NICs, disks, or load balancers remain in the managed resource group after deletion stalls or completes.

This scenario is often **not a Kusto-first problem**. Once the control plane namespace is gone, CAPZ/`capi-provider` pods no longer emit logs — empty CAPZ queries are expected. Use Kusto to find the managed RG and (if the CP was still alive during the window) the Azure delete failure; then inventory and clean up in Azure.

Split the case early:


| Case                         | Signal                                                 | Where the answer lives                                      |
| ---------------------------- | ------------------------------------------------------ | ----------------------------------------------------------- |
| A. Deletion still stuck      | HostedCluster / CP ns still present; CS `uninstalling` | CAPZ + HyperShift + K8s events (Steps 2–4)                  |
| B. Deletion already finished | ARM/CS 404, but managed RG still has resources         | `az` (Step 5) — Kusto only for MRG name + historical errors |




### Step 1: Identify the managed resource group

Cosmos dumps use the **internal** document shape: `properties.customerProperties.platform.`* (not `properties.platform.`*). Prefer the last dump **before** the cluster document disappeared; widen `{START_TIME}` if the cluster is already gone.

```kql
// Preferred: cluster datadump (internal Cosmos shape)
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.content.resourceID =~ '{RESOURCE_ID}'
| summarize content = take_any(log.content), observedTime = max(timestamp) by etag = tostring(log.content._etag)
| top 1 by observedTime desc
| extend content = parse_json(content)
| project observedTime,
          managedResourceGroup = tostring(content.properties.customerProperties.platform.managedResourceGroup),
          subnetId = tostring(content.properties.customerProperties.platform.subnetId),
          nsgId = tostring(content.properties.customerProperties.platform.networkSecurityGroupId)
```

If that returns empty MRG fields, try billing dumps (field is top-level on the billing document) or a string search:

```kql
// Fallback A: billing dump — content.managedResourceGroup
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'billingdump'
| where log.resource_id =~ '{RESOURCE_ID}'
    or log.currentResourceID =~ '{RESOURCE_ID}'
    or tostring(log.content.resourceId) =~ '{RESOURCE_ID}'
| project timestamp,
          managedResourceGroup = tostring(log.content.managedResourceGroup)
| where isnotempty(managedResourceGroup)
| top 1 by timestamp desc
```

```kql
// Fallback B: any backend log that mentioned the field (works after cluster doc is gone)
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where tostring(log) has '{RESOURCE_ID}' and tostring(log) has 'managedResourceGroup'
| extend mrg = extract(@'managedResourceGroup["\']?\s*[:=]\s*["\']?([^"\'\\s,}]+)', 1, tostring(log))
| where isnotempty(mrg)
| distinct mrg
```

```bash
# Fallback C: Azure — RGs managedBy the cluster ARM ID
az group list --subscription '{SUBSCRIPTION_ID}' \
  --query "[?managedBy!=null && contains(to_lower(managedBy), to_lower('{RESOURCE_ID}'))].{name:name, managedBy:managedBy}" -o table
```

Save `{MANAGED_RESOURCE_GROUP}` for later steps.

### Step 2: Confirm whether the control plane was still logging

CAPZ runs in the **control plane namespace** (`{HCP_NS}` = `ocm-{prefix}-{cid}-{cluster_name}`), not the HostedCluster namespace. If this returns nothing, skip to Step 5 — there is no live CAPZ signal to mine.

```kql
// Any capi-provider traffic in the CP namespace during the window?
cluster('{KUSTO_CLUSTER_URI}').database('HostedControlPlaneLogs').table('containerLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == '{HCP_NS}' or namespace_name has '{CID}'
| where pod_name has 'capi-provider-'
| summarize rows = count(), first_seen = min(timestamp), last_seen = max(timestamp) by namespace_name
```

> Prefer exact `namespace_name == '{HCP_NS}'`. `has '{CID}'` is a discovery aid only — it can match the wrong ns if multiple clusters share a prefix collision (rare).



### Step 3: CAPZ Azure delete failures (only if Step 2 returned rows)

Match the `hcpctl snapshot` CAPZ query: parse structured fields, keep **all** azure-controller lines with an `err`, then summarize. Do **not** pre-filter to a short Azure error-code list — that often yields empty results when the extract patterns miss or the failure is phrased differently.

```kql
// Broad CAPZ azure-controller errors (hcpctl clusterAPIProviderLogs pattern)
cluster('{KUSTO_CLUSTER_URI}').database('HostedControlPlaneLogs').table('containerLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == '{HCP_NS}'
| where pod_name has 'capi-provider-'
| extend log_str = tostring(log)
| extend msg = extract(@']\s+"([^"]+)"', 1, log_str)
| extend err = extract(@'err="((?:[^"\\]|\\.)*)"', 1, log_str)
| extend controller = extract(@'controller="([^"]*)"', 1, log_str)
| extend name = extract(@'\sname="([^"]*)"', 1, log_str)
| where controller has 'azure'
| where isnotempty(err)
| summarize first_occurrence = min(timestamp), last_occurrence = max(timestamp), occurrences = count()
    by msg, err, controller, name
| order by occurrences desc
```

If the summarize is still empty, dump raw lines (log format may not match the extract regexes):

```kql
// Raw CAPZ lines — use when structured extracts return nothing
cluster('{KUSTO_CLUSTER_URI}').database('HostedControlPlaneLogs').table('containerLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == '{HCP_NS}'
| where pod_name has 'capi-provider-'
| extend log_str = tostring(log)
| where log_str has_any ('error', 'failed', 'delet', 'AuthorizationFailed', 'ResourceGroupNotFound', 'OperationNotAllowed', 'Throttling', 'AADSTS')
| project timestamp, log_str
| order by timestamp desc
| take 100
```


| Error                             | Likely cause                                                |
| --------------------------------- | ----------------------------------------------------------- |
| `AuthorizationFailed`             | Managed identity lost delete permissions                    |
| `ResourceGroupNotFound`           | Managed RG deleted externally                               |
| `OperationNotAllowed`             | Resource locks or deny assignments                          |
| `Throttling` / `TooManyRequests`  | Azure API rate limits                                       |
| `AADSTS700016` / identity missing | See [§5](#5-deletion-blocked-by-missing-managed-identities) |




### Step 4: HyperShift `CloudResourcesDestroyed` + K8s Warning events

```kql
// HostedCluster conditions from kube-applier readdesire dumps
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.resource_group == '{RESOURCE_GROUP}'
| where log.resource_name == '{CLUSTER_NAME}'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpopenshiftclusters/readdesires'
| summarize content = take_any(log.content), observedTime = take_any(timestamp) by etag = tostring(log.content._etag)
| sort by tolong(content._ts) asc
| extend content = parse_json(content)
| extend manifest = content.properties.status.kubeContent
| where manifest.kind == 'HostedCluster'
| mv-expand condition = manifest.status.conditions
| where tostring(condition.type) in ('CloudResourcesDestroyed', 'InfrastructureReady')
| project observedTime,
          type = tostring(condition.type),
          status = tostring(condition.status),
          reason = tostring(condition.reason),
          message = tostring(condition.message),
          lastTransitionTime = todatetime(condition.lastTransitionTime)
| order by lastTransitionTime desc
```

`CloudResourcesDestroyed=False` means HyperShift never finished Azure cleanup — correlate with CAPZ errors above.

```kql
// Warning events in the CP namespace (often clearer than container logs)
cluster('{KUSTO_CLUSTER_URI}').database('HostedControlPlaneLogs').table('kubernetesEvents')
| where timestamp between ({START_TIME} .. {END_TIME})
| where eventNamespace == '{HCP_NS}' or eventNamespace has '{CID}'
| where kubeEventType == 'Warning'
| project timestamp, objectKind, objectName, reason, message, sourceComponent, count
| order by timestamp desc
| take 100
```



### Live debugging

```bash
# Find managed RG (customer subscription)
az group list --subscription '{SUBSCRIPTION_ID}' \
  --query "[?managedBy!=null && contains(to_lower(managedBy), to_lower('{RESOURCE_ID}'))].{name:name, managedBy:managedBy}" -o table

# Inventory leftovers
az resource list --subscription '{SUBSCRIPTION_ID}' -g '{MANAGED_RESOURCE_GROUP}' \
  --query "[].{name:name, type:type, location:location}" -o table

# Locks / deny that block CAPZ deletes
az lock list --subscription '{SUBSCRIPTION_ID}' -g '{MANAGED_RESOURCE_GROUP}' -o table
az role assignment list --subscription '{SUBSCRIPTION_ID}' -g '{MANAGED_RESOURCE_GROUP}' \
  --include-inherited --query "[?contains(roleDefinitionName, 'Deny') || principalType=='ServicePrincipal'].{principal:principalName, role:roleDefinitionName, scope:scope}" -o table

# If CP ns still exists — live CAPZ signal
hcpctl mc breakglass <mc-name> && export KUBECONFIG=<path-from-output>
kubectl get pods -n "{HCP_NS}" -l cluster.x-k8s.io/provider=infrastructure-azure -o wide
kubectl logs -n "{HCP_NS}" -l cluster.x-k8s.io/provider=infrastructure-azure -c manager --tail=100 --since=1h | \
  grep -iE 'error|failed|AuthorizationFailed|ResourceGroupNotFound|AADSTS|Throttl'
kubectl get azuremachines.infrastructure.cluster.x-k8s.io -n "{HCP_NS}" -o yaml | \
  grep -E 'deletionTimestamp|finalizers|failureMessage|failureReason|providerID' -A2
```

---



## 4. Node Pool Delete Stuck

**Symptom:** Node pool stays in `provisioningState: Deleting` for hours. 

### Step 1: Nodepool Cosmos document state (where is it stuck?)

```kql
cluster('{KUSTO}').database('ServiceLogs').backendLogs
| where timestamp > ago(2d)
| where tostring(log.content.resourceID) =~ '{NODEPOOL_RESOURCE_ID}'
| project timestamp, content = log.content
| order by timestamp desc
| take 10
```

Open the latest content (or project a few fields if you prefer).

Look for:

- properties.provisioningState still Deleting
- serviceProviderProperties.deletionTimestamp set?
- serviceProviderProperties.clusterServiceDeletionTimestamp — null ⇒ CS delete not dispatched
- serviceProviderProperties.clusterServiceID — still set ⇒ waiting for CS 404
- No rows + az resource show on NP is 404 ⇒ doc already gone (delete finished)



### Step 2: Node pool deletion controller conditions

```kql
cluster('{KUSTO}').database('ServiceLogs').backendLogs
| where timestamp > ago(2h)
| where resource_id has '{RESOURCE_ID}'
| project timestamp, msg, cloud_error_code, cloud_error_message, operation, operation_id, resource_id
| order by timestamp desc
| take 50
```

Look for: repeated reconcile errors, OCM4001 / inflight, CS timeouts. Stuck delete scenarios can report no ERROR and be stuck in a loop performing a specific operation which never finishes

### Step 3: Cluster Service view

```kql
// Phase transitions
database('ServiceLogs').clustersServiceLogs
| where timestamp > ago(2h)
| where tostring(log.aro_hcp_node_pool_resource_id) =~ '{NODEPOOL_RESOURCE_ID}' or tostring(log) has '{NODEPOOL_NAME}'
| project timestamp, msg = tostring(log.msg), level
| order by timestamp asc
| take 1000
```

Look for: uninstalling / state transitions; loops; errors. Stuck in uninstalling with no progress ⇒ look downstream (Maestro / HyperShift / CAPZ). Silent after delete ⇒ maybe already 404 in CS.

### Step 4:

```kql
// need {CID} first if you don't have it
cluster('{KUSTO}').database('HostedControlPlaneLogs').containerLogs
| where timestamp > ago(2d)
| where namespace_name has '{CID}'
| where pod_name has 'capi-provider'
| where tostring(log) has '{NODEPOOL_NAME}' or tostring(log) has_any ('error','failed','AADSTS','AuthorizationFailed','OperationNotAllowed')
| project timestamp, pod_name, log
| order by timestamp desc
| take 50
```

Look for: AuthorizationFailed, locks/OperationNotAllowed, AADSTS…, machines stuck deleting. Empty CAPZ + CP ns already gone ⇒ Azure leftovers may still exist; check managed RG with az.

### Live debugging

```bash
# ARM — still deleting? drain timeout / last-pool rejection?
az resource show --ids '{NODEPOOL_RESOURCE_ID}' \
  --query "{provisioningState:properties.provisioningState, nodeDrainTimeoutMinutes:properties.nodeDrainTimeoutMinutes, replicas:properties.replicas}" -o json
az rest --method get \
  --url "https://management.azure.com{RESOURCE_ID}/nodePools?api-version=2024-06-10-preview" | \
  jq '.value[] | {name: .name, state: .properties.provisioningState}'

# Management cluster — NodePool + Machines for this pool
hcpctl mc breakglass <mc-name> && export KUBECONFIG=<path-from-output>
HC_NS="ocm-${CLUSTER_PREFIX}-{CID}"
kubectl get nodepool -n "$HC_NS" '{NODEPOOL_NAME}' -o yaml | \
  jq '{deletionTimestamp: .metadata.deletionTimestamp, finalizers: .metadata.finalizers,
       conditions: [.status.conditions[]? | {type, status, reason, message}]}'
kubectl get machines.cluster.x-k8s.io,azuremachines.infrastructure.cluster.x-k8s.io -n "{HCP_NS}" \
  -l "hypershift.openshift.io/nodePool={NODEPOOL_NAME}" \
  -o custom-columns=KIND:.kind,NAME:.metadata.name,PHASE:.status.phase,DELETION:.metadata.deletionTimestamp,FINALIZERS:.metadata.finalizers

# Drain / eviction stalls (PDB / stuck pods on worker nodes owned by this pool)
kubectl get events -n "{HCP_NS}" --field-selector type=Warning --sort-by='.lastTimestamp' | tail -40
kubectl logs -n "{HCP_NS}" -l cluster.x-k8s.io/provider=infrastructure-azure -c manager --tail=80 --since=1h | \
  grep -iE '{NODEPOOL_NAME}|drain|evict|timeout|failed'
```

Follow [TSG: Node Pool Management](https://eng.ms/docs/cloud-ai-platform/azure-core/azure-cloud-native-and-management-platform/control-plane-bburns/azure-red-hat-openshift/azure-redhat-openshift-team-doc/hcp/troubleshooting/node-pool-management-tsg) which will help with more indepth troubleshooting

---



## 6. Deep dive into cluster deletion workflow



### Live debugging (layer walk)

Use this when Kusto shows the delete was accepted but you need a live “where is it stuck?” sweep. Work top-down; stop at the first broken layer.

```bash
# 1) RP — provisioningState + async op (port-forward on SVC if needed)
hcpctl sc breakglass <svc-name> && export KUBECONFIG=<path-from-output>
kubectl port-forward -n aro-hcp svc/aro-hcp-frontend 8443:8443 &
curl -sk "https://localhost:8443{RESOURCE_ID}?api-version=2024-06-10-preview" | \
  jq '{name: .name, provisioningState: .properties.provisioningState}'

# 2) CS — must be uninstalling (or 404 if CS finished)
kubectl exec -n clusters-service deployment/clusters-service -c clusters-service-server -- \
  curl -s "http://localhost:8000/api/clusters_mgmt/v1/clusters/{CID}" | jq '{id, name, state}'

# 3) Maestro — any leftover bundles (incl. namespace-only)?
kubectl exec -n maestro deployment/maestro -c maestro-server -- sh -c \
  "curl -s 'http://localhost:8000/api/maestro/v1/resource-bundles?size=2900'" | \
  jq --arg cid '{CID}' '[.items[] | select(.manifests | tostring | test($cid)) | {id, name, deleted_at: .metadata.deleted_at}]'

# 4) MGMT — ManifestWork → HostedCluster → CP ns blockers
hcpctl mc breakglass <mc-name> && export KUBECONFIG=<path-from-output>
kubectl get manifestwork,managedcluster -A 2>/dev/null | grep '{CID}'
kubectl get hostedcluster,nodepool -n "ocm-${CLUSTER_PREFIX}-{CID}" -o wide
kubectl get ns "{HCP_NS}" -o json | jq '.status.phase, .status.conditions'
kubectl get hostedcontrolplanes.hypershift.openshift.io,clusters.cluster.x-k8s.io,\
azuremachines.infrastructure.cluster.x-k8s.io,machines.cluster.x-k8s.io -n "{HCP_NS}" \
  -o custom-columns=KIND:.kind,NAME:.metadata.name,DELETION:.metadata.deletionTimestamp,FINALIZERS:.metadata.finalizers
```

Remediation when a layer is confirmed stuck: [cleanup-stuck-cluster-deletion.md](cleanup-stuck-cluster-deletion.md).

### Step 1: Confirm the delete request was accepted

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('frontendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == "aro-hcp"
| where container_name == "aro-hcp-frontend"
| where resource_id =~ "{RESOURCE_ID}"
| where request_method == "delete"
| where msg == "response complete"
| project timestamp, response_status_code, correlation_request_id, client_request_id, duration
| order by timestamp desc
```

**Expected:** `202`. `409` = already deleting. Save `correlation_request_id` for tracing.

### Step 2: ARM operation and service-provider state

Quickly locates which layer the deletion is blocked at before digging deeper.

```kql
// Operation document — is it stuck and does it carry an error?
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.resource_group == '{RESOURCE_GROUP}'
| where log.resource_name == '{CLUSTER_NAME}'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpoperationsstatus'
| summarize content = take_any(log.content), observedTime = min(timestamp) by etag = tostring(log.content._etag)
| sort by observedTime asc
| extend content = parse_json(content)
| where tostring(content.properties.request) == 'Delete'
| project observedTime, operationId = tostring(content.resourceID),
          status = tostring(content.properties.status), error = content.properties.error,
          lastUpdated = todatetime(content.properties.lastUpdatedTime)
```

```kql
// Service-provider state — where did CS dispatch stall?
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.resource_group == '{RESOURCE_GROUP}'
| where log.resource_name == '{CLUSTER_NAME}'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpopenshiftclusters/serviceproviders'
| summarize content = take_any(log.content), observedTime = min(timestamp) by etag = tostring(log.content._etag)
| sort by observedTime asc
| extend content = parse_json(content)
| project observedTime,
          clusterServiceID = tostring(content.properties.clusterServiceID),
          activeOperationID = tostring(content.properties.activeOperationID),
          provisioningState = tostring(content.properties.provisioningState),
          deletionTimestamp = content.properties.deletionTimestamp,
          clusterServiceDeletionTimestamp = content.properties.clusterServiceDeletionTimestamp
```


| Signal                                                          | Likely cause               |
| --------------------------------------------------------------- | -------------------------- |
| `deletionTimestamp` set, `clusterServiceDeletionTimestamp` null | CS delete never dispatched |
| `clusterServiceID` still present                                | CS has not confirmed 404   |
| `activeOperationID` empty while `Deleting`                      | Broken operation link      |




### Step 3: Check child node pools blocking cluster deletion

Cluster deletion waits for all child node pool Cosmos documents to be gone. Node pools do not transition to a `Deleted` state — their documents are removed entirely. An empty result here means no node pools are blocking.

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.content.resourceID has "{RESOURCE_ID}/nodePools"
| summarize content = take_any(log.content), observedTime = max(timestamp) by tostring(log.content.resourceID)
| extend content = parse_json(content)
| project
    observedTime,
    nodePoolId = tostring(content.resourceID),
    provisioningState = tostring(content.properties.provisioningState),
    deletionTimestamp = content.serviceProviderProperties.deletionTimestamp,
    clusterServiceDeletionTimestamp = content.serviceProviderProperties.clusterServiceDeletionTimestamp,
    clusterServiceID = tostring(content.serviceProviderProperties.clusterServiceID),
    usesNewDeletion = content.serviceProviderProperties.usesNewNodePoolDeletionApproach
| where provisioningState == 'Deleting'
```


| Field pattern                                                   | Stuck stage                                                    |
| --------------------------------------------------------------- | -------------------------------------------------------------- |
| `deletionTimestamp` set, `clusterServiceDeletionTimestamp` null | `NodePoolClusterServiceDeleteDispatch`                         |
| `clusterServiceID` still set                                    | Waiting for CS 404 (`NodePoolDeletionClusterServiceIDClearer`) |
| Both CS fields cleared, doc still present                       | Maestro bundles or child docs (`NodePoolDeletionController`)   |


If any node pool is returned, jump to [§4 Node Pool Delete Stuck](#4-node-pool-delete-stuck).

### Step 4: Backend deletion pipeline controller conditions

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.resource_group == '{RESOURCE_GROUP}'
| where log.resource_name == '{CLUSTER_NAME}'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpopenshiftclusters/hcpopenshiftcontrollers'
| where log.content.resourceID has '{CLUSTER_NAME}'
| summarize content = take_any(log.content), observedTime = min(timestamp) by etag = tostring(log.content._etag)
| sort by observedTime asc
| extend content = parse_json(content)
| extend controller_name = extract("/hcpOpenShiftControllers/([^\\/]+)", 1, tostring(content.resourceID))
| where controller_name in (
    'OperationClusterDelete',
    'ClusterClusterServiceDeleteDispatch',
    'ClusterDeletionClusterServiceIDClearer',
    'ClusterChildResourcesCleanupController',
    'ClusterDeletionController'
  )
| mv-expand condition = content.properties.status.conditions
| project observedTime, lastTransitionTime = todatetime(condition.lastTransitionTime),
          controller_name, type = tostring(condition.type), status = tostring(condition.status),
          reason = tostring(condition.reason), message = tostring(condition.message)
| summarize observedTime = min(observedTime) by lastTransitionTime, controller_name, type, status, reason, message
| order by lastTransitionTime asc
```

**Look for:** any controller with `Degraded=True`. The `controller_name`, `reason`, and `message` columns together identify which pipeline stage broke and why:


| Controller                               | `Degraded=True` means                                 |
| ---------------------------------------- | ----------------------------------------------------- |
| `ClusterClusterServiceDeleteDispatch`    | Could not dispatch delete to CS                       |
| `ClusterDeletionClusterServiceIDClearer` | CS polling failed or timed out                        |
| `ClusterChildResourcesCleanupController` | Child Cosmos doc or Maestro bundle cleanup failed     |
| `ClusterDeletionController`              | Final Cosmos delete failed, or a gate is not clearing |
| `OperationClusterDelete`                 | Could not update the ARM operation status             |




### Step 5: CS deletion activity

```kql
// Full CS deletion timeline — read top-to-bottom to see what CS did and where it stopped
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.aro_hcp_cluster_resource_id =~ '{RESOURCE_ID}'
| where isempty(log.aro_hcp_node_pool_resource_id)
| project timestamp, level, msg = tostring(log.msg)
| order by timestamp asc
```

```kql
// Errors and key deletion events only — use if the full timeline is too noisy
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.aro_hcp_cluster_resource_id =~ '{RESOURCE_ID}'
| where isempty(log.aro_hcp_node_pool_resource_id)
| where level == 'ERROR'
    or log.msg has_any ('state', 'destruct', 'delet', 'chain', 'ManifestWork', 'uninstall', 'Skipping')
| project timestamp, level, msg = tostring(log.msg)
| order by timestamp asc
```

**Stuck signal:** The last log entry shows where CS stopped. Look for: `"Running chain deletion to clean deleted cluster"` repeating, `"Not continuing to the next destructor"`, `"Skipping ManifestWork ... as it contains namespaces"` (orphaned namespace bundles — see [fix-maestro-stale-resource-bundle.md](fix-maestro-stale-resource-bundle.md)).

### Step 6: Maestro transition matrix

Localizes service ↔ management cluster delivery failures. See [query cookbook § Maestro](../ai/query-cookbook.md) for the full 7-layer query; abbreviated pattern:

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').containerLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == 'maestro'
| where container_name in ('maestro-server', 'maestro-agent')
| extend logs = tostring(log)
| where logs has '{CID}'
| extend msg = extract('] "([^"]+)"', 1, logs)
| summarize count = count() by container_name, msg
| order by count desc
```

For deletion, confirm delete events reach the agent (`Received event`, `Server side applied`). A break between adjacent layers in the full matrix query indicates where delivery stopped.

**Common patterns:** `"Waiting for namespace deletion"` (CP namespace terminating); `"hostedcluster is still deleting"` (finalizers); `ResourceGroupNotFound` (managed RG deleted externally).

**Remediation:** See [cleanup-stuck-cluster-deletion.md](cleanup-stuck-cluster-deletion.md).

