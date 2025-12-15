# Operator Product Requirements Document

## Metadata
- **Operator Name**: dpf-hcp-bridge-operator
- **Created**: 2025-12-17
- **Last Updated**: 2025-12-17

## Problem Statement

The DPF-HCP Bridge Operator addresses the complexity of manually creating and managing HyperShift HostedCluster resources for DPU-based infrastructure. Currently, users must perform several intricate manual tasks including creating and configuring HostedCluster resources, setting up MetalLB for load balancing, downloading the HostedCluster kubeconfig and injecting it into the DPUCluster CR, and manually approving Certificate Signing Requests (CSRs) for worker nodes. These tasks require deep knowledge of HyperShift architecture and are error-prone when performed manually.

The operator automates this entire workflow by providing a simplified Custom Resource (DPFHCPBridge) that abstracts away the complexity of HyperShift HostedCluster management. Users can declaratively specify their requirements, and the operator handles the orchestration of HostedCluster creation, MetalLB configuration, kubeconfig injection, and CSR auto-approval, significantly reducing the operational burden and expertise required.

## Target Users

Target users to be determined (Open Question).

## Operator Overview

### Purpose
The DPF-HCP Bridge Operator simplifies the deployment and management of HyperShift HostedClusters for DPU-based infrastructure by automating the creation, configuration, and lifecycle management of HostedCluster resources and their dependencies.

## Custom Resources

### DPFHCPBridge

#### Description
One DPFHCPBridge CR represents one HyperShift HostedCluster that will be created and managed on behalf of a DPUCluster.

#### Lifecycle
1. **Created**: The operator validates that the referenced DPUCluster exists, creates the corresponding HostedCluster resource, configures MetalLB for load balancing (if specified), performs BlueField image validation and injection, creates a Secret containing the HostedCluster kubeconfig and references it in the DPUCluster CR, and enables automatic CSR approval for worker nodes.

2. **Ready**: The HostedCluster has reached ready state, MetalLB configuration is successfully applied, BlueField image injection is complete, the kubeconfig Secret has been created and referenced in the DPUCluster CR, and CSR auto-approval is actively running.

3. **Updated**: Only limited fields can be updated (infrastructureAvailabilityPolicy, ocpReleaseImage, metalLBVirtualIP). When these fields change, the operator reconciles the changes to the HostedCluster or MetalLB configuration and re-validates BlueField image compatibility for ocpReleaseImage changes.

4. **Deleted**: The operator deletes the HostedCluster, removes MetalLB configuration, stops CSR auto-approval, removes the kubeconfig Secret and its reference from the DPUCluster CR. Finalizers ensure proper cleanup ordering: HostedCluster deletion completes first, followed by MetalLB cleanup, CSR auto-approval termination, and kubeconfig removal before the DPFHCPBridge CR is fully deleted.

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `dpuClusterRef` | object | Yes | Reference to the DPUCluster CR (name and namespace) |
| `clusterName` | string | Yes | Name for the hosted cluster (becomes HostedCluster name) |
| `baseDomain` | string | Yes | Base domain for the hosted cluster |
| `ocpReleaseImage` | string | Yes | Full pull-spec URL for the OCP release image |
| `sshKeySecretRef` | object | Yes | Reference to Secret containing SSH public key |
| `pullSecretRef` | object | Yes | Reference to Secret containing pull secret |
| `etcdStorageClass` | string | Yes | Storage class name for etcd data volumes |
| `controlPlaneAvailabilityPolicy` | string | Yes | `SingleReplica` or `HighlyAvailable` |
| `infrastructureAvailabilityPolicy` | string | Yes | `SingleReplica` or `HighlyAvailable` |
| `exposeThroughLoadBalancer` | boolean | Yes | If true, use LoadBalancer; if false, use NodePort |
| `metalLBVirtualIP` | string | Conditional | Required if `exposeThroughLoadBalancer` is true |
| `clusterType` | string | Yes | DPUCluster type identifier (for validation) |

## CR Relationships

DPFHCPBridge has a one-to-one relationship with DPUCluster. Each DPFHCPBridge CR references exactly one DPUCluster CR.

## Operator Responsibilities

### Responsibility 1: Creating, Updating, and Deleting HostedCluster

**Trigger**:
- CREATE: When a DPFHCPBridge CR is created (after all validations pass and MetalLB is configured)
- UPDATE: When DPFHCPBridge spec fields change (only infrastructureAvailabilityPolicy or ocpReleaseImage affect HostedCluster; metalLBVirtualIP updates MetalLB configuration, not HostedCluster)
- DELETE: When user deletes DPFHCPBridge CR

**Outcome**:
- CREATE: The operator creates a corresponding HostedCluster CR with configuration derived from the DPFHCPBridge spec fields including clusterName, baseDomain, ocpReleaseImage, sshKeySecretRef, pullSecretRef, etcdStorageClass, controlPlaneAvailabilityPolicy, infrastructureAvailabilityPolicy, and exposeThroughLoadBalancer settings.
- UPDATE: The operator updates the HostedCluster CR to reflect the new configuration after validating the ocpReleaseImage against the ocp-bluefield-images ConfigMap and ensuring the HostedCluster is in ready state before accepting the update.
- DELETE: The HostedCluster and all associated resources are cleaned up following the deletion sequence: operator detects deletion (finalizer prevents immediate removal), deletes HostedCluster, removes MetalLB configuration, stops CSR auto-approval, removes kubeconfig from DPUCluster, removes finalizer, and then DPFHCPBridge is fully deleted.

**Ordering Constraints**:
- CREATE: Must occur after DPUCluster validation, BlueField image validation, and MetalLB configuration
- UPDATE: Must validate HostedCluster ready state before proceeding; ocpReleaseImage validation must occur before HostedCluster update
- DELETE: HostedCluster deletion must complete before other cleanup operations proceed

**Success Criteria**:
- CREATE: HostedCluster CR exists with correct configuration
- UPDATE: HostedCluster CR reflects updated configuration and returns to ready state
- DELETE: HostedCluster CR is fully removed from the cluster

**Edge Cases**:
- UPDATE: If HostedCluster is not in ready state, the operator rejects the update request. If the HostedCluster fails to update and become ready, the operator reflects the failure in the DPFHCPBridge status and continues retrying.
- DELETE: If HostedCluster deletion fails or gets stuck, the operator reflects the failure in the DPFHCPBridge status and continues retrying. The operator waits for HostedCluster to be fully deleted before proceeding with other cleanup.

### Responsibility 2: Creating, Updating, and Deleting MetalLB Configuration

**Trigger**:
- CREATE: DPFHCPBridge CR specifies exposeThroughLoadBalancer=true
- UPDATE: metalLBVirtualIP field in DPFHCPBridge spec is changed
- DELETE: DPFHCPBridge CR is deleted

**Outcome**:
- CREATE: The operator configures MetalLB with the appropriate resources (IPAddressPool and L2Advertisement) using the metalLBVirtualIP from the DPFHCPBridge spec.
- UPDATE: The operator updates the MetalLB configuration with the new VIP.
- DELETE: The operator removes MetalLB configuration resources (IPAddressPool and L2Advertisement).

**Ordering Constraints**: MetalLB configuration must happen BEFORE HostedCluster creation.

**Success Criteria**: MetalLB resources (IPAddressPool and L2Advertisement) exist and are correctly configured with the specified VIP.

**Edge Cases**:
- If MetalLB operator is not installed, the operator reflects an error in DPFHCPBridge status.
- If the VIP address is invalid or conflicts with existing configuration, the operator reports a validation error in DPFHCPBridge status.

### Responsibility 3: Downloading HostedCluster Kubeconfig and Injecting into DPUCluster

**Trigger**: HostedCluster becomes ready (HostedCluster status indicates ready state).

**Outcome**: The operator downloads the HostedCluster's kubeconfig, creates a Secret in the DPUCluster's namespace containing the kubeconfig, and adds a reference to this Secret in the DPUCluster CR's status field.

**Ordering Constraints**:
- Must wait for HostedCluster to be ready before downloading kubeconfig
- Kubeconfig must be available before worker nodes can join

**Success Criteria**:
- Secret exists with the kubeconfig
- DPUCluster CR contains the correct reference to this Secret

**Edge Cases**:
- If kubeconfig download fails, the operator retries with backoff and reflects the error in DPFHCPBridge status.
- If Secret creation fails, the operator retries with backoff and reflects the error in DPFHCPBridge status.
- If DPUCluster CR update fails, the operator retries with backoff and reflects the error in DPFHCPBridge status.
- On DPFHCPBridge deletion, the operator removes the kubeconfig Secret and its reference from the DPUCluster.

### Responsibility 4: CSR Auto-Approval for DPU Worker Nodes

**Trigger**: HostedCluster is ready and worker nodes start joining.

**Outcome**: The operator automatically approves Certificate Signing Requests (CSRs) for worker nodes attempting to join the HostedCluster.

**Ordering Constraints**:
- Must start after HostedCluster is ready
- Must continue running as long as DPFHCPBridge exists
- Must stop when DPFHCPBridge is deleted

**Success Criteria**: Worker node CSRs are approved automatically without manual intervention.

**Edge Cases**:
- If CSR approval logic encounters an error, the operator logs the error, reflects it in DPFHCPBridge status, and continues attempting to approve subsequent CSRs.
- The operator only approves CSRs for the specific HostedCluster, not other clusters.
- On DPFHCPBridge deletion, the operator stops auto-approving CSRs for that HostedCluster.

### Responsibility 5: Validating DPUCluster Existence

**Trigger**: DPFHCPBridge CR is created.

**Outcome**: Before creating the HostedCluster, the operator validates that the referenced DPUCluster CR exists.

**Ordering Constraints**: Must happen before any other operations (first validation step).

**Success Criteria**: DPUCluster CR exists and is accessible.

**Edge Cases**:
- If DPUCluster does not exist, the operator reports an error in DPFHCPBridge status, does not proceed with HostedCluster creation, and retries validation periodically.
- If DPUCluster exists but is in an unexpected state: What constitutes "unexpected state" needs to be defined (Open Question).

### Responsibility 6: BlueField Image Retrieval and Injection

**Trigger**:
- When DPFHCPBridge CR is created
- When ocpReleaseImage field is updated

**Outcome**: The operator validates that the provided ocpReleaseImage exists in the ocp-bluefield-images ConfigMap (ensuring compatibility with BlueField DPUs) and creates a new ConfigMap containing the BlueField container image to override the osImageURL in ignition.

**Ordering Constraints**: Must happen before creating or updating the HostedCluster.

**Success Criteria**:
- The ocpReleaseImage is found in the ocp-bluefield-images ConfigMap
- New ConfigMap is created with BlueField container image for ignition override

**Edge Cases**:
- If ocpReleaseImage is not in the ConfigMap, the operator rejects the create/update request and reports a validation error in DPFHCPBridge status.
- If ocp-bluefield-images ConfigMap is missing or malformed, the operator reports an error in DPFHCPBridge status and prevents HostedCluster creation/update.
- If new ConfigMap creation fails, the operator retries with backoff and reflects the error in DPFHCPBridge status.

## External Dependencies

### DPUCluster CRD

- **Type**: Kubernetes Custom Resource Definition
- **Purpose**: Represents a DPU-based cluster that the HostedCluster will be associated with. The operator validates the existence of the DPUCluster and injects the HostedCluster kubeconfig into the DPUCluster CR's status field.
- **Interaction**: The operator reads the DPUCluster CR to validate its existence, and updates the DPUCluster CR's status field to add a reference to the kubeconfig Secret.
- **Required**: Yes
- **References**: Provided by DPF Operator

### HostedCluster CRD

- **Type**: Kubernetes Custom Resource Definition (provided by HyperShift Operator)
- **Purpose**: Represents a hosted Kubernetes cluster managed by HyperShift. This is the core resource that the operator creates and manages based on the DPFHCPBridge specification.
- **Interaction**: The operator creates, updates, and deletes HostedCluster CRs based on DPFHCPBridge lifecycle events.
- **Required**: Yes
- **References**: Provided by HyperShift Operator

### MetalLB Operator

- **Type**: Kubernetes Operator
- **Purpose**: Provides software-based load balancing for the HostedCluster control plane. When deploying a highly available HyperShift cluster, the control plane requires a stable, single VIP that multiple control plane replicas can share.
- **Interaction**: The operator creates and manages MetalLB resources (IPAddressPool and L2Advertisement) when exposeThroughLoadBalancer is set to true.
- **Required**: Yes (when exposeThroughLoadBalancer=true)
- **References**: MetalLB Operator must be installed on the management cluster

### ocp-bluefield-images ConfigMap

- **Type**: Kubernetes ConfigMap
- **Purpose**: Contains a mapping of OCP release images to compatible BlueField DPU container images. The operator uses this ConfigMap to validate that the specified ocpReleaseImage is compatible with BlueField hardware and to retrieve the appropriate BlueField image for ignition override.
- **Interaction**: The operator reads the ConfigMap to validate ocpReleaseImage entries and retrieve corresponding BlueField container images for creating the ignition override ConfigMap.
- **Required**: Yes
- **References**: Must be present on the management cluster before DPFHCPBridge CRs can be created

## Prerequisites

Before deploying the DPF-HCP Bridge Operator, the following prerequisites must be satisfied:

1. **Helm** - Package manager for Kubernetes, used to provision the DPF-HCP Bridge Operator.

2. **MGMT Cluster** - An OpenShift cluster must be up and running to host the operator and associated resources.

3. **MCE Operator** - Must be deployed on the MGMT cluster to deliver and manage the HyperShift Operator.

4. **DPF Operator** - Must be deployed on the MGMT cluster. This operator provides the DPUCluster CRD that DPFHCPBridge references.

5. **HyperShift Operator** - Must be deployed on the MGMT cluster. This operator provides the HostedCluster CRD that the DPF-HCP Bridge Operator creates and manages.

6. **ODF Operator** - The Hosted Control Plane's etcd requires persistent storage. OpenShift Data Foundation (ODF) must be installed on the MGMT cluster to provide this storage.

7. **MetalLB Operator** - When deploying a highly available HyperShift cluster, the control plane needs a stable, single VIP that the three control plane replicas can share. MetalLB Operator must be installed to provide this functionality, acting as a software load balancer.

8. **ocp-bluefield-images ConfigMap** - Must be present on the management cluster containing the mapping of OCP release images to compatible BlueField DPU container images.

## Deployment

**Deployment Method**: Helm

**Configuration Options**: To be determined based on deployment requirements and architectural decisions.

## Success Metrics

Users can verify that the operator is working correctly through the following indicators:

- **DPFHCPBridge Status**: The DPFHCPBridge CR transitions to ready state.
- **HostedCluster Creation**: HostedCluster resources are successfully created and reach Ready state.
- **Kubeconfig Injection**: The DPUCluster CR includes the kubeconfig reference in its status field.
- **Worker Node Readiness**: Worker nodes in the hosted cluster transition to Ready state after CSRs are approved successfully.
- **MetalLB Configuration**: MetalLB configuration is applied successfully (when exposeThroughLoadBalancer=true).
- **Kubeconfig Accessibility**: Users can retrieve the hosted cluster kubeconfig from the DPUCluster resource.
- **Cluster Connectivity**: Users can connect to the hosted cluster using the retrieved kubeconfig.
- **Clear Error Messages**: When validation fails, clear status messages are provided in the DPFHCPBridge status field.

## User Workflows

### Workflow 1: Create DPFHCPBridge CR

**User**: Cluster administrator or infrastructure operator

1. User creates prerequisite Secrets (SSH key, pull secret) in the appropriate namespace.
2. User creates a DPFHCPBridge CR with the desired configuration (cluster name, domain, OCP version, etc.).
3. Operator validates the DPUCluster reference and ocpReleaseImage compatibility.
4. Operator configures MetalLB (if specified).
5. Operator creates the HostedCluster resource.
6. Operator waits for HostedCluster to become ready.
7. Operator downloads the kubeconfig and creates a Secret.
8. Operator updates the DPUCluster CR with the kubeconfig reference.
9. Operator enables CSR auto-approval for worker nodes.
10. Expected outcome: DPFHCPBridge CR reaches ready state, and users can access the hosted cluster via the kubeconfig.

### Workflow 2: Update DPFHCPBridge CR

**User**: Cluster administrator or infrastructure operator

1. User modifies one of the updateable fields in the DPFHCPBridge CR (infrastructureAvailabilityPolicy, ocpReleaseImage, or metalLBVirtualIP).
2. Operator validates the change (e.g., verifies new ocpReleaseImage is in ocp-bluefield-images ConfigMap).
3. Operator ensures HostedCluster is in ready state before proceeding.
4. Operator updates the corresponding HostedCluster or MetalLB configuration.
5. Expected outcome: Changes are applied successfully, and the HostedCluster returns to ready state.

### Workflow 3: Upgrade OCP Version via DPFHCPBridge CR

**User**: Cluster administrator or infrastructure operator

1. User updates the ocpReleaseImage field in the DPFHCPBridge CR to a new OCP version.
2. Operator validates the new ocpReleaseImage against the ocp-bluefield-images ConfigMap.
3. Operator ensures HostedCluster is in ready state before proceeding.
4. Operator updates the HostedCluster with the new release image.
5. Operator validates BlueField image compatibility and creates/updates the ignition override ConfigMap.
6. Expected outcome: HostedCluster upgrades to the new OCP version and returns to ready state.

### Workflow 4: Delete DPFHCPBridge CR

**User**: Cluster administrator or infrastructure operator

1. User deletes the DPFHCPBridge CR.
2. Operator detects deletion (finalizer prevents immediate removal).
3. Operator deletes the HostedCluster resource.
4. Operator waits for HostedCluster deletion to complete.
5. Operator removes MetalLB configuration.
6. Operator stops CSR auto-approval for the HostedCluster.
7. Operator removes the kubeconfig Secret and its reference from the DPUCluster CR.
8. Operator removes the finalizer from the DPFHCPBridge CR.
9. Expected outcome: DPFHCPBridge CR and all associated resources are fully cleaned up.

### Workflow 5: Troubleshoot Failed DPFHCPBridge CR

**User**: Cluster administrator or infrastructure operator

1. User creates a DPFHCPBridge CR that fails to reach ready state.
2. User examines the DPFHCPBridge status field for error messages.
3. User identifies the issue (e.g., missing DPUCluster, invalid ocpReleaseImage, MetalLB not installed).
4. User corrects the issue (e.g., creates missing prerequisites, updates spec with valid values).
5. Operator automatically retries and reconciles the DPFHCPBridge CR.
6. Expected outcome: DPFHCPBridge CR successfully reconciles and reaches ready state.

### Workflow 6: View DPFHCPBridge Status and Health

**User**: Cluster administrator or infrastructure operator

1. User queries the DPFHCPBridge CR status using kubectl or OpenShift console.
2. User views the status field containing information about:
   - HostedCluster ready state
   - MetalLB configuration status
   - Kubeconfig injection status
   - CSR auto-approval status
   - Any error messages or validation failures
3. Expected outcome: User has clear visibility into the health and state of the DPFHCPBridge and associated resources.

### Workflow 7: Update Secrets (Pull Secret, SSH Key)

**User**: Cluster administrator or infrastructure operator

1. User needs to rotate or update the pull secret or SSH key.
2. User updates the Secret resource referenced by the DPFHCPBridge CR.
3. User updates the DPFHCPBridge CR to trigger reconciliation (may depend on operator design).
4. Operator detects the change and updates the HostedCluster configuration.
5. Expected outcome: HostedCluster uses the updated secrets.

## Observability Requirements

**Metrics** (what should be tracked):
- Standard controller-runtime metrics (reconciliation rate, queue depth, error counts)

**Logs** (what should be logged):
- DPFHCPBridge CR creation, update, and deletion events
- HostedCluster creation, update, and deletion events
- MetalLB configuration creation, update, and deletion events
- Kubeconfig download and Secret creation events
- CSR approval events
- Validation failures (DPUCluster not found, ocpReleaseImage not in ConfigMap, etc.)
- Reconciliation errors and retry attempts

**Events** (Kubernetes events to emit):
- Success events when DPFHCPBridge CR becomes ready
- Success events when HostedCluster is created and becomes ready
- Success events when kubeconfig is injected into DPUCluster
- Warning events on validation failures (missing DPUCluster, invalid ocpReleaseImage, MetalLB issues)
- Warning events on reconciliation errors (HostedCluster creation/update failures, kubeconfig download failures)

## Open Questions

1. **Target Users**: The specific roles and personas who will use this operator need to be identified.

2. **Scale**: The expected number of DPFHCPBridge CRs that will be managed by the operator needs to be determined.

3. **DPUCluster Unexpected State**: The definition of what constitutes an "unexpected state" for a DPUCluster CR needs to be clarified for validation purposes.
