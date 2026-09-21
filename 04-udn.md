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

### Primary Network (eth0)
Primary networks handle default traffic for pods. When pods use primary UDNs, the primary interfaces, which are
commonly designated as eth0, connect directly to those networks. The default gateway for the pod points to
the UDN, not to the default cluster network.

### Secondary Network (eth0 | udn0)
Secondary networks provide additional interfaces to pods. This configuration is useful when a pod needs to
connect to a specific isolated network and to maintain connectivity to default cluster services on the primary
network interface. Secondary UDNs are typically used for specialized workloads that require access to specific
network segments, such as databases or applications with strict security requirements.

### Layer2 VS Layer3 UDN network
OVN-Kubernetes supports both Layer 2 and Layer 3 primary UDN topologies, but VMs require Layer 2. In a Layer 3
topology, each node receives a distinct subnet, so a VM that migrates to a new node must change its IP address. This IP
address change drops active TCP sessions and breaks live migration. Layer 2 UDNs create a single flat logical switch
across all cluster nodes, which enables VMs to retain their IP addresses during live migration and across reboots.

**Layer 2 UDNs provide the following benefits for VM workloads:**

-> Persistent IP addresses during live migration and across reboots

-> Network isolation between tenants without physical infrastructure changes

-> Overlay networking that works in cloud environments without VLAN access

-> Support for overlapping IP address ranges across different namespaces

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


## Pratical

**01) Create 2 Project with Network label**
```
apiVersion: v1
kind: Namespace
metadata:
  name: namespace-blue
  labels:
    k8s.ovn.org/primary-user-defined-network: ""
---
apiVersion: v1
kind: Namespace
metadata:
  name: namespace-green
  labels:
    k8s.ovn.org/primary-user-defined-network: ""
```

```
$ oc apply -f namespaces.yaml
$ oc get project
$ oc get namespace namespace-blue namespace-green --show-labels
```

**02) UDN-Blue**

```
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: udn-blue
  namespace: namespace-blue
spec:
  topology: Layer3
  layer3:
    role: Primary
    subnets:
    - cidr: 10.100.0.0/16
```

**03) UDN-Green**

```
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: udn-green
  namespace: namespace-green
spec:
  topology: Layer3
  layer3:
    role: Primary
    subnets:
    - cidr: 10.200.0.0/16
```

```
$ oc get userdefinednetwork -n namespace-blue
$ oc get userdefinednetwork -n namespace-blue

$ oc describe userdefinednetwork udn-blue -n namespace-blue
$ oc describe userdefinednetwork udn-blue -n namespace-green
```

**04) Create the Application on both project**

```
$ oc new-app --name blue httpd -n namespace-blue
$ oc new-app --name green httpd -n namespace-green
```
**05) Get the IP address for udn**
```
$ oc describe po -n namespace-blue
$ oc describe po -n namespace-green
```

```
Name: app-blue
Namespace: namespace-blue
...output omitted...
"name": "ovn-kubernetes",
"interface" "eth0"
"ips": [
"10.9.0.31"
...output omitted...
"name": "ovn-kubernetes",
"interface" "ovn-udn1"
"ips": [
"10.100.0.9"
],
...output omitted...
```
```
Name: app-green
Namespace: namespace-green
...output omitted...
"name": "ovn-kubernetes",
"interface" "eth0"
"ips": [
"10.9.0.32"
...output omitted...
"name": "ovn-kubernetes",
"interface" "ovn-udn1"
"ips": [
"10.200.0.4"
],
...output omitted...
```

**06) Test the Network connectivity between Blue and Green. it should not work**

```
BLUE_IP=10.100.0.9
GREEN_IP=10.200.0.4

$ oc exec blue -n namespace-blue -- curl --connect-timeout 5 $GREEN_IP
$ oc exec green -n namespace-green -- curl --connect-timeout 5 $BLUE_IP
```

**07) Ensure that these applications can still reach the cluster API server and other cluster services on the defaultcluster network.**

```
$ oc get svc -n default
$ oc -n namespace-green|blue rsh <po> curl -v telnet://kubernetes.default.svc.cluster.local:443
$  oc -n namespace-green|blue rsh <po> curl -v telnet://api.ns-tes.cloud-lab.j2ctechnologies.intern:6443
```
