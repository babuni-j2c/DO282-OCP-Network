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
