# Operator Product Requirements Document

## Metadata
- **Operator Name**: DPF-HCP Bridge Operator
- **Created**: 2025-12-18
- **Last Updated**: 2025-12-18

## Problem Statement

The DPF-HCP Bridge Operator addresses the complexity of manually creating and managing HyperShift HostedCluster resources for DPF Installation On Hypershift. Currently, the process of creating, configuring, updating, and deleting HostedClusters on DPU infrastructure requires manual intervention and coordination between multiple systems.

The operator automates the entire lifecycle management of HostedClusters on DPU infrastructure. It eliminates the need for manual HostedCluster creation, configuration management, updates, and deletion, while seamlessly bridging the connection between DPU clusters and their corresponding hosted control planes.

By automating these manual processes, the operator reduces operational complexity, minimizes human error, and enables platform teams to efficiently manage OpenShift Hosted Control Planes at scale on DPU infrastructure.

## Operator Overview

### Purpose
The DPF-HCP Bridge Operator addresses the complexity of manually creating and managing HyperShift HostedCluster resources for DPF Installation On Hypershift. It automates HostedCluster lifecycle operations, kubeconfig distribution, and cleanup, ensuring seamless integration between the DPU infrastructure layer and the hosted OpenShift control planes.

## Custom Resources

### DPFHCPBridge

#### Description
One instance of the DPFHCPBridge Custom Resource represents a single bridge between a DPU cluster and a HostedCluster. This CR encapsulates the configuration and state of the relationship between the DPU infrastructure and its associated hosted control plane.

Note: DPFHCPBridge is the ONLY Custom Resource managed by this operator. The operator references DPUCluster CRs (managed by the DPFOperator) but does not own or manage them.

#### Lifecycle

1. **Created**: When a DPFHCPBridge CR is created, the operator validates that the referenced DPUCluster exists, then creates a corresponding HostedCluster CR in the management cluster with configuration derived from the DPFHCPBridge spec.

2. **Ready**: The DPFHCPBridge CR reaches the Ready state when both of the following conditions are met:
   - The referenced HostedCluster has reached Ready state
   - The HostedCluster's kubeconfig has been downloaded and successfully injected into the referenced DPUCluster (stored as a Secret with a reference added to the DPUCluster CR's status field)

3. **Updated**: When spec fields in the DPFHCPBridge CR are modified, the operator first validates the changes, then updates the corresponding HostedCluster with the new configuration.

4. **Deleted**: When a DPFHCPBridge CR is deleted, the operator performs cleanup by deleting the HostedCluster from the DPUCluster and removing the kubeconfig reference from the DPUCluster CR.

## CR Relationships

The DPFHCPBridge CR references a DPUCluster CR as an external dependency. The DPUCluster is managed by a separate operator (DPFOperator) and exists outside the scope of the DPF-HCP Bridge Operator. The relationship is a reference-only relationship where the DPFHCPBridge points to an existing DPUCluster but does not own or manage it.

## Operator Responsibilities (High-Level)

### Responsibility 1: Validating DPUCluster Existence

**Trigger**: A DPFHCPBridge CR is created or updated

**Outcome**: The operator verifies that the DPUCluster CR referenced in the DPFHCPBridge spec exists in the cluster

This validation ensures that the bridge can only be established to valid, existing DPU infrastructure, preventing orphaned HostedClusters and configuration errors.

### Responsibility 2: Managing HostedCluster Lifecycle

**Trigger**: A DPFHCPBridge CR is created, updated, or deleted

**Outcome**: The operator creates, updates, or deletes the HostedCluster CR in the management cluster based on the DPFHCPBridge CR lifecycle

This responsibility automates the complete lifecycle management of the hosted control plane infrastructure based on the desired configuration specified by the user.

### Responsibility 3: Managing DPFHCPBridge Status

**Trigger**: The HostedCluster status changes

**Outcome**: The operator reflects the current HostedCluster status in the DPFHCPBridge status field

This provides users with real-time visibility into the state of the hosted control plane through the DPFHCPBridge CR, creating a unified interface for monitoring.

### Responsibility 4: Downloading HostedCluster Kubeconfig and Injecting into DPUCluster

**Trigger**: The HostedCluster becomes ready

**Outcome**: The operator downloads the HostedCluster's kubeconfig, creates a Secret in the DPUCluster's namespace containing the kubeconfig, and adds a reference to this Secret in the DPUCluster CR's status field

This automated kubeconfig distribution enables seamless connectivity between the DPU infrastructure and the hosted control plane, eliminating manual configuration steps.

## Success Criteria

**How users know the operator works correctly**:
- DPFHCPBridge CR transitions to Ready state after successful HostedCluster creation and kubeconfig injection
- HostedCluster resources are successfully created in the management cluster and reach Ready state
- The DPUCluster CR includes the kubeconfig reference in its status field
- Users can retrieve the hosted cluster kubeconfig from the DPUCluster resource
- Users can successfully connect to the hosted cluster using the retrieved kubeconfig

## User Workflows

### Workflow 1: Create, Update, and Delete DPFHCPBridge CR

**Description**: Users manage the complete lifecycle of a DPU-HostedCluster bridge through standard Kubernetes CR operations.

1. User creates a DPFHCPBridge CR with desired configuration (DPUCluster reference, HostedCluster settings)
2. Operator validates the DPUCluster exists, creates the HostedCluster, and injects the kubeconfig
3. User can update the CR to modify configuration (e.g., resource limits, networking settings)
4. Operator applies the updates to the HostedCluster
5. User can delete the CR to clean up all resources
6. Operator removes the HostedCluster and cleans up the kubeconfig reference from the DPUCluster

**Expected outcome**: Complete lifecycle management of HostedClusters through declarative Kubernetes manifests

### Workflow 2: Upgrade OCP Version via DPFHCPBridge CR

**Description**: Users upgrade the OpenShift version of a hosted cluster by updating a single field in the DPFHCPBridge CR.

1. User updates the ocpReleaseImage field in the DPFHCPBridge CR to specify a new OCP version
2. Operator detects the change and updates the corresponding HostedCluster with the new release image
3. HostedCluster performs the upgrade process
4. HostedCluster returns to Ready state after successful upgrade
5. DPFHCPBridge status reflects the updated version

**Expected outcome**: Seamless OpenShift version upgrades through declarative configuration updates, without manual intervention

### Workflow 3: Troubleshoot Failed DPFHCPBridge CR

**Description**: Users diagnose and resolve issues with DPFHCPBridge resources using standard Kubernetes troubleshooting practices.

1. User notices a DPFHCPBridge CR is not reaching Ready state or has degraded
2. User examines the DPFHCPBridge status field for error messages and conditions
3. User reviews the status to identify the root cause (e.g., DPUCluster not found, HostedCluster creation failed, kubeconfig injection failed)
4. User takes corrective actions based on the error information (e.g., fix DPUCluster reference, adjust configuration)
5. Operator reconciles the changes and attempts to resolve the issue

**Expected outcome**: Clear, actionable error messages in the status field enable users to quickly diagnose and resolve configuration or operational issues

## Deployment

**Deployment Method**: Helm charts

## Open Questions

No open questions identified during the interview.
