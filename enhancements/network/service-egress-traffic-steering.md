---
title: service-egress-traffic-steering
authors:
  - "@oribon"
reviewers:
  - "@trozet"
  - "@danwinship"
  - "@SchSeba"
  - "@fedepaol"
approvers:
  - "@danwinship"
  - "@trozet"
api-approvers:
  - "@trozet"
creation-date: 2022-07-28
last-updated: 2022-07-28
status: implementable
tracking-link:
  - https://issues.redhat.com/browse/SDN-2682
---

# OVN service egress traffic steering

## Summary

Some external systems that communicate with applications running on the Kubernetes cluster through a LoadBalancer service expect that the source IP of egress traffic originating from the pods backing the service is identical to the destination IP they use to reach them - i.e the LoadBalancer's external IP (the ingress IP). 
In addition, traffic from these pods should be routed egress via a specific interface on the host. This behavior requires that service access from outside the cluster will only work to this designated interface. This is contradictory to how default services work where a service may be accessed via any node interface.

By annotating a LoadBalancer service, users could request that the source IP of egress packets originating from all of the pods that are endpoints of it would be its external IP. They could also specify from which network the pods' traffic should exit from.
This mechanism will be supported only on clusters using OVN-Kubernetes on "Local Gateway Mode".

## Motivation

Telco customers rely on applications that may initiate traffic bidirectionally over a connection, and thus expect the IP address of the client to be the same in both cases. A common use case is Telco applications that exist outside of the cluster and need to communicate with one or more pods inside the cluster using a service.
Therefore they require that when one of these pod applications initiate traffic towards the external application that it also uses the same IP address.

### Goals

- Provide a mechanism for users running OVN-Kubernetes in "Local Gateway Mode" to request that packets originating from pods backing a specified LoadBalancer service will use the service's IP as their source IP and exit through any interface.

### Non-Goals

- Support OVN-Kubernetes "Shared Gateway Mode".

- Support host-networked pods.

- Announcing the service externally (for service ingress traffic) with OVN-Kubernetes - this part should be handled by a LoadBalancer provider (like MetalLB) as explained later.

- Using the service IP for pod to pod traffic.

## Proposal

Only SNATing a pod's IP to the LoadBalancer service external IP that it is backing is problematic, as usually the external IP is exposed via multiple nodes by the LoadBalancer provider. This means we can't just add an SNAT to the regular traffic flow (in LGW mode) of a pod which is:
`pod -> ovn_cluster_router -> node's mgmt port -> host routing table -> host iptables -> exit through an interface` because we don't have a guarantee that the reply will come back to the pod's node (where the traffic originated).
An external host usually has multiple paths to reach the LoadBalancer external IP and could reply to a node that is not the pod's node - in that case, the other node does not have the proper CONNTRACK entries to send the reply back to the pod and the traffic is lost.
For that reason, we need to make sure that all traffic for the service's pods (Ingress/Egress) is handled by a single node so the right CONNTRACK entries are always matched and the traffic is not lost.

The egress part will be handled by OVN-Kubernetes, which will choose a node that acts as the point of ingress/egress, and steer the relevant pods' egress traffic to its mgmt port, by using logical router policies on the ovn_cluster_router.
When that traffic reaches the node's mgmt port it will use its routing table and iptables.
Because of that, it will also take care of adding the necessary routes and iptables rules on the selected node to SNAT traffic exiting from these pods to the service's external IP using the right interface. The node will also be labeled with `egress-service.k8s.ovn.org/<svc-namespace>-<svc-name>: ""`, which can be consumed by a LoadBalancer provider to handle the ingress part.

The ingress part will be handled by a LoadBalancer provider, such as MetalLB, that would need to select the right node (and only it) for announcing the LoadBalancer service (ingress traffic) according to the `egress-service.k8s.ovn.org/<svc-namespace>-<svc-name>: ""` label set by OVN-Kubernetes.
Taking MetalLB as an example for a LoadBalancer provider, the user will need to create their `L2Advertisement` and/or `BGPAdvertisements` with the `nodeSelectors` field pointing to that label. That way only the node holding the label will be used for announcing the LoadBalancer service IP.
It is worth noting that in MetalLB's case, a given LoadBalancer service can be announced by multiple L2 and BGP advertisements, possibly being (even accidently) announced from multiple nodes. For our use-case the user MUST take care to configure their MetalLB resources in a way that the service is announced only by the node holding the label - a full example is detailed in [Usage Example](#Usage-Example).

To achieve these goals, we introduce a new annotation for users to set on LoadBalancer services: `k8s.ovn.org/egress-service`, which can be either empty or contain an `interface` field: `'{"interface":{"cidr":"172.19.0.0/24"}}'` with a CIDR.
By specifying the `interface` field, only a node that has an interface in that CIDR can be selected for the service's traffic (as explained earlier), and the service's pods egress traffic will exit through that interface.
By not specifying the `interface` field any node in the cluster can be chosen to manage the service's traffic, and the traffic will exit according to the node's routing table.

### Implementation Details/Notes/Constraints

A new annotation `k8s.ovn.org/egress-service` is supported for LoadBalancer services.
When `ovnkube-master` detects that a LoadBalancer service has this annotation it will elect a node to act as the point for all of the traffic of that service (ingress/egress). If the annotation contains a valid CIDR in its `interface` field only a node that has an interface in that CIDR can be elected, which is possible by inspecting all of the nodes' `k8s.ovn.org/host-addresses` annotation.
The CIDR has to match at least one of the interfaces in the cluster, otherwise we don't configure anything.
If the `interface` field is not specified any node can be elected.

After choosing a node, it will create a logical router policy on the ovn_cluster_router for all of the endpoints of the service to steer their egress traffic to that node's mgmt port.
We should take extra care with these policies to not break pod to pod traffic and incoming service traffic, thus we should not steer the traffic when the destination is a pod (pod to pod) or an rtoj port (reply to service traffic).

For example when 10.244.0.3 and 10.244.1.6 are the endpoints of the service and the elected node's mgmt port is 10.244.0.2 we expect policies like these to be created:
```none
$ ovn-nbctl lr-policy-list ovn_cluster_router
       ...
       200           ip4.src == 10.244.0.3/32 && ip4.dst != 100.64.0.0/16 && ip4.dst != 10.244.0.0/16       reroute      10.244.0.2
       200           ip4.src == 10.244.1.6/32 && ip4.dst != 100.64.0.0/16 && ip4.dst != 10.244.0.0/16       reroute      10.244.0.2
       ...
```
After that the service will be annotated with `k8s.ovn.org/egress-service-host=<node_name>` and the node labeled with `egress-service.k8s.ovn.org/<svc-namespace>-<svc-name>: ""`.

When `ovnkube-node` detects that a LoadBalancer service has the `k8s.ovn.org/egress-service` annotation and it is running in the node specified in the service's `k8s.ovn.org/egress-service-host` annotation, it will add the relevant SNATs to the host's iptables.
If the `interface` field is specified, it will find the interface associated with the corresponding CIDR and use it to exit the host, otherwise it will rely on the host's routing table.
In order to use the right interface for the service's egress traffic, packets from the relevant pods will be marked in the PREROUTING chain with a `fwmark` value unique to that service.
An ip rule will be created to send packets marked with that value to a custom routing table that its identifying number matches the same value. The custom routing table will have a single route which sends all traffic via the right interface.

For example when the `eth1` interface on ovn-worker belongs to the CIDR specified in the `interface` field, 10.244.0.3 and 10.244.1.6 are the endpoints of the annotated LoadBalancer service whose external IP is 172.19.0.100, we expect to see iptables rules like these in `ovn-worker`:
```none
$ iptables-save
*mangle
-A PREROUTING -j OVN-KUBE-EGRESS-SVC
-A OVN-KUBE-EGRESS-SVC -s 10.244.0.3/32 -j MARK --set-xmark 0x3e8/0xffffffff
-A OVN-KUBE-EGRESS-SVC -s 10.244.1.6/32 -j MARK --set-xmark 0x3e8/0xffffffff
...
*nat
-A POSTROUTING -j OVN-KUBE-EGRESS-SVC
-A OVN-KUBE-EGRESS-SVC -o eth1 -m mark --mark 0x3e8 -j SNAT --to-source 172.19.0.100
```
and the following ip rule and routing table:
```none
$ ip rule list
1000:	from all fwmark 0x3e8 lookup 1000

$ ip route show table 1000
default via 172.19.0.1 dev eth1
```

After this, for a given service with these final annotations:
```none
$ kubectl describe svc some-service
Name:                     some-service
Namespace:                default
Annotations:              k8s.ovn.org/egress-service: {"interface":{"cidr":"172.19.0.0/24"}} (set by user)
                          k8s.ovn.org/egress-service-host: "ovn-worker"                      (set by ovn-k)
Type:                     LoadBalancer
LoadBalancer Ingress:     172.19.0.100
Endpoints:                10.244.0.3:8080,10.244.1.6:8080
```
the egress traffic flow for the pod `10.244.1.6` on `ovn-worker2` towards an external destination (172.19.0.5) will look like:
```none
                     ┌────────────────────┐
                     │                    │
                     │external destination│
                     │    172.19.0.5      │
                     │                    │
                     └───▲────────────────┘
                         │
     5. packet reaches   │                      2. router policy rereoutes it
        the external     │                         to ovn-worker's mgmt port
        destination with │                      ┌──────────────────┐
        src ip:          │                  ┌───┤ovn cluster router│
        172.19.0.100     │                  │   └───────────▲──────┘
                         │                  │               │
                         │                  │               │1. packet to 172.19.0.5
                      ┌──┴───┐        ┌─────▼┐              │   heads to the cluster router
                   ┌──┘ eth1 └──┐  ┌──┘ mgmt └──┐           │   as usual
                   │ 172.19.0.2 │  │ 10.244.0.2 │           │
                   ├─────▲──────┴──┴─────┬──────┤           │   ┌────────────────┐
4. an iptables rule│     │   ovn-worker  │3.    │           │   │  ovn-worker2   │
   that SNATs to   │     │               │      │           │   │                │
   the service's ip│     │               │      │           │   │                │
   is hit          │     │  ┌────────┐   │      │           │   │ ┌────────────┐ │
                   │     │4.│routes +│   │      │           └───┼─┤    pod     │ │
                   │     └──┤iptables◄───┘      │               │ │ 10.244.1.6 │ │
                   │        └────────┘          │               │ └────────────┘ │
                   │                            │               │                │
                   └────────────────────────────┘               └────────────────┘
                3. from the mgmt port it hits ovn-worker's
                   routes and iptables rules. the packets
                   are first marked with an fwmark value
                   and sent accordingly to a custom routing
                   table which designates them to leave via
                   the interface eth1

```
As mentioned earlier, for the opposite direction (ingress/external client initiates) to work properly the LoadBalancer provider needs to announce the service only from `ovn-worker`.

### TODO: Node Selection
TODO - explain how node selection/failover will work.

### Usage Example

With all of the above implemented, a user can follow these steps to create a functioning LoadBalancer service whose endpoints exit the cluster with its IP using MetalLB.

1. Create the IPAddressPool with the desired IP for the service. It makes sense to set `autoAssign: false` so it is not taken by another service - our service will request that pool explicitly. 
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.19.0.100/32
  autoAssign: false
```

2. Create the LoadBalancer service. We create it with 2 annotations:
- `metallb.universe.tf/address-pool` - to explicitly request the IP to be from the `example-pool`.
- `k8s.ovn.org/egress-service` - to request that all of the endpoints of the service exit the cluster with the service's IP. We also provide an `interface` so that the traffic exits from an interface that belongs to its CIDR.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
  namespace: some-namespace
  annotations:
    metallb.universe.tf/address-pool: example-pool
    k8s.ovn.org/egress-service: '{"interface":{"cidr":"172.19.0.0/24"}}'
spec:
  selector:
    app: example
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
  type: LoadBalancer
```

3. Advertise the service from the node in charge of the service's traffic. So far the service is "broken" - it is not reachable from outside the cluster and if the pods try to send traffic outside it would probably not come back as it is SNATed to an IP which is unknown.
We create the advertisements targeting only the node that is in charge of the service's traffic using the `nodeSelectors` field, relying on ovn-k to label the node properly.
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example-l2-adv
  namespace: metallb-system
spec:
  ipAddressPools:
  - example-pool
  nodeSelectors:
  - matchLabels:
      egress-service.k8s.ovn.org/some-namespace-example-service: ""
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: example-l2-adv
  namespace: metallb-system
spec:
  ipAddressPools:
  - example-pool
  nodeSelectors:
  - matchLabels:
      egress-service.k8s.ovn.org/some-namespace-example-service: ""
```
While possible to create more advertisements resources for the `example-pool`, it is the user's responsibility to make sure that the pool is advertised only by advertisements targeting the node holding the `egress-service.k8s.ovn.org/<svc-namespace>-<svc-name>: ""` label - otherwise the traffic of the service will be broken.
### User Stories
As a user of OpenShift, I should be able to have functioning LoadBalancer services whose backing pods exit the cluster with the service's IP from an interface that belongs to a network I specify.
#### Story 1

As a Telco customer who uses SNMP in OpenShift, I want to access pods that I'm managing using a LoadBalancer service. In order to do so, I need these pods to send traps using the same IP as I use for polling them.

### API Extensions
N/A

### Test Plan

- Unit tests coverage.
- E2E coverage by creating a LoadBalancer service with the proper annotations and validating that:
  - pod to pod traffic works.
  - external client to service works.
  - pod to external client works and is SNATed properly.


### Risks and Mitigations

- The solution might be a bit fragile as it relies on the user to configure the external advertisement of the service manually, with certain limitations.

- Using a single node to handle all of the traffic of a given service might be a bottleneck, and we will also need to try electing nodes in a way that spreads handling these kind of services between them. Failover must also be taken care of in case a node handling a service falls.

- As generally a Service's purpose is to serve ingress traffic, we might be missing some corner cases when using it also to shape egress traffic behavior.

- If a pod is an endpoint of multiple LoadBalancer services that request this functionality the behavior of the SNATs is undefined.

By stating all of the limitations to the user and with enough test coverage we can be confident that the feature is behaving properly for the main use-case.

## Alternatives

EgressIP already does some of the stuff described here, such as steering the traffic of multiple pods through a single node and SNATing their traffic to a single IP. However tying it to a service's IP would require some degree of coordination between the service resource and the EgressIP resource (via a controller).

Also, in its current form EgressIP in baremetal clusters supports only IPs that belong to the primary interface's network (br-ex) and does not respect "Local Gateway Mode" in the sense that it doesn't use the host's iptables. To make it work for our use-case we'd have to refactor a lot of its functionalities for this feature alone, which might break/complicate the way users currently use it.

Having said that, the solution proposed here will probably share/reuse some of the code of EgressIP as they have some sort of similarity.
## Design Details


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
