### Limitations of the Default Pod Network
```
In a standard Red Hat OpenShift cluster that uses the default OVN-Kubernetes network plug-in, the system assigns IP addresses to all pods from a
single shared overlay network. This network typically uses the 10.128.0.0/14 CIDR range or a similar range. The network spans the entire cluster.
The OVN-Kubernetes network plug-in enforces this behavior by managing a single, cluster-wide logical topology. This topology consists of a centralized
logical router and a set of interconnected per-node logical switches, so every pod sits on the same shared Layer 3 network segment.
```

### Tenants do not receive Layer 3 isolation.

```
Pods in different namespaces or projects communicate directly unless a network policy blocks them. Tenants do not share a hard Layer 3 boundary.
```

### The design does not support overlapping IP address subnets.
```
Different tenants cannot use the same private IP address subnet because IP addresses must remain unique across the cluster.
```

### The design applies a single topology to all workloads.
```
All pods have the same overlay characteristics, such as MTUs, encapsulation, and routing behavior, which can conflict with requirements for VMs, legacy applications, or specific performance needs.
```

### Pod IP addresses do not persist across node changes.
```
Live migration of virtual machines presents challenges because the pod IP address is tied to the node's local subnet.
```

### The design limits multinetwork flexibility.
```
Historically, operators used the more complex network attachment definitions (NADs) for secondary networks. These NADs required additional operator management, and offered limited status reporting.
```

### Real-world Use Cases

Multitenant isolation:

Multinamespace connectivity:

Overlapping IP address ranges:

Persistent IP addresses for virtual machines:

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
