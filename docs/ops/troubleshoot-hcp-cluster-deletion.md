# Troubleshooting HCP Cluster Deletion

This guide helps ARO SREs diagnose HCP cluster and node pool deletion failures using Kusto queries. It is the **KQL companion** to break-glass remediation in [cleanup-stuck-cluster-deletion.md](cleanup-stuck-cluster-deletion.md).

Correlated Microsoft TSG sources (do not duplicate their runbook steps here):


| Source                                                                                                                                                                                                                    | Scope                                                              |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| [HCP Cluster Kusto Cookbook](https://dev.azure.com/msazure/One/_git/Azure-Documents-Common?path=/Teams/Azure%20RedHat%20OpenShift/doc/hcp/troubleshooting/aro-hcp-cluster-kusto-cookbook.md)                              | Discovery queries, layer sweeps, schema gotchas                    |
| [Node Pool Management TSG](https://dev.azure.com/msazure/One/_git/Azure-Documents-Common?path=/Teams/Azure%20RedHat%20OpenShift/doc/hcp/troubleshooting/node-pool-management-tsg.md)                                      | Node pool update/delete symptoms, drain timeout, validation errors |
| [Cluster Deletion — Managed Identities Not Found](https://dev.azure.com/msazure/One/_git/Azure-Documents-Common?path=/Teams/Azure%20RedHat%20OpenShift/doc/hcp/runbooks/cluster-deletion/managed-identities-not-found.md) | CAPZ AAD errors, AzureMachine finalizer cleanup                    |
| [ARO-HCP query cookbook](../ai/query-cookbook.md)                                                                                                                                                                         | `hcpctl snapshot` canned queries                                   |




## Failure modes

1. [Cluster Stuck in Deleting](#1-cluster-stuck-in-deleting)
2. [Delete Operation Fails](#2-delete-operation-fails)
3. [Orphaned Azure Resources](#3-orphaned-azure-resources)
4. [Node Pool Delete Stuck](#4-node-pool-delete-stuck)
5. [Deletion Blocked by Missing Managed Identities](#5-deletion-blocked-by-missing-managed-identities)



## Prerequisites

- Access to the regional Kusto cluster (see [logging.md](../logging.md) or the Kusto Cookbook cluster table)
- Databases: `ServiceLogs` (RP, CS, Maestro) and `HostedControlPlaneLogs` (HyperShift, CAPZ, CAPI)
- ARM resource ID of the affected cluster or node pool
- `hcpctl mc breakglass` / `hcpctl sc breakglass` for live checks

> **Conventions:** Replace placeholders before running:
>
>
> | Placeholder                   | Example                                                 |
> | ----------------------------- | ------------------------------------------------------- |
> | `{KUSTO_CLUSTER_URI}`         | `https://hcp-int-uk.uksouth.kusto.windows.net`          |
> | `{SUBSCRIPTION_ID}`           | `64f0619f-ebc2-4156-9d91-c4c781de7e54`                  |
> | `{RESOURCE_GROUP}`            | `my-rg`                                                 |
> | `{CLUSTER_NAME}`              | `my-cluster`                                            |
> | `{RESOURCE_ID}`               | Full cluster ARM resource ID                            |
> | `{NODEPOOL_NAME}`             | `np-1`                                                  |
> | `{NODEPOOL_RESOURCE_ID}`      | Full node pool ARM resource ID                          |
> | `{START_TIME}` / `{END_TIME}` | `datetime(2026-06-17T00:00:00Z)` or `ago(24h)`          |
> | `{CORRELATION_ID}`            | From frontend delete response                           |
> | `{ASYNC_OP_ID}`               | Operation ID segment from `Azure-AsyncOperation` header |
> | `{CID}`                       | Clusters Service internal cluster ID                    |
> | `{HCP_NS}`                    | Control plane ns: `ocm-{prefix}-{cid}-{cluster_name}`   |
> | `{MANAGED_RESOURCE_GROUP}`    | Managed RG name (from Step 1 in §3)                     |
>

> **Schema gotcha** (from Kusto Cookbook): `frontendLogs.level` and `backendLogs.level` are **UPPERCASE** — use `level == "ERROR"`, not `"error"`.



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



## Step 0: Discovery — map Resource ID to internal identifiers

Run this before layer-specific queries (aligned with Kusto Cookbook Step 0). You need `{CID}` for HyperShift/CAPZ queries.

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

Example failures

- Failed to remove finalizer

```
failed to remove finalizer from hostedcluster: hostedclusters.hypershift.openshift.io \"{CLUSTER_NAME}\" not found"
```

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

Example failures

- Manifest work deletion is stuck



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

Example failures

- CID namespace not found
  ```
  "controller failed to sync" err="namespaces \"ocm-arohcpprod-{CID}-{CLUSTER_NAME}\" not found" key="<bundle-id>"

  ```

This means the control plane namespace on the management cluster is already gone, but the Maestro server still has a ResourceBundle pointing at it. The agent keeps trying to sync into a missing namespace and
will never self-resolve.

**Diagnosis:** The HostedCluster, NodePool, ManifestWork, and Machines are all gone on the management cluster. The namespace itself is deleted or terminating. CS cannot proceed because Maestro reports the bu
ndle as not delivered.

**Remediation:** Delete the stale resource bundle from the Maestro server — see [fix-maestro-stale-resource-bundle.md](fix-maestro-stale-resource-bundle.md). Once the bundle is removed, CS completes its dest
ruct chain and the deletion pipeline proceeds.

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

CS logs are only mildly helpful for delete failures. They rarely contain the root cause. If CS is already uninstalling and those messages are looping, stop digging in CS further.

---



## 3. Orphaned Azure Resources

**Symptom:** VMs, NICs, disks, or load balancers remain in the managed resource group after deletion stalls or completes.

This scenario is often **not a Kusto-first problem**. Once the control plane namespace is gone, CAPZ/`capi-provider` pods no longer emit logs — empty CAPZ queries are expected. Use Kusto to find the managed RG and (if the CP was still alive during the window) the Azure delete failure; then inventory and clean up in Azure.

Split the case early:

| Case | Signal | Where the answer lives |
| ---- | ------ | ---------------------- |
| A. Deletion still stuck | HostedCluster / CP ns still present; CS `uninstalling` | CAPZ + HyperShift + K8s events (Steps 2–4) |
| B. Deletion already finished | ARM/CS 404, but managed RG still has resources | Azure Resource Graph / `az` (Step 5) — Kusto only for MRG name + historical errors |


### Step 1: Identify the managed resource group

Cosmos dumps use the **internal** document shape: `properties.customerProperties.platform.*` (not `properties.platform.*`). Prefer the last dump **before** the cluster document disappeared; widen `{START_TIME}` if the cluster is already gone.

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


| Error                            | Likely cause                             |
| -------------------------------- | ---------------------------------------- |
| `AuthorizationFailed`            | Managed identity lost delete permissions |
| `ResourceGroupNotFound`          | Managed RG deleted externally            |
| `OperationNotAllowed`            | Resource locks or deny assignments       |
| `Throttling` / `TooManyRequests` | Azure API rate limits                    |
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
| order by lastTransitionTime asc
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


### Step 5: Inventory what is actually orphaned (Azure)

Run this whenever Step 2 is empty, or after you have `{MANAGED_RESOURCE_GROUP}` from Step 1. Resource Graph / `az` is the source of truth for leftovers.

```kql
// Azure Resource Graph Explorer (not regional HCP Kusto)
Resources
| where subscriptionId =~ '{SUBSCRIPTION_ID}'
| where resourceGroup =~ '{MANAGED_RESOURCE_GROUP}'
| project name, type, location
| order by type asc, name asc
```

```bash
az resource list --subscription '{SUBSCRIPTION_ID}' -g '{MANAGED_RESOURCE_GROUP}' -o table
```


| Orphan type | Likely cause | Fix |
| ----------- | ------------ | --- |
| VMs / VMSS | CAPZ could not delete AzureMachine (auth/lock/identity) | Fix RBAC/locks or remove AzureMachine finalizers; then delete VMs |
| NICs / disks | Partial CAPZ cleanup | Delete remaining NICs/disks after VMs are gone |
| Load balancer / DNS zone | CS destructor skipped or failed | Delete manually; check CS destruct-chain messages |
| Empty managed RG | Resources gone, RG not deleted | `az group delete -n '{MANAGED_RESOURCE_GROUP}'` |
| UAMIs in **customer** RG | Pre-created by customer | Not cleaned by ARO-HCP by design |

Public-cloud managed RGs may have deny assignments — escalate for FPA cleanup if direct delete is blocked. Break-glass remediation: [cleanup-stuck-cluster-deletion.md](cleanup-stuck-cluster-deletion.md).

---



## 4. Node Pool Delete Stuck

**Symptom:** Node pool stays in `provisioningState: Deleting` for hours. Covered in detail by the [Node Pool Management TSG](https://dev.azure.com/msazure/One/_git/Azure-Documents-Common?path=/Teams/Azure%20RedHat%20OpenShift/doc/hcp/troubleshooting/node-pool-management-tsg.md) (drain timeout, last-pool validation, CAPI machine state).

**Not a Kusto failure:** `"The last node pool can not be deleted"` is an admission/validation rejection — customer must add a second node pool first.

### Step 1: Confirm delete accepted

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('frontendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where resource_id =~ '{NODEPOOL_RESOURCE_ID}'
| where request_method == 'delete'
| where msg == 'response complete'
| project timestamp, response_status_code, correlation_request_id, client_request_id
| order by timestamp desc
```



### Step 2: Node pool Cosmos document state

Node pools do not transition to a `Deleted` state — the document is removed entirely when deletion completes. If this query returns no rows, the node pool document is already gone (success).

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.content.resourceID =~ '{NODEPOOL_RESOURCE_ID}'
| summarize content = take_any(log.content), observedTime = max(timestamp) by tostring(log.content.resourceID)
| extend content = parse_json(content)
| project observedTime,
          provisioningState = tostring(content.properties.provisioningState),
          deletionTimestamp = content.serviceProviderProperties.deletionTimestamp,
          clusterServiceDeletionTimestamp = content.serviceProviderProperties.clusterServiceDeletionTimestamp,
          clusterServiceID = tostring(content.serviceProviderProperties.clusterServiceID),
          usesNewDeletion = content.serviceProviderProperties.usesNewNodePoolDeletionApproach
```


| Field pattern                                                   | Stuck stage                                                    |
| --------------------------------------------------------------- | -------------------------------------------------------------- |
| `deletionTimestamp` set, `clusterServiceDeletionTimestamp` null | `NodePoolClusterServiceDeleteDispatch`                         |
| `clusterServiceID` still set                                    | Waiting for CS 404 (`NodePoolDeletionClusterServiceIDClearer`) |
| Both CS fields cleared, doc still present                       | Maestro bundles or child docs (`NodePoolDeletionController`)   |




### Step 3: Node pool deletion controller conditions

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.resource_group == '{RESOURCE_GROUP}'
| where log.resource_name == '{CLUSTER_NAME}'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpopenshiftclusters/nodepools/hcpopenshiftcontrollers'
| where log.content.resourceID has '{NODEPOOL_NAME}'
| summarize content = take_any(log.content), observedTime = min(timestamp) by etag = tostring(log.content._etag)
| sort by observedTime asc
| extend content = parse_json(content)
| extend controller_name = extract("/hcpOpenShiftControllers/([^\\/]+)", 1, tostring(content.resourceID))
| mv-expand condition = content.properties.status.conditions
| project observedTime, lastTransitionTime = todatetime(condition.lastTransitionTime), controller_name,
          type = tostring(condition.type), status = tostring(condition.status),
          reason = tostring(condition.reason), message = tostring(condition.message)
| summarize observedTime = min(observedTime) by lastTransitionTime, controller_name, type, status, reason, message
| order by lastTransitionTime asc
```

**Controllers:** `OperationNodePoolDelete`, `NodePoolClusterServiceDeleteDispatch`, `NodePoolDeletionClusterServiceIDClearer`, `NodePoolChildResourcesCleanupController`, `NodePoolDeletionController`.

### Step 4: CS node pool state and phases

```kql
// Phase transitions
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.aro_hcp_node_pool_resource_id =~ '{NODEPOOL_RESOURCE_ID}'
| where log.msg has 'with state' or log.msg has 'state updated from'
| project timestamp, msg = tostring(log.msg)
| order by timestamp asc
```

```kql
// Full CS node pool object (csstatedump)
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'csstatedump'
| where log.msg == 'cluster-service node pool state dump'
| where log.hcp_nodepool_name =~ '{NODEPOOL_NAME}'
| summarize csNodePool = take_any(log.csNodePool) by timestamp
| sort by timestamp asc
| project csNodePool
```



### Step 5: Maestro readonly bundles blocking delete

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpopenshiftclusters/nodepools/serviceproviders'
| where log.content.resourceID has '{NODEPOOL_NAME}'
| summarize content = take_any(log.content), observedTime = min(timestamp) by etag = tostring(log.content._etag)
| top 1 by observedTime desc
| extend content = parse_json(content)
| project observedTime, maestroReadonlyBundles = content.status.maestroReadonlyBundles
```

Non-empty `maestroReadonlyBundles` blocks `NodePoolDeletionController`.

### Step 6: CAPZ and drain stalls

If the node pool's control plane namespace still has CAPZ traffic, run the CAPZ queries from [§3 Steps 2–3](#step-2-confirm-whether-the-control-plane-was-still-logging) with `{HCP_NS}` / `{CID}` and filter raw lines for `{NODEPOOL_NAME}`.

**Drain-related stalls:** If `nodeDrainTimeoutMinutes` is `0`, delete waits indefinitely for PDB-blocked evictions — confirm via `az resource show` on the node pool properties (Node Pool Management TSG Step 6).

---



## 5. Deletion Blocked by Missing Managed Identities

**Symptom:** Cluster or node pool deletion stuck; CAPZ logs show AAD errors like `AADSTS700016: Application with identifier '...' was not found`. Full remediation is in the [Managed Identities Not Found runbook](https://dev.azure.com/msazure/One/_git/Azure-Documents-Common?path=/Teams/Azure%20RedHat%20OpenShift/doc/hcp/runbooks/cluster-deletion/managed-identities-not-found.md).

### Step 1: Confirm backend saw a delete failure

Modern equivalent of the legacy `HCPServiceLogs.kubesystem` query (use `backendLogs` in production):

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').table('backendLogs')
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where resource_id =~ '{RESOURCE_ID}' or cluster_id == '{CID}'
| where operation == 'Delete' or log has 'Delete'
| where level == 'ERROR' or isnotempty(cloud_error_code)
| project timestamp, level, msg, operation, operation_id, resource_id, cloud_error_code, cloud_error_message
| order by timestamp asc
```



### Step 2: CAPZ AAD / identity errors

```kql
cluster('{KUSTO_CLUSTER_URI}').database('HostedControlPlaneLogs').containerLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name has '{CID}'
| where pod_name startswith 'capi-provider-'
| extend log_str = tostring(log)
| where log_str has_any ('AADSTS', 'AuthorizationFailed', 'Application with identifier', 'was not found')
| project timestamp, log_str
| order by timestamp desc
| take 50
```

**Next steps:** Break-glass to management cluster, inspect `AzureMachine` CRs in the control-plane namespace, remove finalizers only when machines have `deletionTimestamp` — see the external runbook above and [cleanup-stuck-cluster-deletion.md](cleanup-stuck-cluster-deletion.md).

---



## 6. Deep dive into cluster deletion workflow



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

## Appendix A: Kusto cluster URIs


| Environment | Example cluster URI                                         |
| ----------- | ----------------------------------------------------------- |
| INT (UK)    | `https://hcp-int-uk.uksouth.kusto.windows.net`              |
| PROD (US)   | `https://hcp-prod-us.eastus2.kusto.windows.net`             |
| Pattern     | `https://hcp-{env}-{geoShortId}.{region}.kusto.windows.net` |


See [logging.md](../logging.md) and the [Kusto Cookbook](https://dev.azure.com/msazure/One/_git/Azure-Documents-Common?path=/Teams/Azure%20RedHat%20OpenShift/doc/hcp/troubleshooting/aro-hcp-cluster-kusto-cookbook.md) for the full geo table.

## Appendix B: Key tables


| Table                                  | Database               | Deletion debugging                                             |
| -------------------------------------- | ---------------------- | -------------------------------------------------------------- |
| `frontendLogs`                         | ServiceLogs            | DELETE 202, async op polling                                   |
| `backendLogs`                          | ServiceLogs            | `datadump`, `billingdump`, `csstatedump`, controller conditions |
| `clustersServiceLogs`                  | ServiceLogs            | CS phase transitions, `cid`, destruct chain                    |
| `containerLogs`                        | Both                   | Maestro, HyperShift operator                                   |
| `HostedControlPlaneLogs.containerLogs` | HostedControlPlaneLogs | CAPZ (`capi-provider`), CAPI manager                           |
| `kubernetesEvents`                     | HostedControlPlaneLogs | CP-namespace Warning events during Azure cleanup               |


> **Datadump shape gotcha:** cluster Cosmos dumps use `log.content.properties.customerProperties.platform.managedResourceGroup`, not `properties.platform.*`.




## Appendix C: Related docs

- [Cleanup Procedure for Stuck Cluster Deletion](cleanup-stuck-cluster-deletion.md) — break-glass remediation
- [Fix Maestro Stale Resource Bundle](fix-maestro-stale-resource-bundle.md)
- [Kusto Query Cookbook](../ai/query-cookbook.md) — `hcpctl snapshot` query index
- [HCP Cluster Creation Flow](hcp-cluster-creation-flow.md) — creation path (deletion is the inverse)

