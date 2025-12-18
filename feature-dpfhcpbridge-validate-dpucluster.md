# Feature Technical Specification: DPFHCPBridge DPUCluster Validation

## Metadata
- **Feature**: DPFHCPBridge DPUCluster Validation
- **Operator**: DPF-HCP Bridge Operator
- **Version**: v1alpha1
- **Created**: 2025-12-21
- **Last Updated**: 2025-12-21 (Operator-SDK Alignment)
- **Status**: Ready for Implementation

## Overview

### Purpose
This feature validates that the DPUCluster referenced in a DPFHCPBridge CR exists in the cluster before proceeding with any HostedCluster lifecycle management. It ensures that DPFHCPBridge only operates on valid DPUCluster references and provides clear feedback to users when references are invalid.

### Scope
**In Scope:**
- Validating DPUCluster existence at reconciliation time (supports cross-namespace references)
- Setting appropriate status conditions based on validation results
- Blocking HostedCluster lifecycle operations when DPUCluster is invalid
- Handling DPUCluster deletion scenarios (missing DPUCluster)
- Periodic revalidation on reconciliation loops
- Efficient watch-based reconciliation triggering when DPUCluster changes

**Out of Scope:**
- Creating or modifying DPUCluster resources
- Validating DPUCluster internal state or readiness (only existence check)
- Admission webhook validation (validation happens during reconciliation only)

### Trigger
This feature is triggered during **every reconciliation** of a DPFHCPBridge CR, and when watched DPUCluster resources change.

### Prerequisites
- DPFHCPBridge CR exists in the cluster
- DPUCluster CRD is installed in the cluster
- Operator has RBAC permissions to GET DPUCluster resources across namespaces (ClusterRole)

## CR Fields Used/Modified by This Feature

### Target CR
- **Kind:** `DPFHCPBridge`
- **API:** `dpf.hcp.bridge.com/v1alpha1`

### Spec Fields Read
This feature reads the following spec fields from the DPFHCPBridge CR:

| Field | Type | Purpose in This Feature |
|-------|------|-------------------------|
| `dpuClusterRef.name` | string | The name of the DPUCluster to validate for existence |
| `dpuClusterRef.namespace` | string | The namespace of the DPUCluster (supports cross-namespace) |

### Status Fields Modified by This Feature

This feature updates the following existing status fields:

| Field | Type | How Modified | When |
|-------|------|--------------|------|
| `conditions` | []metav1.Condition | Adds/updates `DPUClusterValid` condition | Every reconciliation |

### Conditions Added/Modified by This Feature

#### NEW Condition: `DPUClusterValid`
- **Type:** `DPUClusterValid`
- **Purpose:** Indicates whether the referenced DPUCluster exists and is accessible
- **Set to True:** When the DPUCluster referenced by `spec.dpuClusterRef` (name + namespace) exists in the cluster
- **Set to False:** When the DPUCluster does not exist or cannot be accessed
- **Reason Codes:**
  - `DPUClusterFound`: The referenced DPUCluster exists in the cluster
  - `DPUClusterNotFound`: The referenced DPUCluster does not exist in the specified namespace
  - `DPUClusterAccessError`: Error occurred while attempting to retrieve the DPUCluster (permission denied, API error, etc.)
- **Example Messages:**
  - True: "DPUCluster 'dpucluster-prod-01' found in namespace 'dpf-operator-system'"
  - False (NotFound): "DPUCluster 'dpucluster-prod-01' not found in namespace 'dpf-operator-system'"
  - False (AccessError): "Failed to retrieve DPUCluster 'dpucluster-prod-01' in namespace 'dpf-operator-system': permission denied"

## Reconciliation Logic

### State Detection

**Desired State Source:**
The DPFHCPBridge spec fields `dpuClusterRef.name` and `dpuClusterRef.namespace` contain the NamespacedName of the DPUCluster that should exist.

**Actual State Source:**
Query the Kubernetes API server for the DPUCluster resource with the NamespacedName specified in `spec.dpuClusterRef`.

### Reconciliation Flow

```
1. Extract DPUCluster NamespacedName from DPFHCPBridge spec
   ↓
2. Attempt to GET DPUCluster from API server (cross-namespace)
   ↓
3. Evaluate result and set DPUClusterValid condition
   ↓
4. Determine if reconciliation should proceed to next responsibility
```

### Detailed Steps

#### Step 1: Extract DPUCluster NamespacedName
**Action:** Read `spec.dpuClusterRef.name` and `spec.dpuClusterRef.namespace` from the DPFHCPBridge CR
**Idempotency:** Read operation, inherently idempotent
**Error Handling:** If fields are empty/missing, set condition to False with reason `DPUClusterNotSpecified`
**Requeue:** No requeue needed for this step

```go
// Pseudocode outline
func extractDPUClusterRef(dpfhcpBridge *DPFHCPBridge) (types.NamespacedName, error) {
    if dpfhcpBridge.Spec.DPUClusterRef == nil {
        return types.NamespacedName{}, fmt.Errorf("dpuClusterRef not specified")
    }
    if dpfhcpBridge.Spec.DPUClusterRef.Name == "" {
        return types.NamespacedName{}, fmt.Errorf("dpuClusterRef.name not specified")
    }
    if dpfhcpBridge.Spec.DPUClusterRef.Namespace == "" {
        return types.NamespacedName{}, fmt.Errorf("dpuClusterRef.namespace not specified")
    }

    return types.NamespacedName{
        Name:      dpfhcpBridge.Spec.DPUClusterRef.Name,
        Namespace: dpfhcpBridge.Spec.DPUClusterRef.Namespace,
    }, nil
}
```

#### Step 2: Validate DPUCluster Existence
**Action:** Attempt to GET the DPUCluster resource from the API server (cross-namespace)
**Idempotency:** Read operation, inherently idempotent
**Error Handling:**
- NotFound error: DPUCluster does not exist in specified namespace
- Permission/API errors: Access error
**Requeue:** Requeue with delay if access error (transient), no requeue if NotFound (user must fix)

```go
// Pseudocode outline
func validateDPUClusterExists(ctx context.Context, client client.Client, ref types.NamespacedName) (*DPUCluster, error) {
    dpuCluster := &DPUCluster{}
    err := client.Get(ctx, ref, dpuCluster)
    if err != nil {
        if apierrors.IsNotFound(err) {
            return nil, &NotFoundError{Ref: ref}
        }
        return nil, &AccessError{Ref: ref, Cause: err}
    }
    return dpuCluster, nil
}
```

#### Step 3: Set DPUClusterValid Condition
**Action:** Update the `DPUClusterValid` condition based on validation result
**Idempotency:** Condition updates use meta.SetStatusCondition which handles idempotency
**Error Handling:** Status update failures should be logged and cause requeue
**Requeue:** No requeue on success; requeue on status update failure

```go
// Pseudocode outline
func setDPUClusterValidCondition(ctx context.Context, dpfhcpBridge *DPFHCPBridge, valid bool, reason, message string) error {
    condition := metav1.Condition{
        Type:               "DPUClusterValid",
        Status:             metav1.ConditionTrue,
        ObservedGeneration: dpfhcpBridge.Generation,
        LastTransitionTime: metav1.Now(),
        Reason:             reason,
        Message:            message,
    }
    if !valid {
        condition.Status = metav1.ConditionFalse
    }

    meta.SetStatusCondition(&dpfhcpBridge.Status.Conditions, condition)

    // Update status subresource
    return r.Status().Update(ctx, dpfhcpBridge)
}
```

#### Step 4: Determine Reconciliation Continuation
**Action:** Based on DPUClusterValid condition, decide whether to proceed with HostedCluster lifecycle management
**Idempotency:** Decision logic, inherently idempotent
**Error Handling:** N/A (pure logic)
**Requeue:** Depends on validation result (see Requeue Strategy section)

```go
// Pseudocode outline
func shouldProceedToHostedClusterManagement(dpfhcpBridge *DPFHCPBridge) bool {
    condition := meta.FindStatusCondition(dpfhcpBridge.Status.Conditions, "DPUClusterValid")
    return condition != nil && condition.Status == metav1.ConditionTrue
}
```

### Idempotency Guarantees
All operations in this feature are idempotent:
- Reading `spec.dpuClusterRef` is a read operation
- GET DPUCluster is a read operation (supports cross-namespace)
- Setting status conditions uses `meta.SetStatusCondition` which only updates if values change
- Status updates are applied via Kubernetes status subresource (safe for concurrent updates)

### Finalizer Handling

**Finalizer Name:** N/A

This feature does not require finalizer handling. It performs validation only and does not manage external resources that require cleanup.

### Requeue Strategy

| Condition | Requeue | Delay | Reason |
|-----------|---------|-------|--------|
| DPUCluster found | No | N/A | Validation successful, proceed to next responsibility |
| DPUCluster not found | No | N/A | User must create DPUCluster or fix reference |
| Access error (permission denied) | Yes | 30s | Transient RBAC issue may resolve |
| Access error (API server unavailable) | Yes | Exponential (5s → 60s) | Transient API issue may resolve |
| Status update failure | Yes | Exponential (1s → 60s) | Retry status update |

**Note:** When DPUCluster is not found, the reconciliation does NOT requeue. The DPFHCPBridge will be reconciled again when:
- The DPFHCPBridge CR is updated
- The DPUCluster is created (requires watch on DPUCluster with field indexing)

## Status Management

### Status Fields

```go
// DPFHCPBridgeStatus uses standard metav1.Condition
type DPFHCPBridgeStatus struct {
    // Conditions represent the latest available observations of the DPFHCPBridge's state
    // +optional
    // +listType=map
    // +listMapKey=type
    // +patchMergeKey=type
    // +patchStrategy=merge
    Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type"`

    // Other status fields managed by different features...
}
```

### Status Conditions

#### Condition: DPUClusterValid
- **Type:** `DPUClusterValid`
- **Status:** True/False/Unknown
- **Reason:** Machine-readable reason code
- **When True:** The referenced DPUCluster exists and is accessible
- **When False:** The referenced DPUCluster does not exist or cannot be accessed

```go
// Example condition setting for success case
meta.SetStatusCondition(&dpfhcpBridge.Status.Conditions, metav1.Condition{
    Type:               "DPUClusterValid",
    Status:             metav1.ConditionTrue,
    ObservedGeneration: dpfhcpBridge.Generation,
    LastTransitionTime: metav1.Now(),
    Reason:             "DPUClusterFound",
    Message:            fmt.Sprintf("DPUCluster '%s' found in namespace '%s'", dpuClusterRef.Name, dpuClusterRef.Namespace),
})

// Example condition setting for not found case
meta.SetStatusCondition(&dpfhcpBridge.Status.Conditions, metav1.Condition{
    Type:               "DPUClusterValid",
    Status:             metav1.ConditionFalse,
    ObservedGeneration: dpfhcpBridge.Generation,
    LastTransitionTime: metav1.Now(),
    Reason:             "DPUClusterNotFound",
    Message:            fmt.Sprintf("DPUCluster '%s' not found in namespace '%s'", dpuClusterRef.Name, dpuClusterRef.Namespace),
})

// Example condition setting for access error case
meta.SetStatusCondition(&dpfhcpBridge.Status.Conditions, metav1.Condition{
    Type:               "DPUClusterValid",
    Status:             metav1.ConditionFalse,
    ObservedGeneration: dpfhcpBridge.Generation,
    LastTransitionTime: metav1.Now(),
    Reason:             "DPUClusterAccessError",
    Message:            fmt.Sprintf("Failed to retrieve DPUCluster '%s' in namespace '%s': %v", dpuClusterRef.Name, dpuClusterRef.Namespace, err),
})
```

### Events

| Event Type | Reason | Message Template | When |
|------------|--------|------------------|------|
| Warning | DPUClusterNotFound | "Referenced DPUCluster '%s/%s' not found" | DPUCluster does not exist |
| Warning | DPUClusterAccessError | "Failed to access DPUCluster '%s/%s': %v" | Permission or API error |
| Normal | DPUClusterValidated | "DPUCluster '%s/%s' validated successfully" | DPUCluster found |

```go
// Example event emission using controller-runtime event recorder
r.Recorder.Event(dpfhcpBridge, corev1.EventTypeWarning, "DPUClusterNotFound",
    fmt.Sprintf("Referenced DPUCluster '%s/%s' not found", dpuClusterRef.Namespace, dpuClusterRef.Name))
```

## External Integrations

This feature does not integrate with external systems. It only interacts with the Kubernetes API server to validate DPUCluster existence.

## Error Handling

### Error Categories

#### Validation Errors
**Examples:**
- `spec.dpuClusterRef` is nil, or name/namespace fields are empty
- DPUCluster does not exist in specified namespace (NotFound)

**Handling:**
1. Set `DPUClusterValid` condition to False
2. Set appropriate Reason code (`DPUClusterNotSpecified` or `DPUClusterNotFound`)
3. Emit Warning event
4. Do NOT requeue (user must fix the DPFHCPBridge spec or create the DPUCluster)
5. Block progression to HostedCluster lifecycle management

```go
// Example handling
dpuClusterRef, err := extractDPUClusterRef(dpfhcpBridge)
if err != nil {
    meta.SetStatusCondition(&dpfhcpBridge.Status.Conditions, metav1.Condition{
        Type:    "DPUClusterValid",
        Status:  metav1.ConditionFalse,
        Reason:  "DPUClusterNotSpecified",
        Message: fmt.Sprintf("spec.dpuClusterRef is invalid: %v", err),
    })
    r.Recorder.Event(dpfhcpBridge, corev1.EventTypeWarning, "DPUClusterNotSpecified",
        "spec.dpuClusterRef.name and namespace are required")
    return ctrl.Result{}, r.Status().Update(ctx, dpfhcpBridge)
}
```

#### Transient Errors
**Examples:**
- API server temporarily unavailable
- Network timeout while querying DPUCluster
- Temporary RBAC permission issues (during cluster reconfiguration)

**Handling:**
1. Set `DPUClusterValid` condition to False with Reason `DPUClusterAccessError`
2. Log error at ERROR level
3. Emit Warning event
4. Requeue with exponential backoff (5s → 60s)
5. Block progression to HostedCluster lifecycle management

```go
// Example handling
if err != nil && !apierrors.IsNotFound(err) {
    log.Error(err, "Failed to retrieve DPUCluster", "namespace", dpuClusterRef.Namespace, "name", dpuClusterRef.Name)

    meta.SetStatusCondition(&dpfhcpBridge.Status.Conditions, metav1.Condition{
        Type:    "DPUClusterValid",
        Status:  metav1.ConditionFalse,
        Reason:  "DPUClusterAccessError",
        Message: fmt.Sprintf("Failed to retrieve DPUCluster '%s/%s': %v", dpuClusterRef.Namespace, dpuClusterRef.Name, err),
    })
    r.Recorder.Event(dpfhcpBridge, corev1.EventTypeWarning, "DPUClusterAccessError",
        fmt.Sprintf("Failed to access DPUCluster '%s/%s'", dpuClusterRef.Namespace, dpuClusterRef.Name))

    if err := r.Status().Update(ctx, dpfhcpBridge); err != nil {
        return ctrl.Result{}, err
    }

    // Requeue with exponential backoff
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}
```

#### Permanent Errors
**Examples:**
- RBAC permissions permanently missing (misconfigured operator deployment)
- DPUCluster CRD not installed

**Handling:**
1. Set `DPUClusterValid` condition to False with Reason `DPUClusterAccessError`
2. Log error at ERROR level
3. Emit Warning event
4. Requeue with capped exponential backoff (max 5 minutes)
5. Block progression to HostedCluster lifecycle management
6. Administrator intervention required

```go
// Example handling (same as transient, but with different logging context)
if apierrors.IsForbidden(err) {
    log.Error(err, "RBAC permissions missing for DPUCluster access - operator may be misconfigured",
        "namespace", dpuClusterRef.Namespace, "name", dpuClusterRef.Name)
    // Continue with same transient error handling...
}
```

### Edge Cases

#### Edge Case 1: DPUCluster Deleted After DPFHCPBridge Creation
**Scenario:** DPFHCPBridge exists and references a DPUCluster that was previously valid but has been deleted
**Detection:** GET DPUCluster returns NotFound error during reconciliation
**Handling:**
1. Set `DPUClusterValid` condition to False with Reason `DPUClusterNotFound`
2. Emit Warning event
3. Do NOT delete HostedCluster (other features handle that lifecycle)
4. Do NOT requeue (wait for user to recreate DPUCluster or update DPFHCPBridge)

```go
// Pseudocode
if apierrors.IsNotFound(err) {
    meta.SetStatusCondition(&dpfhcpBridge.Status.Conditions, metav1.Condition{
        Type:    "DPUClusterValid",
        Status:  metav1.ConditionFalse,
        Reason:  "DPUClusterNotFound",
        Message: fmt.Sprintf("DPUCluster '%s/%s' not found - it may have been deleted", dpuClusterRef.Namespace, dpuClusterRef.Name),
    })
    r.Recorder.Event(dpfhcpBridge, corev1.EventTypeWarning, "DPUClusterNotFound",
        fmt.Sprintf("Referenced DPUCluster '%s/%s' not found", dpuClusterRef.Namespace, dpuClusterRef.Name))
    return ctrl.Result{}, r.Status().Update(ctx, dpfhcpBridge)
}
```

#### Edge Case 2: DPFHCPBridge Spec Updated to Reference Different DPUCluster
**Scenario:** User updates `spec.dpuClusterRef` to reference a different DPUCluster (different name or namespace)
**Detection:** Normal reconciliation flow detects spec change via Generation
**Handling:**
1. Validate the NEW DPUCluster reference (including new namespace)
2. Update `DPUClusterValid` condition based on new validation result
3. ObservedGeneration in condition reflects the new Generation
4. Other features handle HostedCluster lifecycle changes (out of scope for this feature)

```go
// Pseudocode - already handled by normal flow
// ObservedGeneration ensures condition reflects current spec
condition.ObservedGeneration = dpfhcpBridge.Generation
```

#### Edge Case 3: Concurrent Reconciliations
**Scenario:** Multiple reconciliation loops triggered simultaneously for the same DPFHCPBridge
**Detection:** Kubernetes controller-runtime handles this via workqueue
**Handling:**
1. Controller-runtime ensures only one reconciliation per resource at a time
2. Status updates use optimistic locking (resourceVersion)
3. If status update fails due to conflict, reconciliation will be requeued
4. Validation logic is idempotent, so re-running is safe

```go
// Pseudocode - handled by controller-runtime and status subresource
err := r.Status().Update(ctx, dpfhcpBridge)
if apierrors.IsConflict(err) {
    // Controller-runtime will requeue automatically
    return ctrl.Result{}, err
}
```

#### Edge Case 4: DPUCluster Exists But Has No Status/Is Not Ready
**Scenario:** DPUCluster resource exists but may not be in a ready/operational state
**Detection:** GET DPUCluster succeeds
**Handling:**
1. Set `DPUClusterValid` condition to True (existence validation only)
2. Validation of DPUCluster readiness/status is **out of scope** for this feature
3. Other features/responsibilities may check DPUCluster status as needed

```go
// Pseudocode - simple existence check
dpuCluster := &DPUCluster{}
err := r.Get(ctx, dpuClusterRef, dpuCluster)
if err == nil {
    // DPUCluster exists - validation passes
    // We do NOT check dpuCluster.Status here
    meta.SetStatusCondition(&dpfhcpBridge.Status.Conditions, metav1.Condition{
        Type:    "DPUClusterValid",
        Status:  metav1.ConditionTrue,
        Reason:  "DPUClusterFound",
        Message: fmt.Sprintf("DPUCluster '%s/%s' found", dpuClusterRef.Namespace, dpuClusterRef.Name),
    })
}
```

## RBAC Permissions

### ClusterRole Rules

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dpf-hcp-bridge-operator-dpucluster-validator
rules:
  # DPUCluster validation (cross-namespace access)
  - apiGroups:
      - provisioning.dpu.nvidia.com  # API group only (no version)
    resources:
      - dpuclusters
    verbs:
      - get
      - list
      - watch

  # DPFHCPBridge status updates
  - apiGroups:
      - dpf.hcp.bridge.com
    resources:
      - dpfhcpbridges/status
    verbs:
      - update
      - patch

  # Events
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
```

**IMPORTANT:** The apiGroups field contains ONLY the API group name (e.g., `provisioning.dpu.nvidia.com`), NOT the version. The version is specified in the resource reference, not in RBAC rules.

### Permission Justification

| Resource | Verbs | Justification |
|----------|-------|---------------|
| `dpuclusters` | get | Required to validate that the referenced DPUCluster exists (cross-namespace) |
| `dpuclusters` | list | Required for efficient caching and field indexing |
| `dpuclusters` | watch | Required to trigger reconciliation when DPUCluster is created/deleted/updated |
| `dpfhcpbridges/status` | update, patch | Required to update the DPUClusterValid condition |
| `events` | create, patch | Required to emit events about validation results |

**Note:** ClusterRole is required (not Role) because DPUCluster can be in any namespace, and DPFHCPBridge supports cross-namespace references.

### Required ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dpf-hcp-bridge-operator
  namespace: dpf-hcp-bridge-operator-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dpf-hcp-bridge-operator-dpucluster-validator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dpf-hcp-bridge-operator-dpucluster-validator
subjects:
  - kind: ServiceAccount
    name: dpf-hcp-bridge-operator
    namespace: dpf-hcp-bridge-operator-system
```

## Testing Strategy

### Unit Tests

#### Test Suite: DPUCluster Validation
**File:** `internal/controller/dpfhcpbridge_dpucluster_validation_test.go`

**Test Cases:**
1. **Test_ValidateDPUCluster_Success**
   - **Given:** DPFHCPBridge with valid `dpuClusterRef` (name + namespace), DPUCluster exists in specified namespace
   - **When:** Reconciliation runs
   - **Then:**
     - `DPUClusterValid` condition is True
     - Reason is `DPUClusterFound`
     - Message includes both namespace and name
     - Event emitted with type Normal
     - No requeue
   - **Mocks:** Fake client with DPUCluster resource in correct namespace

2. **Test_ValidateDPUCluster_NotFound**
   - **Given:** DPFHCPBridge with `dpuClusterRef` that doesn't exist in specified namespace
   - **When:** Reconciliation runs
   - **Then:**
     - `DPUClusterValid` condition is False
     - Reason is `DPUClusterNotFound`
     - Message includes namespace and name
     - Warning event emitted
     - No requeue
   - **Mocks:** Fake client without DPUCluster resource

3. **Test_ValidateDPUCluster_NotSpecified**
   - **Given:** DPFHCPBridge with nil `dpuClusterRef` or empty name/namespace
   - **When:** Reconciliation runs
   - **Then:**
     - `DPUClusterValid` condition is False
     - Reason is `DPUClusterNotSpecified`
     - Warning event emitted
     - No requeue
   - **Mocks:** Fake client

4. **Test_ValidateDPUCluster_AccessError**
   - **Given:** DPFHCPBridge with valid `dpuClusterRef`, API returns permission error
   - **When:** Reconciliation runs
   - **Then:**
     - `DPUClusterValid` condition is False
     - Reason is `DPUClusterAccessError`
     - Warning event emitted
     - Requeue with backoff
   - **Mocks:** Fake client that returns Forbidden error

5. **Test_ValidateDPUCluster_CrossNamespace**
   - **Given:** DPFHCPBridge in namespace A references DPUCluster in namespace B
   - **When:** Reconciliation runs
   - **Then:**
     - Successfully retrieves DPUCluster from namespace B
     - `DPUClusterValid` condition is True
     - Message includes correct namespace
   - **Mocks:** Fake client with DPUCluster in different namespace

6. **Test_ValidateDPUCluster_StatusUpdateFailure**
   - **Given:** DPFHCPBridge with valid `dpuClusterRef`, DPUCluster exists, status update fails
   - **When:** Reconciliation runs
   - **Then:**
     - Error returned
     - Reconciliation will be requeued by controller-runtime
   - **Mocks:** Fake client that fails status updates

7. **Test_ValidateDPUCluster_SpecChange**
   - **Given:** DPFHCPBridge with `dpuClusterRef` changed from A to B (different namespace)
   - **When:** Reconciliation runs
   - **Then:**
     - Validates new DPUCluster B in new namespace
     - ObservedGeneration updated to current Generation
     - Condition reflects validation of new reference
   - **Mocks:** Fake client with DPUCluster B in new namespace

8. **Test_ValidateDPUCluster_IdempotentConditionUpdate**
   - **Given:** DPFHCPBridge already has `DPUClusterValid=True`, DPUCluster still exists
   - **When:** Reconciliation runs multiple times
   - **Then:**
     - Condition remains True
     - LastTransitionTime does NOT change (idempotent)
     - No unnecessary status updates
   - **Mocks:** Fake client with DPUCluster

### Integration Tests
**File:** `test/integration/dpfhcpbridge_dpucluster_test.go`

**Test Cases:**
1. **TestIntegration_DPUClusterValidation_EndToEnd**
   - **Given:** Empty cluster with CRDs installed (using envtest)
   - **When:**
     1. Create DPFHCPBridge referencing non-existent DPUCluster in specific namespace
     2. Verify condition is False
     3. Create the DPUCluster in that namespace
     4. Wait for reconciliation (triggered by watch)
     5. Verify condition becomes True
   - **Then:**
     - Condition transitions from False to True
     - Events are emitted correctly
     - Watch triggers reconciliation automatically
   - **Mocks:** None (real Kubernetes API via envtest)

2. **TestIntegration_DPUClusterDeletion**
   - **Given:** DPFHCPBridge and DPUCluster both exist in different namespaces
   - **When:**
     1. Verify condition is True
     2. Delete DPUCluster
     3. Wait for reconciliation (triggered by watch)
   - **Then:**
     - Condition transitions to False
     - Reason is DPUClusterNotFound
     - Warning event emitted
     - Watch triggers reconciliation automatically
   - **Mocks:** None (real Kubernetes API via envtest)

3. **TestIntegration_FieldIndexing**
   - **Given:** Multiple DPFHCPBridges referencing different DPUClusters
   - **When:** DPUCluster is created/updated/deleted
   - **Then:**
     - Only DPFHCPBridges referencing that specific DPUCluster are reconciled
     - Field index efficiently filters reconciliation targets
   - **Mocks:** None (real Kubernetes API via envtest)

### E2E Tests
**File:** `test/e2e/dpfhcpbridge_dpucluster_test.go`

**Test Cases:**
1. **TestE2E_DPUClusterValidation_RealCluster**
   - **Given:** Real Kubernetes cluster with operator deployed
   - **When:**
     1. Create DPFHCPBridge with invalid DPUCluster reference (wrong namespace)
     2. Monitor status condition
     3. Create DPUCluster in correct namespace
     4. Monitor condition transition
   - **Then:**
     - Condition accurately reflects DPUCluster existence
     - Cross-namespace reference works correctly
     - HostedCluster management blocked when condition is False
   - **Mocks:** None (real cluster)

### Mock Requirements
```go
// Mock client for unit tests
import (
    "sigs.k8s.io/controller-runtime/pkg/client/fake"
    "k8s.io/apimachinery/pkg/runtime"
)

// Create fake client with scheme
scheme := runtime.NewScheme()
_ = dpfv1alpha1.AddToScheme(scheme)
_ = dpuv1alpha1.AddToScheme(scheme) // Add DPUCluster scheme

fakeClient := fake.NewClientBuilder().
    WithScheme(scheme).
    WithObjects(dpuCluster). // Add existing objects
    Build()

// Mock client that returns errors
type ErrorClient struct {
    client.Client
    GetError error
}

func (c *ErrorClient) Get(ctx context.Context, key client.ObjectKey, obj client.Object, opts ...client.GetOption) error {
    if c.GetError != nil {
        return c.GetError
    }
    return c.Client.Get(ctx, key, obj, opts...)
}

// Mock event recorder
import "k8s.io/client-go/tools/record"

fakeRecorder := record.NewFakeRecorder(10) // Buffer size 10
```

## Performance & Scale

### Expected Scale
- **Number of DPFHCPBridge CRs:** 10-100 per cluster (one per DPU deployment scenario)
- **Number of DPUClusters:** 5-50 per cluster (multiple DPU clusters across namespaces)
- **Reconciliation frequency:** Event-driven (on CR/DPUCluster changes) + periodic resync (10 minutes default)
- **Expected duration:** <100ms (single GET request + status update)
- **Concurrent reconciliations:** Up to 5 (MaxConcurrentReconciles)

### Rate Limiting
```go
// Controller manager configuration
import (
    "k8s.io/client-go/util/workqueue"
    ctrl "sigs.k8s.io/controller-runtime"
)

mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Controller: controller.Options{
        MaxConcurrentReconciles: 5,
        RateLimiter: workqueue.NewItemExponentialFailureRateLimiter(
            1*time.Second,  // Base delay
            60*time.Second, // Max delay
        ),
    },
})
```

### Timeout Configuration

| Operation | Timeout | Justification |
|-----------|---------|---------------|
| GET DPUCluster | Inherited from ctx | Controller-runtime provides appropriate timeout |
| Status Update | Inherited from ctx | Controller-runtime provides appropriate timeout |
| Overall Reconciliation | Inherited from ctx | Controller-runtime provides 10s default |

```go
// Use request context directly (best practice)
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // ctx already has appropriate timeout from controller-runtime
    // Just use it directly - DO NOT create new context with timeout

    var dpfhcpBridge dpfv1alpha1.DPFHCPBridge
    if err := r.Get(ctx, req.NamespacedName, &dpfhcpBridge); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // ... use ctx throughout ...
}
```

### Parallelism
**Strategy:** Sequential (within a single DPFHCPBridge reconciliation)
**Rationale:**
- Only one DPUCluster validation per DPFHCPBridge
- No parallel operations needed within single reconciliation
- Multiple DPFHCPBridges reconcile concurrently (controlled by MaxConcurrentReconciles)

**Implementation:**
```go
// Single reconciliation is sequential
func (r *DPFHCPBridgeReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // Step 1: Fetch DPFHCPBridge
    // Step 2: Validate DPUCluster
    // Step 3: Update status
    // Step 4: Continue to next responsibility
}

// Multiple DPFHCPBridges reconcile concurrently
// Controlled by MaxConcurrentReconciles: 5
```

### Caching
The controller-runtime client automatically caches GET/LIST operations:
- DPUCluster resources are cached by the controller-runtime client
- Cache is automatically updated via watch
- No manual cache management needed
- Cache reduces load on API server
- Cache supports cross-namespace queries efficiently

```go
// Client automatically uses cache for GET operations
err := r.Client.Get(ctx, dpuClusterRef, dpuCluster)
// This hits the local cache, not the API server (unless cache is empty)
```

### Field Indexing

Field indexing is **critical** for efficient watch-based reconciliation when DPUClusters change.

**Index Field:** `.spec.dpuClusterRef.name` (for efficient lookups by DPUCluster name)

**Setup in SetupWithManager:**
```go
const dpuClusterRefField = ".spec.dpuClusterRef.name"

func (r *DPFHCPBridgeReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // Add field indexer for efficient lookups
    if err := mgr.GetFieldIndexer().IndexField(
        context.Background(),
        &dpfv1alpha1.DPFHCPBridge{},
        dpuClusterRefField,
        func(obj client.Object) []string {
            bridge := obj.(*dpfv1alpha1.DPFHCPBridge)
            if bridge.Spec.DPUClusterRef == nil || bridge.Spec.DPUClusterRef.Name == "" {
                return nil
            }
            // Index by name only (namespace is filtered in watch handler)
            return []string{bridge.Spec.DPUClusterRef.Name}
        },
    ); err != nil {
        return err
    }

    return ctrl.NewControllerManagedBy(mgr).
        For(&dpfv1alpha1.DPFHCPBridge{}).
        Watches(
            &dpuv1alpha1.DPUCluster{},
            handler.EnqueueRequestsFromMapFunc(r.findDPFHCPBridgesForDPUCluster),
            builder.WithPredicates(predicate.ResourceVersionChangedPredicate{}),
        ).
        Complete(r)
}
```

**Efficient Watch Handler Using Field Index:**
```go
import (
    "sigs.k8s.io/controller-runtime/pkg/predicate"
    "sigs.k8s.io/controller-runtime/pkg/builder"
)

// Map function to trigger reconciliation when DPUCluster changes
func (r *DPFHCPBridgeReconciler) findDPFHCPBridgesForDPUCluster(ctx context.Context, dpuCluster client.Object) []reconcile.Request {
    // Use field index to efficiently find DPFHCPBridges referencing this DPUCluster
    var list dpfv1alpha1.DPFHCPBridgeList
    if err := r.List(ctx, &list,
        client.MatchingFields{dpuClusterRefField: dpuCluster.GetName()},
    ); err != nil {
        r.Log.Error(err, "Failed to list DPFHCPBridges for DPUCluster", "dpuCluster", dpuCluster.GetName())
        return nil
    }

    var requests []reconcile.Request
    for _, bridge := range list.Items {
        // Additional filter: check namespace matches
        if bridge.Spec.DPUClusterRef != nil &&
           bridge.Spec.DPUClusterRef.Namespace == dpuCluster.GetNamespace() {
            requests = append(requests, reconcile.Request{
                NamespacedName: types.NamespacedName{
                    Name:      bridge.Name,
                    Namespace: bridge.Namespace,
                },
            })
        }
    }

    return requests
}
```

**Predicates for Watch Efficiency:**
```go
// Only trigger reconciliation when DPUCluster actually changes
builder.WithPredicates(predicate.ResourceVersionChangedPredicate{})
```

**Benefits of Field Indexing:**
- **Efficiency:** O(1) lookup instead of O(n) list-and-filter
- **Scale:** Handles 100s of DPFHCPBridges efficiently
- **Watch Performance:** Only reconciles affected DPFHCPBridges when DPUCluster changes

## Observability

### Logging

#### Log Levels
**Info Level:**
- "Validating DPUCluster existence" (at start of validation)
- "DPUCluster validation successful" (when validation passes)
- "DPUCluster not found" (when DPUCluster doesn't exist)

**Debug Level:**
- "Fetching DPUCluster from API" (before GET operation)
- "DPUCluster validation result" (detailed validation outcome)
- "Field index lookup for DPUCluster" (watch handler performance)

**Error Level:**
- "Failed to get DPUCluster" (on API errors other than NotFound)
- "Failed to update DPFHCPBridge status" (on status update failures)
- "Field indexing error" (if indexing fails)

```go
// Example logging code using controller-runtime logger
import (
    "sigs.k8s.io/controller-runtime/pkg/log"
)

func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx).WithValues(
        "dpfhcpbridge", req.NamespacedName,
    )

    var dpfhcpBridge dpfv1alpha1.DPFHCPBridge
    if err := r.Get(ctx, req.NamespacedName, &dpfhcpBridge); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    dpuClusterRef, err := extractDPUClusterRef(&dpfhcpBridge)
    if err != nil {
        log.Info("DPUCluster reference not specified", "error", err)
        // ... handle error ...
    }

    log.Info("Validating DPUCluster existence",
        "dpuClusterNamespace", dpuClusterRef.Namespace,
        "dpuClusterName", dpuClusterRef.Name)

    dpuCluster := &dpuv1alpha1.DPUCluster{}
    err = r.Get(ctx, dpuClusterRef, dpuCluster)
    if err != nil {
        if apierrors.IsNotFound(err) {
            log.Info("DPUCluster not found",
                "namespace", dpuClusterRef.Namespace,
                "name", dpuClusterRef.Name)
        } else {
            log.Error(err, "Failed to get DPUCluster",
                "namespace", dpuClusterRef.Namespace,
                "name", dpuClusterRef.Name)
        }
    } else {
        log.Info("DPUCluster validation successful",
            "namespace", dpuClusterRef.Namespace,
            "name", dpuClusterRef.Name)
    }
}
```

### Metrics

```go
// Metric definitions
import (
    "github.com/prometheus/client_golang/prometheus"
    "sigs.k8s.io/controller-runtime/pkg/metrics"
)

var (
    dpuClusterValidationTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "dpfhcpbridge_dpucluster_validation_total",
            Help: "Total number of DPUCluster validations performed",
        },
        []string{"result"}, // result: success, not_found, error
    )

    dpuClusterValidationDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "dpfhcpbridge_dpucluster_validation_duration_seconds",
            Help:    "Duration of DPUCluster validation operations",
            Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1.0, 5.0},
        },
        []string{"result"},
    )
)

// Register metrics in SetupWithManager (not init)
func (r *Reconciler) SetupWithManager(mgr ctrl.Manager) error {
    // Register metrics with error handling
    if err := metrics.Registry.Register(dpuClusterValidationTotal); err != nil {
        if _, ok := err.(prometheus.AlreadyRegisteredError); !ok {
            return err
        }
    }
    if err := metrics.Registry.Register(dpuClusterValidationDuration); err != nil {
        if _, ok := err.(prometheus.AlreadyRegisteredError); !ok {
            return err
        }
    }

    // ... continue with controller setup ...
}
```

**Metrics to track:**
- `dpfhcpbridge_dpucluster_validation_total{result="success"}`: Counter of successful validations
- `dpfhcpbridge_dpucluster_validation_total{result="not_found"}`: Counter of validations where DPUCluster not found
- `dpfhcpbridge_dpucluster_validation_total{result="error"}`: Counter of validation errors
- `dpfhcpbridge_dpucluster_validation_duration_seconds`: Histogram of validation duration

### Events
```go
// Event emission examples using controller-runtime recorder
import (
    corev1 "k8s.io/api/core/v1"
)

// Success
r.Recorder.Event(dpfhcpBridge, corev1.EventTypeNormal, "DPUClusterValidated",
    fmt.Sprintf("DPUCluster '%s/%s' validated successfully", dpuClusterRef.Namespace, dpuClusterRef.Name))

// Not Found
r.Recorder.Event(dpfhcpBridge, corev1.EventTypeWarning, "DPUClusterNotFound",
    fmt.Sprintf("Referenced DPUCluster '%s/%s' not found", dpuClusterRef.Namespace, dpuClusterRef.Name))

// Access Error
r.Recorder.Event(dpfhcpBridge, corev1.EventTypeWarning, "DPUClusterAccessError",
    fmt.Sprintf("Failed to access DPUCluster '%s/%s'", dpuClusterRef.Namespace, dpuClusterRef.Name))
```

### Distributed Tracing
Not applicable for this feature. This is a simple, synchronous operation that doesn't warrant distributed tracing.

## Dependencies

### Go Dependencies

```go
// go.mod additions (most already present in typical operator)
require (
    k8s.io/api v0.28.0              // Kubernetes API types
    k8s.io/apimachinery v0.28.0     // Meta types (Condition, NamespacedName, etc.)
    k8s.io/client-go v0.28.0        // Kubernetes client
    sigs.k8s.io/controller-runtime v0.16.0  // Controller framework and client
    github.com/go-logr/logr v1.2.4  // Logging interface
    github.com/prometheus/client_golang v1.16.0  // Metrics
)
```

### Dependency Matrix

| Dependency | Version | Min K8s | License | Purpose |
|------------|---------|---------|---------|---------|
| controller-runtime | v0.16.0 | 1.27 | Apache-2.0 | Controller framework, client, field indexing |
| client-go | v0.28.0 | 1.27 | Apache-2.0 | Kubernetes API types and errors |
| apimachinery | v0.28.0 | 1.27 | Apache-2.0 | Meta API types (Condition, NamespacedName) |
| go-logr | v1.2.4 | N/A | Apache-2.0 | Logging interface |
| prometheus/client_golang | v1.16.0 | N/A | Apache-2.0 | Metrics instrumentation |

### Compatibility Validation
- **controller-runtime v0.16.0** is compatible with Kubernetes 1.27-1.29 (target: 1.27+)
- **client-go v0.28.0** matches Kubernetes 1.28 (supports 1.27-1.29)
- **metav1.Condition** is available in Kubernetes 1.19+ (well within our target range)
- **Field indexing** is available in controller-runtime v0.6.0+ (v0.16.0 has full support)
- **Go 1.21+** required for controller-runtime v0.16.0

## Implementation Notes

### Code Structure
```
internal/controller/
├── dpfhcpbridge_controller.go           # Main controller
├── dpfhcpbridge_dpucluster_validation.go  # DPUCluster validation logic
└── dpfhcpbridge_dpucluster_validation_test.go  # Unit tests

test/
├── integration/
│   └── dpfhcpbridge_dpucluster_test.go  # Integration tests
└── e2e/
    └── dpfhcpbridge_dpucluster_test.go  # E2E tests
```

### Key Functions
```go
// Main reconciliation entry point
func (r *DPFHCPBridgeReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)

// DPUCluster validation function
func (r *DPFHCPBridgeReconciler) validateDPUClusterExists(
    ctx context.Context,
    dpfhcpBridge *dpfv1alpha1.DPFHCPBridge,
) (bool, error)

// Condition update helper
func (r *DPFHCPBridgeReconciler) setDPUClusterValidCondition(
    ctx context.Context,
    dpfhcpBridge *dpfv1alpha1.DPFHCPBridge,
    valid bool,
    reason string,
    message string,
) error

// Decision function for continuing reconciliation
func shouldProceedToHostedClusterManagement(dpfhcpBridge *dpfv1alpha1.DPFHCPBridge) bool

// Watch handler for DPUCluster changes
func (r *DPFHCPBridgeReconciler) findDPFHCPBridgesForDPUCluster(
    ctx context.Context,
    dpuCluster client.Object,
) []reconcile.Request
```

### Best Practices
- **Use structured logging**: Always include relevant context (namespace/name) in log entries
- **Emit events for user visibility**: Users rely on events to understand validation failures
- **Leverage meta.SetStatusCondition**: Ensures idempotent condition updates with automatic LastTransitionTime handling
- **Use apierrors helpers**: Distinguish NotFound from other errors for proper handling (apierrors.IsNotFound(), apierrors.IsForbidden())
- **Set ObservedGeneration**: Always set this to dpfhcpBridge.Generation to reflect which spec version was validated
- **Don't requeue on user errors**: If DPUCluster is missing, wait for user action (create DPUCluster or update DPFHCPBridge)
- **Do requeue on transient errors**: API server issues should trigger requeue with backoff
- **Use request context**: Don't create new contexts with timeout - use the provided ctx from Reconcile
- **Record metrics**: Track validation outcomes for observability
- **Test idempotency**: Ensure multiple reconciliations produce the same result
- **Separate concerns**: Keep validation logic separate from HostedCluster lifecycle management
- **Field indexing is critical**: Always implement for watch-based reconciliation efficiency
- **Use client.IgnoreNotFound()**: Best practice for handling CR deletion in reconcile
- **ClusterRole for cross-namespace**: Since DPUCluster can be in any namespace, use ClusterRole (not Role)

### Controller Setup with Field Indexing and Predicates
```go
const dpuClusterRefField = ".spec.dpuClusterRef.name"

func (r *DPFHCPBridgeReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // Add field indexer for efficient lookups
    if err := mgr.GetFieldIndexer().IndexField(
        context.Background(),
        &dpfv1alpha1.DPFHCPBridge{},
        dpuClusterRefField,
        func(obj client.Object) []string {
            bridge := obj.(*dpfv1alpha1.DPFHCPBridge)
            if bridge.Spec.DPUClusterRef == nil || bridge.Spec.DPUClusterRef.Name == "" {
                return nil
            }
            return []string{bridge.Spec.DPUClusterRef.Name}
        },
    ); err != nil {
        return err
    }

    return ctrl.NewControllerManagedBy(mgr).
        For(&dpfv1alpha1.DPFHCPBridge{}).
        Watches(
            &dpuv1alpha1.DPUCluster{},
            handler.EnqueueRequestsFromMapFunc(r.findDPFHCPBridgesForDPUCluster),
            builder.WithPredicates(predicate.ResourceVersionChangedPredicate{}),
        ).
        Complete(r)
}

// Efficient watch handler using field index
func (r *DPFHCPBridgeReconciler) findDPFHCPBridgesForDPUCluster(ctx context.Context, dpuCluster client.Object) []reconcile.Request {
    var list dpfv1alpha1.DPFHCPBridgeList
    if err := r.List(ctx, &list,
        client.MatchingFields{dpuClusterRefField: dpuCluster.GetName()},
    ); err != nil {
        return nil
    }

    var requests []reconcile.Request
    for _, bridge := range list.Items {
        // Filter by namespace
        if bridge.Spec.DPUClusterRef != nil &&
           bridge.Spec.DPUClusterRef.Namespace == dpuCluster.GetNamespace() {
            requests = append(requests, reconcile.Request{
                NamespacedName: types.NamespacedName{
                    Name:      bridge.Name,
                    Namespace: bridge.Namespace,
                },
            })
        }
    }

    return requests
}
```

### Error Wrapping
```go
import "fmt"

// Wrap errors with context for better debugging
if err := r.Get(ctx, dpuClusterRef, dpuCluster); err != nil {
    return fmt.Errorf("failed to get DPUCluster '%s/%s': %w", dpuClusterRef.Namespace, dpuClusterRef.Name, err)
}

// Use %w to preserve error type for apierrors.IsNotFound() checks
```

### Owner References (for created resources)
This validation feature doesn't create resources, but for reference when other features create resources:
```go
import "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"

// For any resources created by the operator:
if err := controllerutil.SetControllerReference(dpfhcpBridge, resource, r.Scheme); err != nil {
    return err
}
```

## Open Questions

None. All technical details have been clarified during the interview and operator-SDK alignment review.

## References

- [Kubernetes Conditions API](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties)
- [Controller-Runtime Client](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client)
- [Controller-Runtime Field Indexing](https://book.kubebuilder.io/reference/watching-resources/externally-managed.html)
- [meta.SetStatusCondition](https://pkg.go.dev/k8s.io/apimachinery/pkg/api/meta#SetStatusCondition)
- [Kubernetes Events](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/)
- [Controller-Runtime Predicates](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/predicate)
- [Kubebuilder Book - Watching Resources](https://book.kubebuilder.io/reference/watching-resources.html)

---

**Status:** Ready for implementation by Developer agent (Operator-SDK Aligned)

**Next Steps:**
1. Developer agent implements following this spec
2. Implement field indexing in SetupWithManager
3. Implement unit tests achieving >90% coverage
4. Implement integration tests using envtest (test field indexing)
5. Run E2E tests in real cluster (test cross-namespace access)
6. Update operator documentation with validation behavior
7. Proceed to next DPFHCPBridge responsibility (HostedCluster lifecycle management)
