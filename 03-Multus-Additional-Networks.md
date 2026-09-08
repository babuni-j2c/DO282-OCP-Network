### Multus Additional Network

complex applications and telecommunications workloads often require network isolation and traffic separation. You might need to separate sensitive data plane traffic from control plane traffic, or require high-performance, low-latency connections that bypass the overlay network. For many enterprise use cases, you must attach a pod to multiple networks to meet specific performance or security requirements. Enterprise customers often require dedicated network interfaces for traffic such as storage replication, database backups, or high-throughput data streaming. In each case, the traffic must remain isolated from the default cluster network.


### RHOCP supports two primary methods for adding secondary networks:

#### Network Attachment Definition
The standard method for creating secondary network interfaces for specific pods.

#### User-Defined Network
A feature that primarily segments the primary network by using different default networks per namespace. UDNs can also serve as secondary networks. The next lesson covers UDNs in detail.


### Additional Network Types

#### macvlan	
Creates a virtual network interface with a unique MAC address that connects directly to a single physical interface. The pod appears as a physical device on the external network. The macvlan network is useful for legacy applications that require direct Layer 2 access.

#### ipvlan	
Similar to macvlan, but the pod shares the MAC address of the host interface, and keeps a distinct IP address. The ipvlan network operates in L2 or L3 modes.

#### bridge	
Connects pods on the same host to a virtual bridge, which can be connected to a physical network. Useful for simple node-local communication.

#### bond	
Aggregates multiple network interfaces into a single logical bonded interface for higher availability and throughput.

#### host-device	
Moves a physical network interface from the host namespace directly to the pod's network namespace.

#### SR-IOV	
Single Root I/O Virtualization enables a physical network card to be sliced into virtual functions (VFs), which are passed directly to pods for high throughput and low latency.

#### tap	
Creates a virtual Ethernet device that acts as a Layer 2 device, which networking applications or virtualization often use.

#### route-override	
Overrides and sets custom routes for a pod's secondary network interface.

#### ovn-k8s-cni-overlay	
Creates a secondary network by using OVN-Kubernetes, so you can use OVN logical switches and Layer 2 topologies as a secondary interface.

### MAD
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: storage-network 
  namespace: project1
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "storage-network", 
    "type": "macvlan", 
    "master": "eth1", 
    "mode": "bridge",
    "ipam": {
        "type": "host-local", 
        "subnet": "192.168.100.0/24"
    }
  }'

```

### Apply to the pod
```
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: storage-network 
spec:
  containers:
  - name: db-container
    image: registry.redhat.io/rhel9/postgresql-15
```
### If application is different Project and NAD is on different project. 
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-in-project2
  namespace: project2
  annotations:
    k8s.v1.cni.cncf.io/networks: project1/storage-network 
spec:
  containers:
  - name: my-container
    image: my-image
```

### Verification

```
$ oc exec -it sample-pod -- ip addr

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    ...output omitted...
    inet 127.0.0.1/8 scope host lo
    ...output omitted...
2: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP group default
    ...output omitted...
    inet 10.128.2.26/23 brd 10.128.3.255 scope global eth0
    ...output omitted...
3: net1@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    ...output omitted...
    inet 192.168.100.5/24 brd 192.168.100.255 scope global net1
    ...output omitted...
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multus-2-pod
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: multus-2-pod
  template:
    metadata:
      labels:
        app: multus-2-pod
      annotations:
        k8s.v1.cni.cncf.io/networks: storage-network  
    spec:
      containers:
      - name: ubi-container
```
