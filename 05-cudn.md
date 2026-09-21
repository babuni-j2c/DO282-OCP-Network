## Using a CUDN for Tenant Isolation
CUDN is a custom resource that administrators can use to provide a cluster-scoped UDN for network segmentation
and isolation for workloads in OpenShift. A CUDN resource can join multiple namespaces to the same network. You can
create a CUDN to act as the primary or secondary network for workloads.

In a primary CUDN, the network acts as the primary interface and replaces the default workload network for all
workloads in the assigned namespaces. East-west traffic moves seamlessly across multiple namespaces that use the
same CUDN. This configuration enables pods and VMs to communicate over a private logical switch or a routed subnet,
and maintains a strict "no direct traffic" boundary against other CUDNs. An integrated software-defined router
manages north-south traffic by providing a default gateway within the CUDN. This gateway enables workloads to reach
external networks and to receive incoming data without leaving the isolated tenant environment.

```
apiVersion: v1
kind: Namespace
metadata:
  name: tenant1
  labels:
    k8s.ovn.org/primary-user-defined-network: ""
```

<img width="819" height="506" alt="image" src="https://github.com/user-attachments/assets/8834eed0-395f-4337-bb51-9fb6ac0e423f" />

## Pratical

**Create CUDN**
```
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: development-cudn
spec:
  namespaceSelector:
    matchLabels:
      environment: development
  network:
    topology: Layer2
    layer2:
      role: Primary
      subnets:
      - "10.100.0.0/16"
      ipam:
        lifecycle: Persistent
```
```
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: development-cudn
spec:
  namespaceSelector:
    matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: In
        values:
          - tenant1
          - tenant2
```
**Create Project**
```
apiVersion: v1
kind: Namespace
metadata:
  name: tenant1
  labels:
    environment: development
    k8s.ovn.org/primary-user-defined-network: ""
--
apiVersion: v1
kind: Namespace
metadata:
  name: tenant2
  labels:
    environment: development
    k8s.ovn.org/primary-user-defined-network: ""
```

**Deploy the Application on both project** 

```
$ oc new-app --name test1 -n namespace-1
$ oc new-app --name test2 -n namespace-2
```
**Connection test and validation**
```
$ oc describe po -n namespace-1
$ oc describe po -n namespace-2
# get the IP address
$ oc -n namespace-1 rsh po curl -v telnet://ip:port
```
