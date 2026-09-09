### OVN 
Open Virtual Network

### OVS
Open vSwitch 


### Open Virtual Network

OVN uses OVS as the underlying forwarding engine. OVN represents networking by using a set of logical objects, such as switches, routers, and ports, which are defined declaratively in databases. 
OVN translates these logical objects into OpenFlow rules that are programmed on the local Open vSwitch (OVS) instances on each node. OpenFlow is an open standard protocol that enables a controller 
to remotely program the packet-forwarding behavior of network switches, to separate the control plane from the data plane.

### Benefits of Using OVN

OVN extends OVS by adding a distributed control plane and provides the following features:

Traffic between pods is routed directly on each node, which bypasses the congestion and latency of a single central router.

OVN uses the Geneve overlay encapsulation for node-to-node communication.

Ingress and egress traffic is centralized on the gateway nodes.

Logical L2 switches and L3 routers are modeled declaratively.

OVN provides native support for ACLs that map to Kubernetes network policies.

Load balancing and Network Address Translation (NAT) are implemented through logical flows.

OVN provides DHCP services for IP address assignment.

OVN uses database-based configuration, such as the northbound database (NBDB) and southbound database (SBDB), for consistent state management.


### OVN Logical Constructs

OVN represents networking by using a set of logical objects that abstract the underlying physical infrastructure, to enable administrators to define network topology declaratively.

OVN L2 logical objects
OVN defines networking through the following layer 2 logical objects:

Logical switch (LS)
A logical switch is an L2 broadcast domain.

Logical switch port (LSP)
A logical switch port connects endpoints or routers to logical switches. Its type is set to router for router attachments.

Localnet port
A localnet port is a logical switch port that connects a logical switch to the physical network such as a VLAN or bridge.

OVN L3 logical objects
OVN defines networking through the following layer 3 logical objects:

Logical router (LR)
A logical router is an L3 routing device that provides distributed forwarding for east-west traffic.

Logical router port (LRP)
A logical router port connects logical routers to logical switches and carries IP addresses.

Distributed gateway port
A distributed gateway port is a special logical router port. It handles centralized north-south functions such as NAT and load balancing (LB) on a designated chassis.

Additional constructs include access control lists (ACLs), load balancers, and DHCP options.


### Integrating OVN

OVN-Kubernetes creates the following cluster topology features:

OVN-Kubernetes creates one central distributed cluster router named ovn_cluster_router.

OVN-Kubernetes creates per-node logical switches with individual subnets for local pods.

Pod interfaces connect to the per-node logical switches via LSPs and veth pairs.

A join switch connects the per-node logical switches to the central cluster router.

Distributed gateway ports on the cluster router handle egress NAT and ingress load balancing.

Kubernetes services map to OVN load balancers.

Kubernetes network policies map to OVN ACLs.


### OVN-Kubernetes Components pods

```
$ oc get po -n openshift-ovn-kubernetes
```

#### ovnkube-controller
	
-> OVN-Kubernetes component.

-> Performs IPAM to allocate the pod IP address from the pod subnet of each node.

-> Allocates the pod IP address from the pod subnet of each node.

-> Translates Kubernetes network configuration into OVN structures and stores them in the NBDB

#### nbdb

-> Native OVN component.

-> Stores the logical elements that the ovnkube-controller container created.

#### northd	

-> Native OVN component.

-> Converts the OVN logical elements from NBDB to OVN logical flows.

-> Stores the converted OVN flows in the SBDB.

#### sbdb	

-> Native OVN component.

-> Stores the logical flows that the northd container created.

#### ovn-controller	

-> Native OVN component.

-> Converts the logical flows in SBDB to OpenFlow.

-> Configures OVS to match the intended network state.

#### ovn-acl-logging	

-> Native OVN component.

-> Rotates the OVN log files.

#### kube-rbac-proxy-node	

-> OVN-Kubernetes component.

-> Performs RBAC authentication against the Kubernetes API.

#### kube-rbac-proxy-ovn-metrics	

-> OVN-Kubernetes component.

-> Metrics RBAC for the ovn-controller container.


### OVN-Kubernetes Traffic Types

The OpenShift cluster has two kinds of network traffic, which depend on whether the traffic is internal or external to the cluster. For example, internal network traffic is the communication between two pods, 
or between a pod and a service. The external network traffic occurs for ingress traffic from outside the cluster and for egress traffic when a pod connects to an outside resource.


#### north-south network traffic (Also called vertical traffic)

-> Corresponds to ingress and egress network traffic that goes outside the Kubernetes cluster.

-> The external network bridge (br-ex) handles the traffic.

-> The traffic uses the IP address of the node when communicating to outside networks.

#### east-west network traffic (Also called horizontal traffic)

-> Corresponds to network traffic that is handled within the Kubernetes cluster.

-> pod-to-pod and service to service traffic flow

-> The internal network bridge (br-int) handles the traffic.

-> OVN-Kubernetes uses Geneve tunnels to encapsulate traffic between different cluster nodes.

-> This internal traffic is secured by using Kubernetes network policies.


### OVS external network bridge (br-ex)
### OVS integration network bridge (br-int)
