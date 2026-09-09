### Limitations of the Default Pod Network

In a standard Red Hat OpenShift cluster that uses the default OVN-Kubernetes network plug-in, the system assigns IP addresses to all pods from a single shared overlay network. 
This network typically uses the 10.128.0.0/14 CIDR range or a similar range. The network spans the entire cluster. The OVN-Kubernetes network plug-in enforces this behavior by managing a single, 
cluster-wide logical topology. This topology consists of a centralized logical router and a set of interconnected per-node logical switches, so every pod sits on the same shared Layer 3 network segment.


### Tenants do not receive Layer 3 isolation.


Pods in different namespaces or projects communicate directly unless a network policy blocks them. Tenants do not share a hard Layer 3 boundary.


### The design does not support overlapping IP address subnets.

Different tenants cannot use the same private IP address subnet because IP addresses must remain unique across the cluster.


### The design applies a single topology to all workloads.

All pods have the same overlay characteristics, such as MTUs, encapsulation, and routing behavior, which can conflict with requirements for VMs, legacy applications, or specific performance needs.


### Pod IP addresses do not persist across node changes.

Live migration of virtual machines presents challenges because the pod IP address is tied to the node's local subnet.


### The design limits multinetwork flexibility.

Historically, operators used the more complex network attachment definitions (NADs) for secondary networks. These NADs required additional operator management, and offered limited status reporting.


### Real-world Use Cases

Multitenant isolation:

Multinamespace connectivity:

Overlapping IP address ranges:

Persistent IP addresses for virtual machines:

### Network Topologies for User-defined Networks

Cluster administrators can configure primary networks as layer 2 or layer 3 network types by using UDNs. This flexibility
enables diverse network topologies and use cases, from simple flat networks to more complex segmented
architectures. Layer 2 UDNs provide a flat network topology where all pods are on the same broadcast domain. Layer 3
UDNs create isolated routing domains for each network, to enable overlapping IP address spaces and enhanced
security through routing policies.

**Note:** Although Layer 3 UDN networks perform seamlessly for containerized applications, the implementation is not
ideal for VM-based applications. When deploying VMs within a UDN, Red Hat recommends configuring the
primary network as a Layer 2 network to ensure proper routing, addressing, and isolation. By using a Layer 3
network for VMs, connectivity issues can occur, especially when migrating a VM between hosts, because of
how OVN-Kubernetes handles network addresses.

### Primary Network
Primary networks handle default traffic for pods. When pods use primary UDNs, the primary interfaces, which are
commonly designated as eth0, connect directly to those networks. The default gateway for the pod points to
the UDN, not to the default cluster network.

### Secondary Network
Secondary networks provide additional interfaces to pods. This configuration is useful when a pod needs to
connect to a specific isolated network and to maintain connectivity to default cluster services on the primary
network interface. Secondary UDNs are typically used for specialized workloads that require access to specific
network segments, such as databases or applications with strict security requirements.

## UDN layer3
```
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: my-l3-network
  namespace: my-project
spec:
  topology: Layer3 
  layer3:
    role: Primary 
    subnets:
    - cidr: 10.0.1.0/24 
      hostSubnet: 24 
```

## UDN layer2
```
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: my-l2-network
  namespace: my-project
spec:
  topology: Layer2
  layer2:
    role: Primary
    subnets:
      - "10.200.0.0/16"
    ipam:
      lifecycle: Persistent
```
```
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: my-project
  annotations:
    k8s.v1.cni.cncf.io/networks: my-l3-network 
spec:
  containers:
  - name: my-app-container
    image: my-app-image
```
