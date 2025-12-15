# Operator Requirements - Working Notes

This file contains verified information from the product requirements interview.

---

# Category 1: Operator Name
## Status: ✓ VERIFIED by user
- Operator name: dpf-hcp-bridge-operator

# Category 2: Operator Purpose & Problem Statement
## Status: ✓ VERIFIED by user

- Problem:
  - Manual creation and management of HyperShift HostedCluster resources
  - Configuring MetalLB for load balancing
  - Downloading HostedCluster kubeconfig and injecting it into DPUCluster
  - Approving CSRs for worker nodes
  - Deep understanding of HyperShift required

- Manual processes being automated:
  - Creating and managing HyperShift HostedCluster resources
  - Configuring MetalLB for load balancing
  - Downloading HostedCluster kubeconfig and injecting it into DPUCluster
  - Approving CSRs for worker nodes

- Target users: TBD (Open Question)

- Scale:
  - Number of DPFHCPBridge CRs: TBD (Open Question)
  - Deployment: Operator runs on management cluster

# Category 3: Custom Resources Definition
## Status: ✓ VERIFIED by user

## Initial Capture:

### DPFHCPBridge CR

- CR Name: DPFHCPBridge
- Represents: One DPFHCPBridge CR represents one HyperShift HostedCluster that will be created and managed on behalf of a DPUCluster

- Lifecycle stages:
  1. Created: Operator validates DPUCluster exists, creates HostedCluster, configures MetalLB, injects images, creates kubeconfig secret, references it in DPUCluster, enables CSR auto-approval
  2. Ready: HostedCluster ready, MetalLB configured, image injection complete, kubeconfig created and referenced in DPUCluster, CSR auto-approval enabled
  3. Updated: Limited fields can be updated (Infrastructure Availability Mode, ocpReleaseImage, VIP), operator reconciles changes, re-validates BlueField image compatibility
  4. Deleted: HostedCluster deleted, MetalLB cleaned up, kubeconfig secret and reference removed from DPUCluster, CSR auto-approval stopped, finalizers ensure proper cleanup order

- Spec fields (verified):

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

- Relationships:
  - One-to-one relationship with DPUCluster (one DPFHCPBridge references exactly one DPUCluster)

# Category 4: Operator Responsibilities
## Status: ✓ VERIFIED by user

## Initial Capture:

### Responsibility 1: Creating, Updating, and Deleting HostedCluster

**CREATE**:
- Trigger: When a DPFHCPBridge CR is created (after all validations pass and MetalLB is configured)
- Outcome: The operator creates a corresponding HostedCluster CR with configuration derived from the DPFHCPBridge spec fields (clusterName, baseDomain, ocpReleaseImage, sshKeySecretRef, pullSecretRef, etcdStorageClass, controlPlaneAvailabilityPolicy, infrastructureAvailabilityPolicy, exposeThroughLoadBalancer, etc.)

**UPDATE**:
- Trigger: When DPFHCPBridge spec fields change (only infrastructureAvailabilityPolicy or ocpReleaseImage affect HostedCluster; metalLBVirtualIP updates MetalLB configuration, not HostedCluster)
- Validation: For ocpReleaseImage changes, the operator validates the new image against the ocp-bluefield-images ConfigMap
- Safety check: If the HostedCluster is not in ready state, reject the update request
- Outcome: Update the corresponding HostedCluster CR to reflect the new configuration
- Edge cases:
  - If the HostedCluster fails to update and become ready: Reflect the failure in the DPFHCPBridge status and continue retrying

**DELETE**:
- Trigger: User deletes DPFHCPBridge CR
- Full deletion sequence:
  1. User deletes DPFHCPBridge resource
  2. Operator detects deletion (finalizer prevents immediate removal)
  3. Operator deletes HostedCluster
  4. Operator removes MetalLB configuration
  5. Operator stops CSR auto-approval
  6. Operator removes kubeconfig from DPUCluster
  7. Operator removes finalizer
  8. DPFHCPBridge is fully deleted
- Outcome: HostedCluster and all associated resources are cleaned up
- Edge cases:
  - If HostedCluster deletion fails or gets stuck: Reflect the failure in the DPFHCPBridge status and continue retrying
  - Operator waits for HostedCluster to be fully deleted before proceeding with other cleanup

### Responsibility 2: Creating, Updating, and Deleting MetalLB Configuration

**CREATE**:
- Trigger: DPFHCPBridge CR specifies exposeThroughLoadBalancer=true
- Outcome: The operator configures MetalLB with the appropriate resources (IPAddressPool and L2Advertisement) using the metalLBVirtualIP from the DPFHCPBridge spec
- Ordering Constraints: Must happen BEFORE HostedCluster creation
- Success Criteria: MetalLB resources (IPAddressPool and L2Advertisement) exist and are correctly configured

**UPDATE**:
- Trigger: metalLBVirtualIP field in DPFHCPBridge spec is changed
- Outcome: Update MetalLB configuration with the new VIP

**DELETE**:
- Trigger: DPFHCPBridge CR is deleted
- Outcome: Operator removes MetalLB configuration resources (IPAddressPool and L2Advertisement)

**Edge Cases**:
- If MetalLB operator is not installed: Reflect error in DPFHCPBridge status
- If VIP address is invalid or conflicts with existing configuration: Report validation error in DPFHCPBridge status

### Responsibility 3: Downloading HostedCluster Kubeconfig and Injecting into DPUCluster

**Trigger**: HostedCluster becomes ready (HostedCluster status indicates ready state)

**Outcome**:
- Operator downloads the HostedCluster's kubeconfig
- Creates a Secret in the DPUCluster's namespace containing the kubeconfig
- Adds a reference to this Secret in the DPUCluster CR's status field

**Ordering Constraints**:
- Must wait for HostedCluster to be ready before downloading kubeconfig
- Kubeconfig must be available before worker nodes can join

**Success Criteria**:
- Secret exists with the kubeconfig
- DPUCluster CR contains the correct reference to this Secret

**Edge Cases**:
- If kubeconfig download fails: Retry with backoff, reflect error in DPFHCPBridge status
- If Secret creation fails: Retry with backoff, reflect error in DPFHCPBridge status
- If DPUCluster CR update fails: Retry with backoff, reflect error in DPFHCPBridge status
- On DPFHCPBridge deletion: Operator removes the kubeconfig Secret and its reference from DPUCluster

### Responsibility 4: CSR Auto-Approval for DPU Worker Nodes

**Trigger**: HostedCluster is ready and worker nodes start joining

**Outcome**: The operator automatically approves Certificate Signing Requests (CSRs) for worker nodes attempting to join the HostedCluster

**Ordering Constraints**:
- Must start after HostedCluster is ready
- Must continue running as long as DPFHCPBridge exists
- Must stop when DPFHCPBridge is deleted

**Success Criteria**: Worker node CSRs are approved automatically without manual intervention

**Edge Cases**:
- If CSR approval logic encounters an error: Log the error, reflect in DPFHCPBridge status, continue attempting to approve subsequent CSRs
- Only approve CSRs for the specific HostedCluster (not other clusters)
- On DPFHCPBridge deletion: Operator stops auto-approving CSRs for that HostedCluster

### Responsibility 5: Validating DPUCluster Existence

**Trigger**: DPFHCPBridge CR is created

**Outcome**: Before creating the HostedCluster, the operator validates that the referenced DPUCluster CR exists

**Ordering Constraints**: Must happen before any other operations (first validation step)

**Success Criteria**: DPUCluster CR exists and is accessible

**Edge Cases**:
- If DPUCluster does not exist: Report error in DPFHCPBridge status, do not proceed with HostedCluster creation, retry validation periodically
- If DPUCluster exists but is in an unexpected state: Document what "unexpected state" means (Open Question)

### Responsibility 6: BlueField Image Retrieval and Injection

**Trigger**:
- When DPFHCPBridge CR is created
- When ocpReleaseImage field is updated

**Outcome**:
1. Validate that the provided ocpReleaseImage exists in the ocp-bluefield-images ConfigMap (ensuring compatibility with BlueField DPUs)
2. Create a new ConfigMap containing the BlueField container image to override the osImageURL in ignition

**Ordering Constraints**: Must happen before creating or updating the HostedCluster

**Success Criteria**:
- The ocpReleaseImage is found in the ocp-bluefield-images ConfigMap
- New ConfigMap is created with BlueField container image for ignition override

**Edge Cases**:
- If ocpReleaseImage is not in the ConfigMap: Reject the create/update request, report validation error in DPFHCPBridge status
- If ocp-bluefield-images ConfigMap is missing or malformed: Report error in DPFHCPBridge status, prevent HostedCluster creation/update
- If new ConfigMap creation fails: Retry with backoff, reflect error in DPFHCPBridge status

# Category 6: Prerequisites & Deployment
## Status: ✓ VERIFIED by user

## Initial Capture:

**Prerequisites**:
1. Helm - Package manager for Kubernetes, used to provision the DPF-HCP Bridge Operator
2. MGMT Cluster - OpenShift cluster up and running
3. MCE Operator - deployed on the MGMT cluster to deliver and manage Hypershift Operator
4. DPF Operator - deployed on the MGMT cluster
5. Hypershift Operator - deployed on the MGMT cluster (provides HostedCluster CRD)
6. ODF Operator - The Hosted Control Plane's etcd requires persistent storage. ODF must be installed on the MGMT cluster.
7. MetalLB Operator - When deploying an HA Hypershift cluster the control plane needs a stable, single VIP that the three control plane replicas can share. MetalLB Operator used to provide this functionality, acting as a software load balancer.

**Note**: The ocp-bluefield-images ConfigMap is also required but was already captured as an external dependency in Category 5.

**Deployment Method**: Helm

# Category 7: Success Criteria
## Status: ✓ VERIFIED by user

**How users know the operator works correctly:**
- DPFHCPBridge in ready state
- HostedCluster resources are successfully created and reach Ready state
- DPUCluster includes the kubeconfig reference
- Worker nodes in the hosted cluster are Ready after CSRs approved successfully
- MetalLB configuration is applied successfully
- Users can retrieve the hosted cluster kubeconfig from the DPUCluster resource
- Users can connect to the hosted cluster using the kubeconfig
- Clear status messages when validation fails

**Metrics/Observability (high-level):**
- Standard controller-runtime metrics

**Key User Workflows:**
1. Create DPFHCPBridge CR
2. Update DPFHCPBridge CR
3. Upgrade OCP version via DPFHCPBridge CR
4. Delete DPFHCPBridge CR
5. Troubleshoot Failed DPFHCPBridge CR
6. View DPFHCPBridge status and health
7. Update secrets (pull secret, SSH key)