# Custom Resource Definition: DPFHCPBridge

## Metadata
- **CR Kind**: DPFHCPBridge
- **API Group**: dpf.hcp.bridge.com
- **API Version**: v1alpha1
- **Operator**: DPF HCP Bridge Operator
- **Scope**: Namespaced
- **Created**: 2025-12-21
- **Last Updated**: 2025-12-21 (Operator-SDK Alignment)
- **Status**: Ready for Implementation

## Overview

### Purpose
The DPFHCPBridge Custom Resource represents a Hosted Control Plane (HCP) cluster running on DPU hardware, managed through the HyperShift platform. It enables users to declaratively define and manage OpenShift clusters where the control plane runs on DPUs while worker nodes run on host machines, providing infrastructure separation and improved security posture.

### Relationships
- **References DPUCluster CR**: Each DPFHCPBridge must reference an existing DPUCluster CR (from dpf-operator) which provides the underlying DPU infrastructure. The DPUCluster CR is **namespaced** and can exist in a different namespace (cross-namespace reference supported).
- **Creates HyperShift HostedCluster**: The operator creates and manages a HyperShift HostedCluster CR based on this DPFHCPBridge specification
- **Creates HyperShift NodePool**: The operator creates and manages HyperShift NodePool CR(s) for worker nodes

## API Specification

### Complete Type Definition

```go
package v1alpha1

import (
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// DPFHCPBridge is the Schema for the dpfhcpbridges API
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:scope=Namespaced,shortName=dpfhcp,categories={dpf,hcp}
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`,description="Current phase of the DPFHCPBridge"
// +kubebuilder:printcolumn:name="DPUCluster",type=string,JSONPath=`.spec.dpuClusterRef.name`,description="Referenced DPUCluster name"
// +kubebuilder:printcolumn:name="Ready",type=string,JSONPath=`.status.conditions[?(@.type=="Ready")].status`,description="Ready condition status"
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`,description="Time since creation"
// +kubebuilder:storageversion
// +kubebuilder:validation:XValidation:rule="self.exposeThroughLoadBalancer == false || self.metalLBVirtualIP != ''",message="metalLBVirtualIP is required when exposeThroughLoadBalancer is true"
// +kubebuilder:validation:XValidation:rule="self.exposeThroughLoadBalancer == true || self.metalLBVirtualIP == ''",message="metalLBVirtualIP must be empty when exposeThroughLoadBalancer is false"
type DPFHCPBridge struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   DPFHCPBridgeSpec   `json:"spec,omitempty"`
	Status DPFHCPBridgeStatus `json:"status,omitempty"`
}

// DPFHCPBridgeList contains a list of DPFHCPBridge
// +kubebuilder:object:root=true
type DPFHCPBridgeList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []DPFHCPBridge `json:"items"`
}

// DPFHCPBridgeSpec defines the desired state of DPFHCPBridge
type DPFHCPBridgeSpec struct {
	// DPUClusterRef references the DPUCluster CR that provides the DPU infrastructure
	// for this hosted control plane. The DPUCluster must exist in the cluster before
	// creating this DPFHCPBridge. Supports cross-namespace references.
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:XValidation:rule="self == oldSelf",message="dpuClusterRef is immutable"
	DPUClusterRef ObjectReference `json:"dpuClusterRef"`

	// BaseDomain is the base DNS domain for the hosted cluster. This will be used
	// to construct the cluster's API server endpoint and ingress routes.
	// Example: "clusters.example.com" results in API at "api.my-cluster.clusters.example.com"
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Pattern=`^([a-z0-9]+(-[a-z0-9]+)*\.)+[a-z]{2,}$`
	// +kubebuilder:validation:MinLength=4
	// +kubebuilder:validation:MaxLength=253
	// +kubebuilder:validation:XValidation:rule="self == oldSelf",message="baseDomain is immutable"
	BaseDomain string `json:"baseDomain"`

	// OCPReleaseImage specifies the OpenShift Container Platform release image to use
	// for the hosted cluster. Must be a valid image reference.
	// Example: "quay.io/openshift-release-dev/ocp-release:4.15.0-x86_64"
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Pattern=`^[a-z0-9.\-:/]+$`
	// +kubebuilder:validation:MinLength=10
	// +kubebuilder:validation:MaxLength=512
	OCPReleaseImage string `json:"ocpReleaseImage"`

	// SSHKeySecretRef references a Secret containing the SSH public key(s) for
	// accessing cluster nodes. The Secret must exist in the same namespace.
	// Secret must contain key: "ssh-publickey"
	// +kubebuilder:validation:Required
	SSHKeySecretRef corev1.LocalObjectReference `json:"sshKeySecretRef"`

	// PullSecretRef references a Secret containing the container registry pull secret
	// for pulling OpenShift images. The Secret must exist in the same namespace.
	// Secret must contain key: ".dockerconfigjson" in docker config format
	// +kubebuilder:validation:Required
	PullSecretRef corev1.LocalObjectReference `json:"pullSecretRef"`

	// EtcdStorageClass specifies the StorageClass name to use for etcd persistent volumes.
	// The StorageClass must exist in the management cluster and support dynamic provisioning.
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=253
	EtcdStorageClass string `json:"etcdStorageClass"`

	// ControlPlaneAvailabilityPolicy defines the availability policy for control plane components.
	// Valid values: "HighlyAvailable" (3 replicas) or "SingleReplica" (1 replica)
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Enum=HighlyAvailable;SingleReplica
	ControlPlaneAvailabilityPolicy AvailabilityPolicy `json:"controlPlaneAvailabilityPolicy"`

	// InfrastructureAvailabilityPolicy defines the availability policy for infrastructure components
	// (like ingress controllers). Valid values: "HighlyAvailable" or "SingleReplica"
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Enum=HighlyAvailable;SingleReplica
	InfrastructureAvailabilityPolicy AvailabilityPolicy `json:"infrastructureAvailabilityPolicy"`

	// ExposeThroughLoadBalancer determines whether the hosted cluster API server and ingress
	// should be exposed through a LoadBalancer service (true) or NodePort (false).
	// When true, MetalLBVirtualIP must be provided.
	// +kubebuilder:validation:Required
	ExposeThroughLoadBalancer bool `json:"exposeThroughLoadBalancer"`

	// MetalLBVirtualIP specifies the virtual IP address for MetalLB LoadBalancer services.
	// Required when ExposeThroughLoadBalancer is true, must be empty when false.
	// Must be a valid IPv4 address from the MetalLB IP pool configured in the cluster.
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:Pattern=`^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])$|^$`
	// +kubebuilder:default=""
	MetalLBVirtualIP string `json:"metalLBVirtualIP,omitempty"`

	// ClusterType is a user-defined label to categorize the hosted cluster.
	// Common values: "dpf-production", "dpf-staging", "dpf-dev"
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Pattern=`^[a-z0-9]([-a-z0-9]*[a-z0-9])?$`
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=63
	ClusterType string `json:"clusterType"`
}

// ObjectReference references another Kubernetes object by name and namespace.
// Used for cross-namespace references to DPUCluster CRs.
type ObjectReference struct {
	// Name of the referenced object
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=253
	Name string `json:"name"`

	// Namespace of the referenced object. Supports cross-namespace references.
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=63
	Namespace string `json:"namespace"`
}

// AvailabilityPolicy defines the availability mode for components
// +kubebuilder:validation:Enum=HighlyAvailable;SingleReplica
type AvailabilityPolicy string

const (
	// HighlyAvailable runs 3 replicas for high availability
	HighlyAvailable AvailabilityPolicy = "HighlyAvailable"
	// SingleReplica runs 1 replica for development/testing
	SingleReplica AvailabilityPolicy = "SingleReplica"
)

// DPFHCPBridgeStatus defines the observed state of DPFHCPBridge
type DPFHCPBridgeStatus struct {
	// Phase represents the current lifecycle phase of the DPFHCPBridge
	// +kubebuilder:validation:Enum=Pending;Provisioning;Ready;Failed;Deleting
	// +optional
	Phase DPFHCPBridgePhase `json:"phase,omitempty"`

	// Conditions represent the latest available observations of the DPFHCPBridge's state
	// +optional
	// +listType=map
	// +listMapKey=type
	// +patchMergeKey=type
	// +patchStrategy=merge
	Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type"`

	// HostedClusterRef contains the reference to the created HyperShift HostedCluster
	// +optional
	HostedClusterRef *ObjectReference `json:"hostedClusterRef,omitempty"`

	// NodePoolRef contains the reference to the created HyperShift NodePool
	// +optional
	NodePoolRef *ObjectReference `json:"nodePoolRef,omitempty"`

	// APIEndpoint is the URL of the hosted cluster's Kubernetes API server
	// Example: "https://api.my-cluster.clusters.example.com:6443"
	// +optional
	APIEndpoint string `json:"apiEndpoint,omitempty"`

	// IngressEndpoint is the URL of the hosted cluster's default ingress controller
	// Example: "*.apps.my-cluster.clusters.example.com"
	// +optional
	IngressEndpoint string `json:"ingressEndpoint,omitempty"`

	// KubeconfigSecretRef references the Secret containing the kubeconfig for accessing
	// the hosted cluster. This secret is created by HyperShift.
	// +optional
	KubeconfigSecretRef *corev1.LocalObjectReference `json:"kubeconfigSecretRef,omitempty"`

	// ObservedGeneration reflects the generation of the most recently observed DPFHCPBridge
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`

	// Message provides a human-readable message with details about the current state
	// +optional
	Message string `json:"message,omitempty"`
}

// DPFHCPBridgePhase defines the lifecycle phases
// +kubebuilder:validation:Enum=Pending;Provisioning;Ready;Failed;Deleting
type DPFHCPBridgePhase string

const (
	// Pending indicates the resource has been created but reconciliation hasn't started
	DPFHCPBridgePhasePending DPFHCPBridgePhase = "Pending"
	// Provisioning indicates the hosted cluster is being created
	DPFHCPBridgePhaseProvisioning DPFHCPBridgePhase = "Provisioning"
	// Ready indicates the hosted cluster is fully operational
	DPFHCPBridgePhaseReady DPFHCPBridgePhase = "Ready"
	// Failed indicates the hosted cluster creation or operation has failed
	DPFHCPBridgePhaseFailed DPFHCPBridgePhase = "Failed"
	// Deleting indicates the hosted cluster is being deleted
	DPFHCPBridgePhaseDeleting DPFHCPBridgePhase = "Deleting"
)

func init() {
	SchemeBuilder.Register(&DPFHCPBridge{}, &DPFHCPBridgeList{})
}
```

### GroupVersion Info

```go
// Package v1alpha1 contains API Schema definitions for the dpf.hcp.bridge v1alpha1 API group
// +kubebuilder:object:generate=true
// +groupName=dpf.hcp.bridge.com
package v1alpha1

import (
	"k8s.io/apimachinery/pkg/runtime/schema"
	"sigs.k8s.io/controller-runtime/pkg/scheme"
)

var (
	// GroupVersion is group version used to register these objects
	GroupVersion = schema.GroupVersion{Group: "dpf.hcp.bridge.com", Version: "v1alpha1"}

	// SchemeBuilder is used to add go types to the GroupVersionKind scheme
	SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}

	// AddToScheme adds the types in this group-version to the given scheme.
	AddToScheme = SchemeBuilder.AddToScheme
)
```

## Spec Fields

### Field: `dpuClusterRef`
- **Type:** `ObjectReference` (struct with `name` and `namespace` fields)
- **Required:** Yes
- **Description:** References the DPUCluster CR that provides the underlying DPU infrastructure for this hosted control plane. The DPUCluster must exist before creating this DPFHCPBridge. Supports cross-namespace references (the DPUCluster can be in a different namespace).
- **Validation:**
  - `name`: MinLength=1, MaxLength=253, Required
  - `namespace`: MinLength=1, MaxLength=63, Required
  - The referenced DPUCluster must exist (validated at reconciliation time)
  - CEL validation ensures immutability: `self == oldSelf`
- **Default:** N/A (required)
- **Immutable:** Yes (cannot change DPU cluster after creation)
- **Example:**
  ```yaml
  dpuClusterRef:
    name: prod-dpu-cluster
    namespace: dpf-operator-system
  ```

### Field: `baseDomain`
- **Type:** `string`
- **Required:** Yes
- **Description:** The base DNS domain for the hosted cluster. Used to construct the cluster's API server endpoint (api.<cluster-name>.<baseDomain>) and ingress routes (*.apps.<cluster-name>.<baseDomain>).
- **Validation:**
  - Pattern: `^([a-z0-9]+(-[a-z0-9]+)*\.)+[a-z]{2,}$` (valid DNS domain format)
  - MinLength: 4
  - MaxLength: 253
  - Must be lowercase letters, numbers, hyphens, and dots only
  - CEL validation ensures immutability: `self == oldSelf`
- **Default:** N/A (required)
- **Immutable:** Yes (cannot change domain after cluster creation)
- **Example:** `"clusters.example.com"`

### Field: `ocpReleaseImage`
- **Type:** `string`
- **Required:** Yes
- **Description:** The OpenShift Container Platform release image to use for the hosted cluster. Must be a valid OCI image reference pointing to an OpenShift release.
- **Validation:**
  - Pattern: `^[a-z0-9.\-:/]+$` (valid image reference format)
  - MinLength: 10
  - MaxLength: 512
  - Must be a pullable image (validated at reconciliation time)
- **Default:** N/A (required)
- **Immutable:** No (can be updated to trigger cluster upgrades)
- **Example:** `"quay.io/openshift-release-dev/ocp-release:4.15.0-x86_64"`

### Field: `sshKeySecretRef`
- **Type:** `corev1.LocalObjectReference` (struct with `name` field only)
- **Required:** Yes
- **Description:** References a Secret containing the SSH public key(s) for accessing cluster nodes. The Secret must exist in the same namespace as the DPFHCPBridge and contain the key `ssh-publickey`. Uses Kubernetes standard LocalObjectReference type for same-namespace references.
- **Validation:**
  - `name`: Must be a valid Secret name
  - Secret must exist in same namespace (validated at reconciliation time)
  - Secret must contain key `ssh-publickey` (validated at reconciliation time)
  - Value must be a valid SSH public key format (validated at reconciliation time)
- **Default:** N/A (required)
- **Immutable:** No (can be updated to rotate SSH keys)
- **Example:**
  ```yaml
  sshKeySecretRef:
    name: cluster-ssh-key
  ```

### Field: `pullSecretRef`
- **Type:** `corev1.LocalObjectReference` (struct with `name` field only)
- **Required:** Yes
- **Description:** References a Secret containing the container registry pull secret for pulling OpenShift images. The Secret must exist in the same namespace and be in Docker config JSON format. Uses Kubernetes standard LocalObjectReference type for same-namespace references.
- **Validation:**
  - `name`: Must be a valid Secret name
  - Secret must exist in same namespace (validated at reconciliation time)
  - Secret must contain key `.dockerconfigjson` (validated at reconciliation time)
  - Value must be valid Docker config JSON (validated at reconciliation time)
- **Default:** N/A (required)
- **Immutable:** No (can be updated to change registry credentials)
- **Example:**
  ```yaml
  pullSecretRef:
    name: cluster-pull-secret
  ```

### Field: `etcdStorageClass`
- **Type:** `string`
- **Required:** Yes
- **Description:** The StorageClass name to use for etcd persistent volumes. The StorageClass must exist in the management cluster and support dynamic provisioning with ReadWriteOnce access mode.
- **Validation:**
  - MinLength: 1
  - MaxLength: 253
  - StorageClass must exist in management cluster (validated at reconciliation time)
  - StorageClass must support dynamic provisioning (validated at reconciliation time)
- **Default:** N/A (required)
- **Immutable:** Yes (cannot change storage class after etcd PVCs are created - enforced at reconciliation time by checking status)
- **Example:** `"ceph-rbd-retain"`

### Field: `controlPlaneAvailabilityPolicy`
- **Type:** `AvailabilityPolicy` (enum: `HighlyAvailable` or `SingleReplica`)
- **Required:** Yes
- **Description:** Defines the availability policy for control plane components (API server, controller manager, scheduler). HighlyAvailable runs 3 replicas with anti-affinity rules, SingleReplica runs 1 replica for development/testing.
- **Validation:**
  - Must be exactly `HighlyAvailable` or `SingleReplica` (enum validation)
- **Default:** N/A (required)
- **Immutable:** No (can be updated to scale control plane, triggers reconciliation)
- **Example:** `"HighlyAvailable"`

### Field: `infrastructureAvailabilityPolicy`
- **Type:** `AvailabilityPolicy` (enum: `HighlyAvailable` or `SingleReplica`)
- **Required:** Yes
- **Description:** Defines the availability policy for infrastructure components like ingress controllers and internal DNS. HighlyAvailable runs multiple replicas for redundancy, SingleReplica runs 1 replica.
- **Validation:**
  - Must be exactly `HighlyAvailable` or `SingleReplica` (enum validation)
- **Default:** N/A (required)
- **Immutable:** No (can be updated to scale infrastructure components)
- **Example:** `"HighlyAvailable"`

### Field: `exposeThroughLoadBalancer`
- **Type:** `bool`
- **Required:** Yes
- **Description:** Determines whether the hosted cluster API server and ingress should be exposed through LoadBalancer services (true) or NodePort services (false). When true, requires MetalLB to be configured in the management cluster and metalLBVirtualIP to be specified.
- **Validation:**
  - Boolean value (true/false)
  - CEL cross-field validation ensures metalLBVirtualIP is non-empty when true
  - CEL cross-field validation ensures metalLBVirtualIP is empty when false
- **Default:** N/A (required)
- **Immutable:** No (can be changed to switch exposure method, triggers service recreation)
- **Example:** `true`

### Field: `metalLBVirtualIP`
- **Type:** `string`
- **Required:** Conditionally (required when `exposeThroughLoadBalancer=true`, must be empty when `false`)
- **Description:** The virtual IP address for MetalLB LoadBalancer services. Must be a valid IPv4 address from the MetalLB IP pool configured in the management cluster. Only used when exposeThroughLoadBalancer is true.
- **Validation:**
  - Pattern: `^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])$|^$` (valid IPv4 or empty string)
  - Must be from MetalLB IP pool (validated at reconciliation time)
  - Must not be already in use by another service (validated at reconciliation time)
  - CEL validation enforces cross-field dependency with exposeThroughLoadBalancer
- **Default:** `""` (empty string)
- **Immutable:** No (can be changed to use different VIP)
- **Example:** `"192.168.1.100"` (when LoadBalancer is enabled) or `""` (when NodePort is used)

### Field: `clusterType`
- **Type:** `string`
- **Required:** Yes
- **Description:** A user-defined label to categorize the hosted cluster. Used for organizational purposes, billing, monitoring, or policy enforcement. Common values include environment indicators or purpose labels.
- **Validation:**
  - Pattern: `^[a-z0-9]([-a-z0-9]*[a-z0-9])?$` (valid Kubernetes label value format)
  - MinLength: 1
  - MaxLength: 63
  - Must start and end with alphanumeric, can contain hyphens
- **Default:** N/A (required)
- **Immutable:** No (can be updated for re-categorization)
- **Example:** `"dpf-production"`, `"dpf-staging"`, `"dpf-dev"`

### Nested Types

#### ObjectReference
```go
type ObjectReference struct {
	// Name of the referenced object
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=253
	Name string `json:"name"`

	// Namespace of the referenced object
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=63
	Namespace string `json:"namespace"`
}
```

**Purpose:** Used for cross-namespace references to DPUCluster CRs, which may exist in a different namespace (e.g., `dpf-operator-system`).

**Field: `name`**
- **Type:** `string`
- **Required:** Yes
- **Description:** Name of the referenced Kubernetes object
- **Validation:** MinLength=1, MaxLength=253
- **Example:** `"prod-dpu-cluster"`

**Field: `namespace`**
- **Type:** `string`
- **Required:** Yes
- **Description:** Namespace of the referenced Kubernetes object
- **Validation:** MinLength=1, MaxLength=63
- **Example:** `"dpf-operator-system"`

#### LocalObjectReference (from corev1)
Used for `sshKeySecretRef`, `pullSecretRef`, and `kubeconfigSecretRef` (in status). This is the standard Kubernetes type for same-namespace Secret references.

```go
// From k8s.io/api/core/v1
type LocalObjectReference struct {
	// Name of the referent
	Name string `json:"name,omitempty"`
}
```

### Field Dependencies

**Cross-field validation rules:**

1. **MetalLB VIP Dependency (enforced via CEL):**
   - **If** `exposeThroughLoadBalancer == true` **Then** `metalLBVirtualIP` must be non-empty and a valid IPv4 address
   - **If** `exposeThroughLoadBalancer == false` **Then** `metalLBVirtualIP` must be empty string `""`
   - This is enforced by CEL validation at the DPFHCPBridge type level (no webhook needed for this validation)

2. **Availability Policy Consistency (Recommended):**
   - While not enforced, it's recommended that `controlPlaneAvailabilityPolicy` and `infrastructureAvailabilityPolicy` are set to the same value for consistency
   - Mixed policies (e.g., HighlyAvailable control plane with SingleReplica infrastructure) are allowed but may indicate misconfiguration
   - Controller emits a Warning event if policies differ

**Mutually Exclusive Fields:**
- None (all fields can coexist)

**Immutability Constraints (enforced via CEL):**
- `dpuClusterRef`: Immutable after creation (CEL: `self == oldSelf`)
- `baseDomain`: Immutable after creation (CEL: `self == oldSelf`)
- `etcdStorageClass`: Immutable after etcd PVCs are created (enforced at reconciliation time by checking if HostedCluster exists)

## Status Fields

### Field: `phase`
- **Type:** `DPFHCPBridgePhase` (enum string)
- **Description:** Represents the current lifecycle phase of the DPFHCPBridge. This provides a high-level summary of the cluster state.
- **Populated By:** All reconciliation features update this field based on cluster state
- **Example:** `"Ready"`

**Valid values:**
- `Pending`: Resource created, initial validation in progress
- `Provisioning`: Hosted cluster is being created, resources are being provisioned
- `Ready`: Hosted cluster is fully operational and healthy
- `Failed`: Cluster creation or operation has failed permanently
- `Deleting`: Cluster is being deleted, cleanup in progress

### Field: `conditions`
- **Type:** `[]metav1.Condition` (array of standard Kubernetes conditions)
- **Description:** Detailed condition information about the DPFHCPBridge state. Provides granular insight into different aspects of cluster health and readiness.
- **Populated By:** Various reconciliation features populate specific condition types
- **Example:**
  ```yaml
  conditions:
  - type: Ready
    status: "True"
    reason: ClusterReady
    message: "Hosted cluster is fully operational"
    lastTransitionTime: "2025-12-21T10:30:00Z"
  ```

### Field: `hostedClusterRef`
- **Type:** `*ObjectReference` (pointer to struct)
- **Description:** Reference to the HyperShift HostedCluster CR that was created by this operator. Populated after successful HostedCluster creation.
- **Populated By:** Cluster creation feature during HostedCluster creation
- **Example:**
  ```yaml
  hostedClusterRef:
    name: prod-cluster-hcp
    namespace: clusters
  ```

### Field: `nodePoolRef`
- **Type:** `*ObjectReference` (pointer to struct)
- **Description:** Reference to the HyperShift NodePool CR that manages worker nodes for this hosted cluster. Populated after successful NodePool creation.
- **Populated By:** NodePool creation feature during worker node provisioning
- **Example:**
  ```yaml
  nodePoolRef:
    name: prod-cluster-workers
    namespace: clusters
  ```

### Field: `apiEndpoint`
- **Type:** `string`
- **Description:** The HTTPS URL of the hosted cluster's Kubernetes API server. This is the endpoint clients use to interact with the hosted cluster.
- **Populated By:** API endpoint discovery feature after LoadBalancer/NodePort service is ready
- **Example:** `"https://api.prod-cluster.clusters.example.com:6443"`

### Field: `ingressEndpoint`
- **Type:** `string`
- **Description:** The wildcard DNS pattern for the hosted cluster's default ingress controller. Applications expose routes under this domain.
- **Populated By:** Ingress endpoint discovery feature after ingress controller is ready
- **Example:** `"*.apps.prod-cluster.clusters.example.com"`

### Field: `kubeconfigSecretRef`
- **Type:** `*corev1.LocalObjectReference` (pointer to standard Kubernetes type)
- **Description:** Reference to the Secret containing the kubeconfig file for accessing the hosted cluster. This secret is created by HyperShift and contains admin credentials. The secret is in the same namespace as the DPFHCPBridge.
- **Populated By:** Kubeconfig discovery feature after HyperShift creates the secret
- **Example:**
  ```yaml
  kubeconfigSecretRef:
    name: prod-cluster-admin-kubeconfig
  ```

### Field: `observedGeneration`
- **Type:** `int64`
- **Description:** The generation of the DPFHCPBridge spec that was most recently reconciled. Used to determine if status is up-to-date with spec changes.
- **Populated By:** Every reconciliation loop updates this to match `metadata.generation`
- **Example:** `5`

### Field: `message`
- **Type:** `string`
- **Description:** Human-readable message providing additional context about the current state. Typically contains error messages when phase is Failed, or progress information during Provisioning.
- **Populated By:** All reconciliation features can update this with relevant messages
- **Example:** `"Waiting for control plane pods to become ready (2/3 running)"`

## Status Phases

| Phase | Meaning | Next Phases | Set By |
|-------|---------|-------------|--------|
| `Pending` | CR created, initial validation starting | `Provisioning`, `Failed` | Initial reconciliation / Validation feature |
| `Provisioning` | Resources being created, cluster deploying | `Ready`, `Failed` | Cluster creation feature / HostedCluster reconciliation |
| `Ready` | Hosted cluster fully operational | `Failed`, `Deleting` | Health check feature / Readiness validation |
| `Failed` | Permanent failure occurred | `Provisioning` (after manual fix), `Deleting` | Error handling in any feature |
| `Deleting` | Cluster deletion in progress | (final state, CR removed) | Finalizer cleanup / Deletion reconciliation |

**Phase Transition Rules:**
- `Pending` → `Provisioning`: After successful validation and HostedCluster creation starts
- `Provisioning` → `Ready`: After all components are healthy and endpoints are available
- `Provisioning` → `Failed`: After unrecoverable error or timeout during provisioning
- `Ready` → `Failed`: If cluster becomes unhealthy or critical components fail
- Any → `Deleting`: When CR is marked for deletion (deletionTimestamp set)
- `Failed` → `Provisioning`: After user fixes issue and spec is updated (manual recovery)

## Status Conditions

### Condition: Ready
- **Type:** `Ready`
- **Status:** True/False/Unknown
- **Meanings:**
  - **True**: Hosted cluster is fully operational, API server is accessible, all control plane components are healthy
  - **False**: Hosted cluster is not ready, one or more components are unhealthy or unavailable
  - **Unknown**: Readiness state cannot be determined (e.g., during initial provisioning)
- **Reason Codes:**
  - `ClusterReady`: Cluster is fully operational
  - `WaitingForControlPlane`: Control plane pods are not all running yet
  - `WaitingForInfrastructure`: Infrastructure components (ingress, DNS) not ready
  - `APIServerUnreachable`: Cannot connect to hosted cluster API server
  - `ValidationFailed`: Spec validation failed, cluster cannot be created
- **Set By:** Cluster health monitoring feature, readiness validation feature

**Example:**
```yaml
- type: Ready
  status: "True"
  reason: ClusterReady
  message: "All control plane and infrastructure components are healthy"
  lastTransitionTime: "2025-12-21T10:35:00Z"
  observedGeneration: 1
```

### Condition: HostedClusterCreated
- **Type:** `HostedClusterCreated`
- **Status:** True/False/Unknown
- **Meanings:**
  - **True**: HyperShift HostedCluster CR was successfully created
  - **False**: HostedCluster creation failed or was deleted
  - **Unknown**: HostedCluster creation has not been attempted yet
- **Reason Codes:**
  - `Created`: HostedCluster CR successfully created
  - `CreationFailed`: Failed to create HostedCluster (API error, validation error)
  - `NotCreated`: HostedCluster creation not yet attempted
  - `Deleted`: HostedCluster was deleted (unexpected)
- **Set By:** Cluster creation feature

**Example:**
```yaml
- type: HostedClusterCreated
  status: "True"
  reason: Created
  message: "HostedCluster 'prod-cluster-hcp' created successfully"
  lastTransitionTime: "2025-12-21T10:25:00Z"
  observedGeneration: 1
```

### Condition: NodePoolReady
- **Type:** `NodePoolReady`
- **Status:** True/False/Unknown
- **Meanings:**
  - **True**: NodePool is ready and worker nodes are healthy
  - **False**: NodePool is not ready or worker nodes are unhealthy
  - **Unknown**: NodePool status unknown (creation in progress)
- **Reason Codes:**
  - `NodesReady`: All worker nodes are ready and joined the cluster
  - `WaitingForNodes`: Waiting for worker nodes to provision or join
  - `NodePoolCreationFailed`: Failed to create NodePool CR
  - `NodesUnhealthy`: Some worker nodes are unhealthy or not ready
- **Set By:** NodePool management feature, worker node monitoring feature

**Example:**
```yaml
- type: NodePoolReady
  status: "True"
  reason: NodesReady
  message: "3/3 worker nodes are ready"
  lastTransitionTime: "2025-12-21T10:33:00Z"
  observedGeneration: 1
```

### Condition: DPUClusterValid
- **Type:** `DPUClusterValid`
- **Status:** True/False/Unknown
- **Meanings:**
  - **True**: Referenced DPUCluster exists and is in valid state
  - **False**: DPUCluster doesn't exist, is not ready, or has insufficient resources
  - **Unknown**: DPUCluster validation not yet performed
- **Reason Codes:**
  - `Valid`: DPUCluster is available and ready
  - `NotFound`: Referenced DPUCluster CR does not exist
  - `NotReady`: DPUCluster exists but is not in Ready state
  - `InsufficientResources`: DPUCluster doesn't have enough resources for this hosted cluster
- **Set By:** DPUCluster validation feature (prerequisite checking)

**Example:**
```yaml
- type: DPUClusterValid
  status: "True"
  reason: Valid
  message: "DPUCluster 'prod-dpu-cluster' is ready with sufficient resources"
  lastTransitionTime: "2025-12-21T10:20:00Z"
  observedGeneration: 1
```

### Condition: ServiceExposureReady
- **Type:** `ServiceExposureReady`
- **Status:** True/False/Unknown
- **Meanings:**
  - **True**: API server and ingress services are properly exposed and accessible
  - **False**: Service exposure is not working (LoadBalancer pending, NodePort unreachable)
  - **Unknown**: Service exposure configuration in progress
- **Reason Codes:**
  - `LoadBalancerReady`: LoadBalancer services have external IPs assigned
  - `NodePortReady`: NodePort services are configured and accessible
  - `LoadBalancerPending`: Waiting for LoadBalancer IP assignment (MetalLB)
  - `VIPConflict`: Requested MetalLB VIP is already in use
  - `ServiceCreationFailed`: Failed to create LoadBalancer/NodePort services
- **Set By:** Service exposure feature (LoadBalancer/NodePort management)

**Example:**
```yaml
- type: ServiceExposureReady
  status: "True"
  reason: LoadBalancerReady
  message: "API server exposed at 192.168.1.100:6443, Ingress at 192.168.1.100:443"
  lastTransitionTime: "2025-12-21T10:28:00Z"
  observedGeneration: 1
```

## Validation Rules

### Admission Validation

**CEL validation (declarative, in CRD) performs the following checks automatically on CREATE and UPDATE:**

1. **Spec Field Format Validation:**
   - `baseDomain`: Valid DNS domain format (regex pattern)
   - `ocpReleaseImage`: Valid OCI image reference format
   - `metalLBVirtualIP`: Valid IPv4 format (when not empty)
   - `clusterType`: Valid Kubernetes label value format
   - All string lengths within min/max bounds

2. **Cross-Field Validation (via CEL at DPFHCPBridge level):**
   - If `exposeThroughLoadBalancer == true`, then `metalLBVirtualIP` must be non-empty
   - If `exposeThroughLoadBalancer == false`, then `metalLBVirtualIP` must be empty string
   - `controlPlaneAvailabilityPolicy` and `infrastructureAvailabilityPolicy` must be valid enum values

3. **Immutability Enforcement (via CEL on UPDATE only):**
   - `dpuClusterRef`: CEL rule `self == oldSelf` prevents changes after creation
   - `baseDomain`: CEL rule `self == oldSelf` prevents changes after creation
   - `etcdStorageClass`: Immutability enforced at reconciliation time (checked via status)

4. **Secret Reference Format:**
   - `sshKeySecretRef.name` and `pullSecretRef.name` validated by LocalObjectReference type
   - References implicitly point to same namespace (LocalObjectReference pattern)

**CEL validation does NOT check:**
- Whether referenced resources actually exist (DPUCluster, Secrets, StorageClass)
- Whether MetalLB VIP is from valid pool or already in use
- Whether OCP release image is pullable
- Content/format of SSH keys or pull secrets

**Webhook validation (optional, for complex cases):**
- Can be added for additional validation beyond CEL capabilities
- Recommended for etcdStorageClass immutability check based on HostedCluster existence
- Can emit warning events for policy mismatches

### Reconciliation Validation

**Reconciliation-time validation (in controller logic) performs deeper checks:**

1. **Resource Existence Checks:**
   - DPUCluster CR exists in specified namespace
   - SSH key Secret exists and contains `ssh-publickey` key
   - Pull secret Secret exists and contains `.dockerconfigjson` key
   - StorageClass exists in management cluster
   - MetalLB IPAddressPool exists (if using LoadBalancer)

2. **Resource Content Validation:**
   - SSH key Secret contains valid SSH public key format
   - Pull secret Secret contains valid Docker config JSON
   - StorageClass supports dynamic provisioning and RWO access mode
   - OCP release image is pullable with provided pull secret

3. **Resource State Validation:**
   - DPUCluster is in `Ready` phase
   - DPUCluster has sufficient resources for hosted cluster
   - MetalLB VIP is not already in use by another LoadBalancer service
   - Requested OCP version is compatible with HyperShift version

4. **Dependency Validation:**
   - HyperShift operator is installed and running
   - MetalLB is installed (if exposeThroughLoadBalancer=true)
   - Required CRDs are present (HostedCluster, NodePool)

**Validation failure handling:**
- CEL validation failures: Return error to user, CR not created/updated
- Reconciliation validation failures: Set condition to False, emit event, requeue with backoff

### Cross-Field Validation Examples

**MetalLB VIP and Exposure Method (CEL at type level):**
```go
// +kubebuilder:validation:XValidation:rule="self.exposeThroughLoadBalancer == false || self.metalLBVirtualIP != ''",message="metalLBVirtualIP is required when exposeThroughLoadBalancer is true"
// +kubebuilder:validation:XValidation:rule="self.exposeThroughLoadBalancer == true || self.metalLBVirtualIP == ''",message="metalLBVirtualIP must be empty when exposeThroughLoadBalancer is false"
type DPFHCPBridgeSpec struct {
    // ...
}
```

**Immutability via CEL:**
```go
// +kubebuilder:validation:XValidation:rule="self == oldSelf",message="dpuClusterRef is immutable"
DPUClusterRef ObjectReference `json:"dpuClusterRef"`

// +kubebuilder:validation:XValidation:rule="self == oldSelf",message="baseDomain is immutable"
BaseDomain string `json:"baseDomain"`
```

**Availability Policy Consistency (controller logic - warning only):**
```go
// Emit warning event if policies mismatch (recommended practice)
if dpfhcpBridge.Spec.ControlPlaneAvailabilityPolicy != dpfhcpBridge.Spec.InfrastructureAvailabilityPolicy {
    r.Recorder.Event(dpfhcpBridge, corev1.EventTypeWarning, "MismatchedAvailability",
        "ControlPlane and Infrastructure availability policies differ - this may indicate misconfiguration")
}
```

### What happens if validation fails?

**CEL Validation Failure:**
- CREATE: CR is rejected, not created in cluster, user receives error message from API server
- UPDATE: Update is rejected, CR remains unchanged, user receives error message from API server
- Error message clearly indicates which CEL rule failed and includes the custom message

**Reconciliation Validation Failure:**
- Controller sets `DPUClusterValid` condition to False with appropriate reason
- Controller sets `Ready` condition to False
- Controller sets `phase` to `Failed` (for permanent errors) or `Pending` (for transient errors)
- Kubernetes event is emitted with error details
- Controller requeues with exponential backoff for transient errors (missing resources that may appear)
- Controller does not requeue for permanent errors (user must fix and update spec)

## Finalizer Management

The DPFHCPBridge operator uses a finalizer to ensure proper cleanup of resources when the CR is deleted.

**Finalizer Name:** `dpf.hcp.bridge.com/dpfhcpbridge-finalizer`

**Cleanup Operations:**
1. Delete the HostedCluster CR (Kubernetes garbage collection handles cascading deletion)
2. Delete the NodePool CR
3. Remove kubeconfig reference from DPUCluster status (if present)
4. Wait for all resources to be fully deleted
5. Remove finalizer from DPFHCPBridge

**Implementation Pattern:**
```go
const finalizerName = "dpf.hcp.bridge.com/dpfhcpbridge-finalizer"

func (r *DPFHCPBridgeReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var dpfhcpBridge dpfv1alpha1.DPFHCPBridge
    if err := r.Get(ctx, req.NamespacedName, &dpfhcpBridge); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Handle deletion
    if !dpfhcpBridge.DeletionTimestamp.IsZero() {
        if controllerutil.ContainsFinalizer(&dpfhcpBridge, finalizerName) {
            // Perform cleanup
            if err := r.cleanup(ctx, &dpfhcpBridge); err != nil {
                return ctrl.Result{}, err
            }

            // Remove finalizer
            controllerutil.RemoveFinalizer(&dpfhcpBridge, finalizerName)
            if err := r.Update(ctx, &dpfhcpBridge); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }

    // Add finalizer if not present
    if !controllerutil.ContainsFinalizer(&dpfhcpBridge, finalizerName) {
        controllerutil.AddFinalizer(&dpfhcpBridge, finalizerName)
        if err := r.Update(ctx, &dpfhcpBridge); err != nil {
            return ctrl.Result{}, err
        }
    }

    // ... normal reconciliation logic ...
}
```

## Examples

### Pattern 1: Production HA with LoadBalancer

```yaml
apiVersion: dpf.hcp.bridge.com/v1alpha1
kind: DPFHCPBridge
metadata:
  name: prod-cluster
  namespace: dpf-hcp-bridge-system
spec:
  # Reference to the production DPU cluster (cross-namespace)
  dpuClusterRef:
    name: prod-dpu-cluster
    namespace: dpf-operator-system

  # DNS configuration for production environment
  baseDomain: clusters.example.com

  # OpenShift 4.15 release
  ocpReleaseImage: quay.io/openshift-release-dev/ocp-release:4.15.0-x86_64

  # SSH key for node access (same-namespace)
  sshKeySecretRef:
    name: prod-ssh-key

  # Pull secret for OpenShift images (same-namespace)
  pullSecretRef:
    name: prod-pull-secret

  # Ceph storage for etcd
  etcdStorageClass: ceph-rbd-retain

  # High availability configuration for production
  controlPlaneAvailabilityPolicy: HighlyAvailable
  infrastructureAvailabilityPolicy: HighlyAvailable

  # Expose via MetalLB LoadBalancer for production access
  exposeThroughLoadBalancer: true
  metalLBVirtualIP: "192.168.1.100"

  # Production cluster classification
  clusterType: dpf-production
```

**This pattern is used for:**
- Production workloads requiring high availability
- Clusters that need external LoadBalancer access
- Environments with 3+ DPU nodes available
- SLA-critical applications

**Expected behavior:**
- 3 control plane replicas with pod anti-affinity
- 3 etcd instances for quorum
- Multiple ingress controller replicas
- API server accessible via MetalLB VIP at `https://api.prod-cluster.clusters.example.com:6443`
- Ingress routes accessible at `*.apps.prod-cluster.clusters.example.com`

### Pattern 2: Development/Single Replica with NodePort

```yaml
apiVersion: dpf.hcp.bridge.com/v1alpha1
kind: DPFHCPBridge
metadata:
  name: dev-cluster
  namespace: dpf-hcp-bridge-system
spec:
  # Reference to the development DPU cluster
  dpuClusterRef:
    name: dev-dpu-cluster
    namespace: dpf-operator-system

  # DNS configuration for development environment
  baseDomain: dev.example.com

  # OpenShift 4.15 release (same version for testing parity)
  ocpReleaseImage: quay.io/openshift-release-dev/ocp-release:4.15.0-x86_64

  # SSH key for node access
  sshKeySecretRef:
    name: dev-ssh-key

  # Pull secret for OpenShift images
  pullSecretRef:
    name: dev-pull-secret

  # Standard storage for development (faster provisioning)
  etcdStorageClass: standard

  # Single replica configuration for development
  controlPlaneAvailabilityPolicy: SingleReplica
  infrastructureAvailabilityPolicy: SingleReplica

  # Expose via NodePort for development access (no MetalLB required)
  exposeThroughLoadBalancer: false
  # metalLBVirtualIP omitted (defaults to empty string)

  # Development cluster classification
  clusterType: dpf-dev
```

**This pattern is used for:**
- Development and testing environments
- Resource-constrained clusters (single DPU node)
- Quick iteration and experimentation
- CI/CD test clusters
- Environments without MetalLB

**Expected behavior:**
- 1 control plane replica (single API server, controller manager, scheduler)
- 1 etcd instance (no quorum, faster startup)
- 1 ingress controller replica
- API server accessible via NodePort at `https://<node-ip>:<nodeport>`
- Ingress routes accessible via NodePort
- Faster provisioning time, lower resource usage
- No high availability guarantees

### Pattern 3: Staging with HA Control Plane but Single Infrastructure

```yaml
apiVersion: dpf.hcp.bridge.com/v1alpha1
kind: DPFHCPBridge
metadata:
  name: staging-cluster
  namespace: dpf-hcp-bridge-system
spec:
  dpuClusterRef:
    name: staging-dpu-cluster
    namespace: dpf-operator-system

  baseDomain: staging.example.com
  ocpReleaseImage: quay.io/openshift-release-dev/ocp-release:4.15.0-x86_64

  sshKeySecretRef:
    name: staging-ssh-key

  pullSecretRef:
    name: staging-pull-secret

  etcdStorageClass: ceph-rbd

  # HA control plane for testing HA scenarios
  controlPlaneAvailabilityPolicy: HighlyAvailable

  # Single replica infrastructure to save resources
  infrastructureAvailabilityPolicy: SingleReplica

  # LoadBalancer for production-like access patterns
  exposeThroughLoadBalancer: true
  metalLBVirtualIP: "192.168.1.150"

  clusterType: dpf-staging
```

**This pattern is used for:**
- Pre-production validation environments
- Testing control plane HA without full infrastructure HA
- Cost optimization while maintaining critical HA testing
- Validating upgrade procedures

**Expected behavior:**
- 3 control plane replicas (testing HA failover scenarios)
- 1 ingress controller replica (acceptable for staging)
- LoadBalancer access (production-like networking)
- Lower resource usage than full HA production setup

## Implementation Notes

### Code Structure

```
api/
  v1alpha1/
    dpfhcpbridge_types.go        # CR type definitions (shown in this document)
    dpfhcpbridge_webhook.go       # Admission webhook (optional, for complex validation)
    groupversion_info.go          # API group/version registration
    zz_generated.deepcopy.go      # Generated by controller-gen

config/
  crd/
    bases/
      dpf.hcp.bridge.com_dpfhcpbridges.yaml  # Generated CRD YAML
  rbac/
    role.yaml                     # Generated RBAC rules
  manager/
    manager.yaml                  # Operator deployment
  samples/
    dpf_v1alpha1_dpfhcpbridge.yaml  # Sample CR

internal/controller/
  dpfhcpbridge_controller.go     # Main reconciliation logic
```

### Key Considerations

1. **CRD Generation:**
   - Use `controller-gen` with kubebuilder markers to generate CRD
   - Command: `controller-gen crd:crdVersions=v1 paths=./api/v1alpha1 output:crd:dir=config/crd/bases`
   - CEL validation requires Kubernetes 1.25+
   - Test generated CRD applies cleanly to target Kubernetes versions

2. **CEL Validation (Kubernetes 1.25+):**
   - Cross-field validation uses CEL (no webhook needed)
   - Immutability enforcement uses CEL `self == oldSelf` pattern
   - CEL runs in API server (no webhook infrastructure required)
   - Falls back to webhook validation for Kubernetes <1.25

3. **Status Subresource:**
   - Enabled via `+kubebuilder:subresource:status` marker
   - Use `client.Status().Update()` for status-only updates
   - Implement proper optimistic concurrency control with resourceVersion

4. **Field Indexing:**
   - Add index on `spec.dpuClusterRef.name` for efficient lookups
   - Enables efficient watch filtering for DPUCluster changes
   - See feature spec for implementation details

5. **Finalizers:**
   - Single finalizer: `dpf.hcp.bridge.com/dpfhcpbridge-finalizer`
   - Use `controllerutil` helpers for add/remove/check operations
   - Ensure cleanup is idempotent

6. **Owner References:**
   - Set owner references on created HostedCluster and NodePool
   - Enables Kubernetes garbage collection
   - Use `controllerutil.SetControllerReference()`

7. **Print Columns:**
   - `kubectl get dpfhcp` shows Phase, DPUCluster, Ready status, and Age
   - Improves UX for cluster operators
   - Defined via `+kubebuilder:printcolumn` markers

8. **Categories:**
   - DPFHCPBridge included in `dpf` and `hcp` categories
   - `kubectl get dpf` shows all DPF-related resources
   - `kubectl get hcp` shows all HCP-related resources

9. **Backwards Compatibility:**
   - Plan for API versioning (v1alpha1 → v1beta1 → v1)
   - When adding new fields, use `+optional` marker and provide defaults
   - Document migration path for spec changes
   - Consider conversion webhooks for multi-version support

10. **Testing:**
    - Unit test CEL validation with table-driven tests
    - Test all validation scenarios (valid, invalid cross-fields, immutability)
    - Use envtest for integration testing CR creation/updates
    - Test CRD schema enforcement

11. **Security:**
    - Secrets referenced by name (not embedded)
    - LocalObjectReference for same-namespace Secrets
    - RBAC for DPFHCPBridge creation (namespace-scoped)
    - RBAC for cross-namespace DPUCluster access (cluster-scoped)

12. **Observability:**
    - Emit events for major state transitions
    - Log validation failures
    - Include observedGeneration in all status updates
    - Prometheus metrics for reconciliation

## References

- [HyperShift API Documentation](https://hypershift-docs.netlify.app/)
- [Kubernetes Custom Resource Definitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Kubebuilder CRD Validation](https://book.kubebuilder.io/reference/markers/crd-validation.html)
- [CEL in CRDs (Kubernetes 1.25+)](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation-rules)
- [Kubernetes Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
- [MetalLB Configuration](https://metallb.universe.tf/configuration/)
- [Controller-Runtime Client](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client)
- [ControllerUtil Helpers](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/controller/controllerutil)
- DPF Operator DPUCluster CR specification (internal)

---

**Status:** Ready for implementation (Operator-SDK Aligned)

**Next Steps:**
1. Initialize project with operator-sdk/kubebuilder
2. Generate CRD using controller-gen with all kubebuilder markers
3. Test CEL validation on Kubernetes 1.25+
4. Implement controller with field indexing and proper finalizers
5. Create sample YAML manifests for common use cases
6. Write comprehensive unit and integration tests
7. Design and implement feature workflows
