# Troubleshooting HCP Cluster Deletion

This guide helps ARO SREs diagnose and resolve HCP cluster deletion failures using Kusto queries. It covers four failure modes:

1. [Cluster Stuck in Deleting](#1-cluster-stuck-in-deleting)
2. [Delete Operation Never Completes, Still Listed](#2-delete-operation-never-completes-still-listed)
3. [Delete Operation Fails](#3-delete-operation-fails)
4. [Orphaned Azure Resources](#4-orphaned-azure-resources)

## Prerequisites

- Access to the regional Kusto cluster (e.g. `hcp-prod-usc`, `hcp-int-uk`)
- Familiarity with the two Kusto databases:
  - `ServiceLogs` — frontend, backend, Clusters Service, fleet logs
  - `HostedControlPlaneLogs` — HyperShift, control-plane-operator, management cluster logs
- The ARM resource ID of the affected cluster, e.g.:
`/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/{name}`
- Access to management and service clusters via `hcpctl mc breakglass` / `hcpctl sc breakglass` for live remediation

> **Conventions:** All queries below use placeholder variables. Replace these before running:
>
>
> | Placeholder                   | Example                                        |
> | ----------------------------- | ---------------------------------------------- |
> | `{KUSTO_CLUSTER_URI}`         | `https://hcp-int-uk.uksouth.kusto.windows.net` |
> | `{SUBSCRIPTION_ID}`           | `64f0619f-ebc2-4156-9d91-c4c781de7e54`         |
> | `{RESOURCE_GROUP}`            | `my-rg`                                        |
> | `{CLUSTER_NAME}`              | `my-cluster`                                   |
> | `{RESOURCE_ID}`               | Full ARM resource ID                           |
> | `{START_TIME}` / `{END_TIME}` | `datetime(2026-06-17T00:00:00Z)`               |
> | `{ASYNC_OP_PATH}`             | The path from `Azure-AsyncOperation` header    |
>

## Deletion Architecture Overview

Understanding the deletion chain is essential. Deletion flows through these layers in order:

```
ARM DELETE → RP Frontend (Cosmos: op=Deleting, DeletionTimestamp)
  → RP Backend polls Cluster Service (CS) every 10s
    → CS sets state to "uninstalling", starts destruct chain
      → CS deletes Maestro ResourceBundles
        → Maestro Agent deletes ManifestWork on Management Cluster
          → ManifestWork cleanup triggers HostedCluster, ManagedCluster deletion
            → HyperShift removes Azure resources from managed RG
              → Namespace cleanup, finalizer removal
```

**Key rule:** Blocking at *any* level prevents cleanup of everything above it. Deleting management cluster resources before the source (CS/Maestro) is cleared causes them to be **recreated**.

### Provisioning State Lifecycle (Delete)


| Phase                 | ARM `provisioningState` | CS State                 | Signal                                        |
| --------------------- | ----------------------- | ------------------------ | --------------------------------------------- |
| Delete accepted       | `Deleting`              | `ready` → `uninstalling` | Frontend Cosmos transaction                   |
| Uninstall in progress | `Deleting`              | `uninstalling`           | Backend polls CS every ~10s                   |
| CS cluster gone       | `Succeeded`             | 404                      | Backend calls `SetDeleteOperationAsCompleted` |
| Resource removed      | GET → 404               | 404                      | Cosmos cluster doc deleted                    |


---

## 1. Cluster Stuck in Deleting

**Symptom:** The cluster remains in `provisioningState: Deleting` and never reaches a terminal state, hanging well beyond the expected deletion SLO. E2E tests use a 45-minute timeout for deletion.

### Step 1: Confirm the Delete Request Was Accepted

Verify the ARM DELETE reached the frontend and returned 202 Accepted.

```kql
// Find the delete request in frontend logs
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').frontendLogs
| where timestamp > ago(6h)
| where namespace_name == "aro-hcp"
| where container_name == "aro-hcp-frontend"
| where cluster_name has "/subscriptions/{SUBSCRIPTION_ID}/resourcegroups/{RESOURCE_GROUP}/providers/microsoft.redhatopenshift/hcpopenshiftclusters/{CLUSTER_NAME}"
| where request_method == "delete"
| where msg == "response complete"
| where response_status_code == 202
| project timestamp,  level, correlation_request_id, response_status_code, client_request_id, duration
```

**Expected:** A 202 response. If you see 409, the cluster was already deleting (reissued delete). If you see 204, the cluster didn't exist at that point. Save the `correlation_id` and `client_request_id` for downstream tracing.

### Step 2: Find the Async Operation ID

```kql
// Resolve the async operation path from the delete request
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').frontendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.correlation_request_id =~ '{CORRELATION_ID}'
| where log.msg == 'response complete'
| where log.method == 'DELETE'
| extend asyncOpPath = extract("(hcpOperationStatuses/[a-f0-9-]+)", 1, tostring(log.path))
| project timestamp, asyncOpPath, client_request_id = tostring(log.client_request_id)
| where isnotempty(asyncOpPath)
```

### Step 3: Check Async Operation Polling Status

```kql
// Check if the operation ever reached a terminal state
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').frontendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.path has '{ASYNC_OP_ID}'
| where log.msg == 'response complete'
| summarize
    first_poll = min(timestamp),
    last_poll = max(timestamp),
    total_polls = count()
  by
    method = tostring(log.method),
    status_code = tostring(log.response_status_code)
| order by first_poll asc
```

**Expected (stuck):** Only 200 responses for the entire window — meaning the operation stayed `InProgress`/`Deleting` and never resolved. If 200s stop and the path returns 404, the operation expired (TTL is 7 days).

### Step 4: Check the Backend Operation Controller

The `OperationClusterDelete` controller polls Cluster Service and updates the operation status. Check its conditions for errors.

```kql
// OperationClusterDelete controller condition snapshot
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.resource_group == '{RESOURCE_GROUP}'
| where log.resource_name == '{CLUSTER_NAME}'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpopenshiftclusters/hcpopenshiftcontrollers'
| where log.content.resourceID has '{CLUSTER_NAME}'
| summarize content = take_any(log.content), observedTime = take_any(timestamp) by etag = tostring(log.content._etag)
| sort by tolong(content._ts) asc
| extend content = parse_json(content)
| extend controller_name = extract("/hcpOpenShiftControllers/([^\\/]+)", 1, tostring(content.resourceID))
| where controller_name == 'OperationClusterDelete'
| mv-expand condition = content.properties.status.conditions
| project
    observedTime,
    lastTransitionTime = todatetime(condition.lastTransitionTime),
    controller_name,
    type = tostring(condition.type),
    status = tostring(condition.status),
    reason = tostring(condition.reason),
    message = tostring(condition.message)
| summarize observedTime = min(observedTime) by lastTransitionTime, controller_name, type, status, reason, message
| order by lastTransitionTime asc
```

**Look for:** `Degraded=True` conditions — these indicate errors in the deletion controller.

### Step 5: Check Cluster Service Phase Transitions

This is the most critical query. If CS is stuck in `uninstalling`, the problem is downstream (Maestro/HyperShift).

```kql
// CS phase transitions for the cluster
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.aro_hcp_cluster_resource_id =~ '{RESOURCE_ID}'
| where isempty(log.aro_hcp_node_pool_resource_id)
| where log has 'state to' or log has 'now in'
| project timestamp, msg = tostring(log.msg)
| order by timestamp asc
```

**Expected (stuck):** Last transition is `uninstalling` with no subsequent state change. This means CS issued the delete to Maestro but the ManifestWork on the management cluster hasn't been cleaned up.

### Step 6: Check What CS Is Doing During Uninstall

```kql
// CS deletion loop activity — look for repeated destruct chain messages
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.aro_hcp_cluster_resource_id =~ '{RESOURCE_ID}'
| where isempty(log.aro_hcp_node_pool_resource_id)
| summarize
    first_occurrence = min(timestamp),
    last_occurrence = max(timestamp),
    occurrences = count()
  by msg = tostring(log.msg)
| order by first_occurrence asc
| where occurrences > 5
```

**Look for:**

- `"Running chain deletion to clean deleted cluster"` in a tight loop → CS is retrying cleanup
- `"Not continuing to the next destructor"` → A destructor step is blocking
- `"requested manifest work ... deletion"` repeated → ManifestWork won't disappear
- `"Skipping ManifestWork ... as it contains namespaces"` → Orphaned namespace bundles (Scenario 3 in cleanup SOP)

### Step 7: Resolve the CS Internal Cluster ID (cid)

You'll need the CS `cid` for downstream Maestro queries.

```kql
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.aro_hcp_cluster_resource_id =~ '{RESOURCE_ID}'
| where tostring(cid) != ''
| distinct cid
```

### Step 8: Check Maestro Transition Matrix

This is the **single most important diagnostic** for delivery failures between service and management clusters. It produces per-bundle rows tracking 7 layers.

```kql
// Maestro 7-layer transition matrix
// First, build the bundle map from CS logs
let bundleMapCS = cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where cid == '{CID}'
| where log.msg has 'successfully sent create request of maestro bundle' and log.msg has 'containing resource'
| extend bundleId = extract("bundle id '([^']+)'", 1, tostring(log.msg))
| extend bundleName = extract("maestro bundle name '([^']+)'", 1, tostring(log.msg))
| extend resource = extract("containing resource '([^']+)'", 1, tostring(log.msg))
| extend groupVersion = extract("with GVK '([^,]+),", 1, tostring(log.msg))
| extend resourceKind = extract("Kind=([^']+)", 1, tostring(log.msg))
| where isnotempty(bundleId) and isnotempty(resource)
| extend resourceNamespace = extract("^([^/]+)/", 1, resource)
| extend resourceName = extract("/(.+)$", 1, resource)
| distinct bundleId, bundleName, resource, groupVersion, resourceKind, resourceNamespace, resourceName;
let bundleMap = bundleMapCS
| project bundleId, groupVersion, resourceKind, resourceNamespace, resourceName,
          source = 'clustersService',
          label = strcat(groupVersion, '/', resourceKind, ' ', resource);
let bundleIds = bundleMap | project bundleId;
let resourceNameToBundleId = bundleMap
| where isnotempty(resourceName)
| distinct resourceName, bundleId1 = bundleId;
// Now query Maestro transitions
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').containerLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == 'maestro'
| where container_name in ('maestro-server', 'maestro-agent')
| extend logs = tostring(log)
| extend bundleId = extract('("resourceid":|resourceID=)"([^"]+)"', 2, logs)
| extend resourceName = extract('resourceName="([^"]+)"', 1, logs)
| lookup resourceNameToBundleId on resourceName
| extend bundleId = coalesce(bundleId, bundleId1)
| where isnotempty(bundleId)
| where bundleId in (bundleIds)
| extend msg = extract('] "([^"]+)"', 1, logs)
| extend layer = case(
    container_name == 'maestro-server' and msg == 'receive the event from client',        '1_server_spec_from_client',
    container_name == 'maestro-server' and msg == 'Sending event',                        '2_server_spec_to_broker',
    container_name == 'maestro-agent'  and msg == 'Received event',                       '3_agent_spec_from_broker',
    container_name == 'maestro-agent'  and msg == 'Server side applied',                  '4_agent_acted_on_cluster',
    container_name == 'maestro-agent'  and msg == 'Noop because its read-only',           '4_agent_acted_on_cluster',
    container_name == 'maestro-agent'  and msg == 'Sending event',                        '5_agent_status_to_broker',
    container_name == 'maestro-server' and msg == 'Updating resource status',             '6_server_status_from_broker',
    container_name == 'maestro-server' and msg == 'send the event to status subscribers', '7_server_status_to_subscribers',
    'other')
| where layer != 'other'
| lookup bundleMap on bundleId
| summarize count = count() by source, label, layer
| evaluate pivot(layer, sum(count))
| order by source asc, label asc
```

**Healthy invariants:**

- Layers 1, 2, 3 should have equal counts (spec delivery)
- Layer 4 should be >= spec count (agent may re-apply)
- Layers 5, 6, 7 should be roughly equal and >= spec count (status feedback)
- A **break between adjacent layers** immediately localizes the failure

**For deletion:** The key signal is whether the delete event reaches the agent (layer 3→4) and whether the agent can act on it (layer 4 count). If layer 4 is zero or stuck, the ManifestWork deletion isn't reaching the management cluster.

### Step 9: Check HyperShift Operator Logs

If Maestro successfully delivered the delete, check why HyperShift can't complete it.

```kql
// HyperShift operator log patterns for the affected HostedCluster
// Namespace format: ocm-{env_prefix}-{cid}
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').containerLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == 'hypershift'
| where container_name == 'operator'
| where log.controllerKind == 'HostedCluster'
    and log.HostedCluster.namespace == 'ocm-{ENV_PREFIX}-{CID}'
| summarize
    first_occurrence = min(timestamp),
    last_occurrence = max(timestamp),
    occurrences = count()
  by msg = tostring(log.msg), err = tostring(log.error)
| order by first_occurrence asc
```

**Common stuck patterns:**

- `"Waiting for namespace deletion"` (tight loop) → CP namespace stuck terminating; see [Scenario 4/5 in cleanup SOP](cleanup-stuck-cluster-deletion.md#scenario-4-cp-namespace-stuck-terminating)
- `"hostedcluster is still deleting"` (tight loop) → Finalizer cleanup blocked; check CAPI resources
- `ResourceGroupNotFound` errors → Azure managed RG was deleted externally; safe to force-remove HyperShift finalizer

### Step 10: Check HostedCluster Conditions

```kql
// HostedCluster conditions from the ReadDesire mirror in Cosmos
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.resource_group == '{RESOURCE_GROUP}'
| where log.resource_name == '{CLUSTER_NAME}'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpopenshiftclusters/readdesires'
| summarize content = take_any(log.content), observedTime = take_any(timestamp) by etag = tostring(log.content._etag)
| top 1 by tolong(content._ts) desc
| extend content = parse_json(content)
| extend manifest = content.properties.status.kubeContent
| where manifest.kind == 'HostedCluster'
| mv-expand condition = manifest.status.conditions
| project
    observedTime,
    type = tostring(condition.type),
    status = tostring(condition.status),
    reason = tostring(condition.reason),
    message = tostring(condition.message),
    lastTransitionTime = todatetime(condition.lastTransitionTime)
```

**Look for:** `CloudResourcesDestroyed=True` (Azure resources are gone, safe to force-finalize), error conditions on `InfrastructureReady` or `Available`.

### Step 11: Check CAPZ Logs for Azure Resource Cleanup Issues

If HyperShift is stuck on infrastructure cleanup (VMs, disks, NICs in the managed RG):

```kql
// CAPI Provider Azure (CAPZ) logs — Azure resource deletion errors
// {HCP_NS} = ocm-{env_prefix}-{cid}-{cluster_name}
cluster('{KUSTO_CLUSTER_URI}').database('HostedControlPlaneLogs').containerLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == '{HCP_NS}'
| where pod_name has 'capi-provider-'
| extend msg = extract(@']\s+"([^"]+)"', 1, tostring(log))
| extend err = extract(@'err="((?:[^"\\]|\\.)*)"', 1, tostring(log))
| extend controller = extract(@'controller="([^"]*)"', 1, tostring(log))
| extend controllerKind = extract(@'controllerKind="([^"]*)"', 1, tostring(log))
| extend name = extract(@'\sname="([^"]*)"', 1, tostring(log))
| where controller contains 'azure'
| project timestamp, msg, err, controller, controllerKind, name
| summarize
    first_occurrence = min(timestamp),
    last_occurrence = max(timestamp),
    occurrences = count()
  by msg, err, controller, controllerKind, name
| order by first_occurrence asc
```

**Look for:** Azure API errors (auth failures, throttling, resource locks) preventing VM/NIC/disk deletion.

### Remediation Decision Tree

```
Is CS stuck in "uninstalling"?
├── YES → Is Maestro transition matrix broken at layer 3-4?
│   ├── YES → Maestro delivery issue. Check Maestro agent logs and MQTT connectivity.
│   └── NO → ManifestWork exists but HostedCluster can't delete. Check HyperShift operator logs.
│       ├── "Waiting for namespace deletion" → Finalizers stuck. See cleanup SOP Strategy 3.
│       ├── ResourceGroupNotFound → Managed RG externally deleted. Remove HyperShift finalizer.
│       └── CAPZ Azure errors → Fix Azure-side issue (locks, permissions), or force-remove AzureMachine finalizers.
├── NO, CS shows no record → CS already cleaned up. Check for orphaned Maestro bundles (Strategy 2 in cleanup SOP).
└── NO, CS shows ready → Delete never propagated to CS. Re-issue delete via CS API (Strategy 1 in cleanup SOP).
```

**For manual remediation steps**, see [Cleanup Procedure for Stuck Cluster Deletion](cleanup-stuck-cluster-deletion.md).

---

## 2. Delete Operation Never Completes, Still Listed

**Symptom:** The async ARM operation stays `InProgress` indefinitely. The cluster continues to appear in `az resource list` / Portal after deletion was issued. GET returns the cluster with `provisioningState: Deleting`.

This is closely related to Scenario 1 (stuck deletion) but focuses on the ARM operation tracking layer.

### Step 1: Confirm Operation Document Exists and Its Status

```kql
// Check the ARM resource document state over time
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.content.resourceID =~ '{RESOURCE_ID}'
| summarize content = take_any(log.content) by etag = tostring(log.content._etag)
| sort by tolong(content._ts) asc
| project content
```

**Look for:** The cluster document with `provisioningState: Deleting`, `deletionTimestamp`, and `activeOperationID`. The `activeOperationID` links to the operation document.

### Step 2: Trace the Operation Document State

```kql
// Find the delete operation and its status over time
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.resource_group == '{RESOURCE_GROUP}'
| where log.resource_name == '{CLUSTER_NAME}'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpoperationsstatus'
| summarize content = take_any(log.content), observedTime = take_any(timestamp) by etag = tostring(log.content._etag)
| sort by tolong(content._ts) asc
| extend content = parse_json(content)
| project
    observedTime,
    operationId = tostring(content.resourceID),
    status = tostring(content.properties.status),
    request = tostring(content.properties.request),
    error = content.properties.error,
    lastUpdated = todatetime(content.properties.lastUpdatedTime)
| where request == 'Delete'
```

**Expected (stuck):** Status stays `Deleting` for the entire window. If the operation document has expired (TTL=7 days) and is gone, ARM will stop polling, but the cluster resource doc may still exist in Cosmos.

### Step 3: Check Why the Cluster Document Was Never Deleted

The cluster disappears from ARM listings only when its Cosmos document is deleted. In the legacy path, `SetDeleteOperationAsCompleted` deletes the Cosmos cluster doc when CS returns 404 **and** all children (node pools, external auths) are gone.

```kql
// Check if there are still child resources (node pools, external auths) blocking cluster doc deletion
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.content.resourceID has '{RESOURCE_ID}'
| where log.content.resourceType has 'nodepools' or log.content.resourceType has 'externalauth'
| summarize content = take_any(log.content), lastSeen = max(timestamp) by etag = tostring(log.content._etag)
| extend content = parse_json(content)
| project
    lastSeen,
    resourceType = tostring(content.resourceType),
    resourceID = tostring(content.resourceID),
    provisioningState = tostring(content.properties.provisioningState),
    deletionTimestamp = content.properties.deletionTimestamp
| order by lastSeen desc
```

**If children still exist with `provisioningState: Deleting`**, the legacy path waits for them. Trace each child through CS separately.

### Step 4: Check for Stale activeOperationID

A mismatch between the resource's `activeOperationID` and the current operation can prevent status updates.

```kql
// Check the internal service-provider state for the cluster
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.resource_group == '{RESOURCE_GROUP}'
| where log.resource_name == '{CLUSTER_NAME}'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpopenshiftclusters/serviceproviders'
| where log.content.resourceID has '{CLUSTER_NAME}'
| summarize content = take_any(log.content) by etag = tostring(log.content._etag)
| sort by tolong(content._ts) asc
| extend content = parse_json(content)
| project
    clusterServiceID = tostring(content.properties.clusterServiceID),
    activeOperationID = tostring(content.properties.activeOperationID),
    provisioningState = tostring(content.properties.provisioningState),
    deletionTimestamp = content.properties.deletionTimestamp,
    clusterServiceDeletionTimestamp = content.properties.clusterServiceDeletionTimestamp
```

**Look for:**

- `clusterServiceID` still present → CS cluster hasn't been confirmed deleted yet
- `deletionTimestamp` is set but `clusterServiceDeletionTimestamp` is not → CS delete was never dispatched (new-path controller issue)
- `activeOperationID` empty with `provisioningState: Deleting` → Operation link is broken

### Step 5: Verify Operation Notification

If the operation completed but ARM never got notified (missing async notification POST):

```kql
// Look for backend notification attempts
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log has '{ASYNC_OP_ID}'
| where log.msg has 'notification' or log.msg has 'notify'
| project timestamp, msg = tostring(log.msg), level = tostring(log.level)
| order by timestamp asc
```

### Remediation

If the cluster doc is genuinely orphaned in Cosmos (CS is 404, children are gone, but the Cosmos cluster document was never deleted due to a transient error):

1. Verify CS has no record of the cluster (Phase 2, Step 5 in [cleanup SOP](cleanup-stuck-cluster-deletion.md))
2. Verify Maestro has no remaining bundles (Phase 2, Step 6)
3. Verify management cluster has no remaining resources (Phase 1)
4. If all layers are clean, the issue is a Cosmos document leak — escalate to RP team for manual Cosmos cleanup

---

## 3. Delete Operation Fails

**Symptom:** The ARM operation ends in `provisioningState: Failed` with an error. The cluster resource may still exist.

### Step 1: Find the Error from the Operation

```kql
// Get the terminal operation status with error details
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').frontendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.path has '{ASYNC_OP_ID}'
| where log.msg == 'response complete'
| where log.response_status_code == '200'
| top 1 by timestamp desc
| project timestamp, path = tostring(log.path), status = tostring(log.response_status_code)
```

Then check the backend for the error that caused the failure:

```kql
// Backend logs with cloud error codes for the cluster
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where resource_group == '{RESOURCE_GROUP}'
| where resource_name == '{CLUSTER_NAME}'
| where isnotempty(cloud_error_code) or isnotempty(cloud_error_message)
| project
    timestamp,
    operation,
    operation_id,
    cloud_error_code,
    cloud_error_message,
    level,
    msg
| order by timestamp asc
```

### Step 2: Check for CS Error State

If CS moved the cluster to `error` state instead of completing `uninstalling`:

```kql
// CS phase transitions — look for 'error' state
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.aro_hcp_cluster_resource_id =~ '{RESOURCE_ID}'
| where isempty(log.aro_hcp_node_pool_resource_id)
| where log has 'state to' or log has 'now in' or log has 'error'
| project timestamp, msg = tostring(log.msg), level = tostring(log.level)
| order by timestamp asc
```

### Step 3: Check for Inflight Check Failures

The backend performs inflight checks before CS operations. `OCM4001` errors indicate CS rejected the operation.

```kql
// Inflight check errors
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where resource_group == '{RESOURCE_GROUP}'
| where resource_name == '{CLUSTER_NAME}'
| where log has 'inflight' or log has 'OCM4001'
| project timestamp, msg = tostring(log.msg), level = tostring(log.level)
| order by timestamp asc
```

### Step 4: Check Controller Condition Timeline for Errors

```kql
// Full condition transition timeline — look for Degraded=True
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.resource_group == '{RESOURCE_GROUP}'
| where log.resource_name == '{CLUSTER_NAME}'
| where log.content.resourceType =~ 'microsoft.redhatopenshift/hcpopenshiftclusters/hcpopenshiftcontrollers'
| where log.content.resourceID has '{CLUSTER_NAME}'
| summarize content = take_any(log.content), observedTime = take_any(timestamp) by etag = tostring(log.content._etag)
| sort by tolong(content._ts) asc
| extend content = parse_json(content)
| extend controller_name = extract("/hcpOpenShiftControllers/([^\\/]+)", 1, tostring(content.resourceID))
| mv-expand condition = content.properties.status.conditions
| project
    observedTime,
    lastTransitionTime = todatetime(condition.lastTransitionTime),
    controller_name,
    type = tostring(condition.type),
    status = tostring(condition.status),
    reason = tostring(condition.reason),
    message = tostring(condition.message)
| where status == 'True' and type == 'Degraded'
| summarize observedTime = min(observedTime) by lastTransitionTime, controller_name, type, status, reason, message
| order by lastTransitionTime asc
```

### Step 5: Check for Frontend-Side Errors

If the error occurred during the initial DELETE request processing (before async handoff):

```kql
// Frontend error logs for the delete request
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').frontendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.resource_group =~ '{RESOURCE_GROUP}'
| where log.resource_name =~ '{CLUSTER_NAME}'
| where log.method == 'DELETE'
| where toint(log.response_status_code) >= 400
| project
    timestamp,
    status = tostring(log.response_status_code),
    error = tostring(log.error),
    path = tostring(log.path),
    correlation_id = tostring(log.correlation_request_id)
| order by timestamp asc
```

**Common error patterns:**


| Error Code              | Meaning                               | Action                                                           |
| ----------------------- | ------------------------------------- | ---------------------------------------------------------------- |
| 409 Conflict            | Cluster already deleting              | Safe to ignore — check the existing delete operation             |
| 500 InternalServerError | Cosmos or CS connectivity failure     | Retry the delete                                                 |
| CS `error` state        | CS encountered an unrecoverable issue | Check CS logs for root cause, may need manual CS/Maestro cleanup |
| `OCM4001`               | CS inflight check failure             | Wait and retry, or check CS for the blocking operation           |


### Remediation

- If the operation failed but the cluster resource still exists, **re-issue the DELETE**. The frontend accepts new delete requests after a previous one has failed.
- If CS is in an `error` state, use `hcpctl sc breakglass` and re-issue the delete via CS API directly (see [cleanup SOP Strategy 1](cleanup-stuck-cluster-deletion.md#strategy-1-delete-via-clusters-service-api-preferred)).

---

## 4. Orphaned Azure Resources

**Symptom:** After cluster deletion completes (or stalls), VMs, NICs, disks, load balancers, or managed identities remain in the managed resource group.

### Understanding Resource Ownership


| Resource                         | Created By             | Managed RG | Customer RG |
| -------------------------------- | ---------------------- | ---------- | ----------- |
| VMs (via VMSS)                   | HyperShift/CAPI/CAPZ   | Yes        | No          |
| NICs                             | HyperShift/CAPI/CAPZ   | Yes        | No          |
| Disks (OS, etcd PVCs)            | HyperShift/CAPI/CAPZ   | Yes        | No          |
| Load Balancer                    | Cluster Service        | Yes        | No          |
| DNS Zone                         | Cluster Service        | Yes        | No          |
| User-Assigned Managed Identities | Customer (pre-created) | No         | Yes         |
| VNet, Subnet, NSG                | Customer (pre-created) | No         | Yes         |


The managed resource group name follows the pattern `arohcp-{clusterName}-{uuid}` (or customer-specified). It has `managedBy` pointing at the HCP cluster ARM ID.

### Step 1: Identify the Managed Resource Group

```kql
// Find the managed resource group name from the cluster creation/config
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where container_name == 'aro-hcp-backend'
| where log.controller_name == 'datadump'
| where log.content.resourceID =~ '{RESOURCE_ID}'
| summarize content = take_any(log.content) by etag = tostring(log.content._etag)
| top 1 by tolong(content._ts) desc
| extend content = parse_json(content)
| project
    managedResourceGroup = tostring(content.properties.platform.managedResourceGroup),
    subnetId = tostring(content.properties.platform.subnetId),
    nsgId = tostring(content.properties.platform.networkSecurityGroupId)
```

### Step 2: Check CAPZ Logs for Azure Resource Deletion Errors

CAPZ (Cluster API Provider Azure) is responsible for deleting VMs and associated resources. If it fails, resources are orphaned.

```kql
// CAPZ errors during AzureMachine deletion
// {HCP_NS} = ocm-{env_prefix}-{cid}-{cluster_name}  (control plane namespace)
cluster('{KUSTO_CLUSTER_URI}').database('HostedControlPlaneLogs').containerLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where namespace_name == '{HCP_NS}'
| where pod_name has 'capi-provider-'
| extend msg = extract(@']\s+"([^"]+)"', 1, tostring(log))
| extend err = extract(@'err="((?:[^"\\]|\\.)*)"', 1, tostring(log))
| extend controller = extract(@'controller="([^"]*)"', 1, tostring(log))
| extend name = extract(@'\sname="([^"]*)"', 1, tostring(log))
| where controller contains 'azure'
| where isnotempty(err)
| project timestamp, msg, err, controller, name
| summarize
    first_occurrence = min(timestamp),
    last_occurrence = max(timestamp),
    occurrences = count()
  by msg, err, controller, name
| order by occurrences desc
```

**Look for:**

- `AuthorizationFailed` → Managed identity lost permissions to delete resources
- `ResourceGroupNotFound` → Managed RG was deleted externally but CAPZ still references it
- `OperationNotAllowed` → Resource locks or deny assignments prevent deletion
- `Throttling` / `TooManyRequests` → Azure API rate limits; typically resolves on retry

### Step 3: Check Kubernetes Events for Resource Deletion Failures

```kql
// K8s events in the control plane namespace during deletion
cluster('{KUSTO_CLUSTER_URI}').database('HostedControlPlaneLogs').kubernetesEvents
| where timestamp between ({START_TIME} .. {END_TIME})
| where eventNamespace == '{HCP_NS}'
| where kubeEventType == 'Warning'
| project
    timestamp,
    objectKind,
    objectName,
    reason,
    message,
    sourceComponent,
    firstSeen,
    lastSeen,
    count
| order by timestamp desc
```

### Step 4: Check HyperShift for CloudResourcesDestroyed Condition

If the HostedCluster was deleted but cloud resources weren't cleaned up:

```kql
// HostedCluster condition timeline — focus on CloudResourcesDestroyed
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').backendLogs
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
| project
    observedTime,
    type = tostring(condition.type),
    status = tostring(condition.status),
    reason = tostring(condition.reason),
    message = tostring(condition.message),
    lastTransitionTime = todatetime(condition.lastTransitionTime)
| order by lastTransitionTime asc
```

**If `CloudResourcesDestroyed=False`**, HyperShift was unable to clean up Azure resources. The CAPZ logs (Step 2) will show why.

### Step 5: Check for Orphaned DNS Resources

```kql
// CS DNS cleanup activity
cluster('{KUSTO_CLUSTER_URI}').database('ServiceLogs').clustersServiceLogs
| where timestamp between ({START_TIME} .. {END_TIME})
| where log.aro_hcp_cluster_resource_id =~ '{RESOURCE_ID}'
| where log has 'dns' or log has 'DNS' or log has 'zone'
| project timestamp, msg = tostring(log.msg), level = tostring(log.level)
| order by timestamp asc
```

### Step 6: Identify Specific Orphaned Resources via Azure Activity Logs

If you have access to the customer's subscription activity logs:

```kql
// Check Azure Resource Graph for resources in the managed RG
// (Run this in Azure Resource Graph Explorer, not Kusto)
Resources
| where resourceGroup =~ '{MANAGED_RESOURCE_GROUP}'
| where subscriptionId =~ '{SUBSCRIPTION_ID}'
| project name, type, location, tags, properties
```

### Common Orphan Scenarios and Fixes


| Orphan Type                             | Likely Cause                                           | Fix                                                                             |
| --------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------- |
| VMs still running                       | CAPZ couldn't delete AzureMachine (auth/lock)          | Remove resource locks, fix RBAC, then re-trigger delete or manually delete VMs  |
| NICs remaining                          | VMs were deleted but NICs had external references      | Manually delete NICs via Azure CLI                                              |
| Disks (OS/data)                         | CAPZ deletion failed mid-way                           | Manually delete orphaned disks                                                  |
| Load Balancer                           | CS cleanup didn't run or failed                        | Manually delete LB; check CS destruct chain logs                                |
| Managed RG exists but is empty          | Deletion completed but RG itself wasn't cleaned up     | Delete the RG manually: `az group delete -n {MRG}`                              |
| Managed RG exists with deny assignments | Public cloud deny assignments prevent direct deletion  | Must delete through RP → CS → HyperShift chain; or escalate for FPA cleanup     |
| DNS zone / records                      | CS DNS destructor failed                               | Manually delete DNS zone from managed RG, remove delegations from regional zone |
| User-Assigned Managed Identities        | These live in the **customer RG** and were pre-created | These are **not cleaned up by ARO-HCP by design** — the customer manages them   |


### Azure CLI Commands for Orphan Cleanup

```bash
# List all resources in the managed resource group
az resource list --resource-group "{MANAGED_RESOURCE_GROUP}" -o table

# Delete specific resource types
az vm delete --ids $(az vm list -g "{MANAGED_RESOURCE_GROUP}" --query "[].id" -o tsv) --yes
az network nic delete --ids $(az network nic list -g "{MANAGED_RESOURCE_GROUP}" --query "[].id" -o tsv)
az disk delete --ids $(az disk list -g "{MANAGED_RESOURCE_GROUP}" --query "[].id" -o tsv) --yes
az network lb delete --ids $(az network lb list -g "{MANAGED_RESOURCE_GROUP}" --query "[].id" -o tsv)

# Delete the entire managed resource group (if all resources can be removed)
az group delete -n "{MANAGED_RESOURCE_GROUP}" --yes --no-wait
```

> **Warning:** In public cloud, managed RGs may have deny assignments. Use `--force-deletion-types` if needed:
>
> ```bash
> az group delete -n "{MANAGED_RESOURCE_GROUP}" --yes --force-deletion-types Microsoft.Compute/virtualMachines Microsoft.Compute/virtualMachineScaleSets
> ```

---

## Appendix A: Quick-Reference Kusto Cluster URIs


| Environment      | Kusto Cluster                                               | Notes                           |
| ---------------- | ----------------------------------------------------------- | ------------------------------- |
| INT (UK)         | `https://hcp-int-uk.uksouth.kusto.windows.net`              | Integration                     |
| PROD (US Canary) | `https://hcp-prod-usc.{region}.kusto.windows.net`           | Matches `*-eastus2euap` Grafana |
| General pattern  | `https://hcp-{env}-{geoShortId}.{region}.kusto.windows.net` | See `docs/logging.md`           |


## Appendix B: Key Table Reference


| Table                 | Database    | Key Columns for Deletion                                                                                    |
| --------------------- | ----------- | ----------------------------------------------------------------------------------------------------------- |
| `frontendLogs`        | ServiceLogs | `log.method`, `log.response_status_code`, `log.path`, `log.correlation_request_id`, `log.client_request_id` |
| `backendLogs`         | ServiceLogs | `log.controller_name`, `log.content` (Cosmos doc dumps), `cloud_error_code`, `operation`, `operation_id`    |
| `clustersServiceLogs` | ServiceLogs | `log.aro_hcp_cluster_resource_id`, `cid`, `log.msg` (phase transitions)                                     |
| `containerLogs`       | Both        | `namespace_name`, `container_name`, `log` (structured JSON for Maestro/HyperShift)                          |
| `kubernetesEvents`    | Both        | `eventNamespace`, `objectKind`, `objectName`, `reason`, `message`                                           |


## Appendix C: Namespace Naming Conventions


| Namespace               | Purpose                     | Pattern                                 |
| ----------------------- | --------------------------- | --------------------------------------- |
| HostedCluster namespace | HostedCluster CR + secrets  | `ocm-{env_prefix}-{cid}`                |
| Control Plane namespace | CP pods, CAPI objects, etcd | `ocm-{env_prefix}-{cid}-{cluster_name}` |
| Maestro (SVC)           | Maestro server              | `maestro`                               |
| HyperShift (MGMT)       | HyperShift operator         | `hypershift`                            |
| ManifestWork (MGMT)     | ACM work resources          | `local-cluster`                         |


Environment prefixes: `arohcpint`, `arohcpprod`, `arohcpstg`.

## Appendix D: Related SOPs

- [Cleanup Procedure for Stuck Cluster Deletion](cleanup-stuck-cluster-deletion.md) — Manual break-glass remediation
- [HCP Cluster Creation Flow](hcp-cluster-creation-flow.md) — Understanding the creation path (deletion is the inverse)
- [Fix Maestro Stale Resource Bundle](fix-maestro-stale-resource-bundle.md) — Maestro-specific cleanup
- [Kusto Query Cookbook](../ai/query-cookbook.md) — Full index of canned queries
- [Kusto Debugging Reference](../ai/kusto-debugging.md) — Table schemas and common patterns

