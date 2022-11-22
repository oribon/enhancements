---
title: service-fwmark
authors:
  - "@oribon"
reviewers:
  - "@trozet"
  - "@fedepaol"
approvers:
  - "@danwinship"
  - "@trozet"
api-approvers:
  - "@trozet"
creation-date: 2022-11-29
last-updated: 2022-11-29
status: implementable
tracking-link:
  - https://issues.redhat.com/browse/CNF-6577
---

# OVN Services FWMark

## Summary

Some cluster admins require elaborate routing policies in their Kubernetes cluster, which include creating custom routing tables and `ip rules` that use them based on the `fwmark` of packets.

By introducing a new CRD `ServiceFWMark`, cluster admins could specify a fwmark value for the traffic of pods belonging to a given service.
This includes both packets originating from pods (egress) and return traffic (as a reply to an external client calling the service).
The CRD will be Namespaced, with each resource name corresponding to a Service in the namespace.
The resources will be watched by the ovnkube nodes, which in turn will configure the iptables rules to achieve that.

## Motivation

Telco customers require support for setting a fwmark for their 5G applications, allowing that traffic to be routed by custom routing tables, with preconfigured `ip rules` that use fwmark to determine the correct routing table.

### Goals

- Provide a mechanism for cluster admins using OVN-Kubernetes in "Local Gateway Mode" to set a fwmark value on the traffic of pods belonging to a specific service.

### Non-Goals

- Manage ip rules and custom routing tables - this feature does not care about what is being done with the fwmark on each node and it is up for the cluster admin to configure their infrastructure to consume the marks (e.g by creating relevant kubernetes-nmstate NodeNetworkConfigurationPolicies).

- Modify anything related to the traffic flow which specifically concerns "Shared Gateway Mode" as the packets will not be marked in most cases by the nature of that gateway mode. It might make sense to have this feature enabled on "Local Gateway Mode" only.

## Proposal

To achieve marking the traffic of the pods of a given service with a fwmark, we introduce a new CRD `ServiceFWMark` which allows a cluster admin to specify a fwmark to be set on all of the packets related to the pods of a specific service - both those originating from the pods (egress) and return traffic (as a reply to an external client calling the service).

### Implementation Details/Notes/Constraints

A new API `ServiceFWMark` under the `k8s.ovn.org/v1` version will be added to `pkg/crd`.

A new controller in OVN-K will watch `ServiceFWMark`, `Services` and `EndpointSlices` objects, creating the necessary iptables rules to set the fwmark on the traffic of the service's pods targeted by the `ServiceFWMark` resource.

The reply traffic of the service will be marked by creating an iptables rule matching the ClusterIP of the service on each node.
The egress traffic of the service's endpoints will be marked by creating an iptables rule matching the endpoint's IP on its node.

When the service is an "Egress Service" its traffic is handled by a single node, thus in that case we will create all of these rules on that node only, as it doesn't make sense to have them on the other nodes.

For example, assuming there is a Service named service1 in namespace default with ClusterIP of 100.100.100.100, 10.244.0.3 (on node1) and 10.244.1.6 (on node2) are endpoints of it, creating the following ServiceFWMark:

```yaml
kind: ServiceFWMark
apiVersion: k8s.ovn.org/v1
metadata:
  name: service1
  namespace: default
spec:
  fwmark: 1000
```
    
will result in the following iptables rules being created on node1:
```none
$ iptables-save
*mangle
-A PREROUTING -j OVN-KUBE-SERVICE-FWMARK
-A OVN-KUBE-SERVICE-FWMARK -s 100.100.100.100 -j MARK --set-mark 1000 -m comment --comment default/service1
-A OVN-KUBE-SERVICE-FWMARK -s 10.244.0.3 -j MARK --set-mark 1000 -m comment --comment default/service1
```

If service1 is an "Egress Service" that its node is "node2" (can be inferred from inspecting the `k8s.ovn.org/egress-service-host` annotation), node1 will have no iptables rules created while the following rules will be created on node2:
```none
$ iptables-save
*mangle
-A PREROUTING -j OVN-KUBE-SERVICE-FWMARK
-A OVN-KUBE-SERVICE-FWMARK -s 100.100.100.100 -j MARK --set-mark 1000 -m comment --comment default/service1
-A OVN-KUBE-SERVICE-FWMARK -s 10.244.0.3 -j MARK --set-mark 1000 -m comment --comment default/service1
-A OVN-KUBE-SERVICE-FWMARK -s 10.244.1.6 -j MARK --set-mark 1000 -m comment --comment default/service1
```

### User Stories

As a user of OpenShift, I should be able to set a fwmark on the traffic of pods of a given service.

#### Story 1

As a Telco customer who wants to use VRFs in OpenShift by creating ip rules that match on fwmark to lookup custom routing tables, I need a way to mark the traffic (egress/reply) of specific services with a valid fwmark in order to ensure that the right ip rules are matched.
There is a [POC](https://github.com/fedepaol/returntraffic/tree/main/withovnk) that describes this requirement in more detail.

### API Extensions

A new namespace-scoped CRD is introduced:

```go
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +kubebuilder:resource:path=servicefwmarks
// +kubebuilder::singular=servicefwmark
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// ServiceFWMark is a CRD that allows the user to define a fwmark value
// for all the traffic of a given service, which means that all traffic
// originating from endpoints of the service and all traffic heading to it
// will be marked with the given fwmark.
type ServiceFWMark struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ServiceFWMarkSpec   `json:"spec,omitempty"`
	Status ServiceFWMarkStatus `json:"status,omitempty"`
}

// ServiceFWMarkSpec defines the desired state of ServiceFWMark
type ServiceFWMarkSpec struct {
	// The fwmark value to set for the service's traffic.
	// +kubebuilder:validation:Maximum:=2000
	// +kubebuilder:validation:Minimum:=1000
	FWMark int `json:"fwmark"`
}
```

### Test Plan

* Unit tests coverage

* E2E ?

## Alternatives

OVN's logical router policies have an option to set a `pkt_mark` which in turn programs the flows to mark packets.
While using OVN's objects is preferred over iptables rules, the mark set by this option is not preserved when the packet goes out from one node to another (across tunnels), which means that this will not work with "Egress Services" since they use logical router policies that reroute to a different node.

With that being said, if OVN does support preserving the mark in "reroute" policies in the future the proposed API will still be valid and we could shift the implementation to leverage logical router policies instead of iptables rules.

### Graduation Criteria


#### Dev Preview -> Tech Preview


#### Tech Preview -> GA


#### Removing a deprecated feature


### Upgrade / Downgrade Strategy


### Version Skew Strategy


### Operational Aspects of API Extensions


#### Failure Modes


#### Support Procedures


## Implementation History


### Drawbacks

### Workflow Description