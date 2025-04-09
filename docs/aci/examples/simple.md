---
title: Simple Design Config Example
parent: Config Examples
layout: default
nav_order: 1
---

# Example configuration - Simple Design

This example assumes you are familiar with [Talos Linux](https://www.talos.dev/) and upstream Kubernetes. 

Talos Linux is a minimal and secure operating system tailored for Kubernetes clusters. Instead of a traditional shell or package manager, it uses a declarative configuration file for setup and management. Note that the configuration in this example applies similarly to other distributions.

This cluster will be composed of a mix of x86 and ARM64 nodes

* 3x Control Plane nodes
* 3x dedicated ARM64 worker nodes
* 2x dedicated `ingress nodes`
* 2x dedicated `egress nodes`

The following subnets will be used:
* Node subnet: `192.168.2.0/28`
* Egress Node subnet: `192.168.2.96/28`
* BGP peering subnet: `192.168.2.16/28`
* Load-balancer services pool: `192.168.5.0/28`

![Upstream K8s topology](../images/upstream-K8s-topo.png)

## ACI configuration

* Create a new `cilium-cluster-1` tenant. All of the subsequent config is placed inside this tenant.
* Create a new `nodes` BD.
  * Note: In this example, this BD is mapped to a pre-existing VRF in the common tenant. Alternatively, you can create a VRF within the new tenant.
  * Add the node subnet SVI `192.168.2.14/28` as the primary subnet.
  * Add an additional Egress GW subnet SVI: `192.168.2.110/28` Note: This subnet is optional; you can use the Node Subnet instead. However, using a separate subnet provides a clearer distinction of roles.
* Create a new `k8s` application profile.
* Create a `nodes` EPG and provision connectivity to your `K8s` nodes through VMM, static port binding, or the AAEP (Attachable Access Entity Profile).
* Create a `nodes` ESG and use the `192.168.2.0/28` as IP selector for the ESG.
  * Add any required contracts to allow for communication such as reaching the cluster.
* Create an `App-1` ESG and configure `192.168.2.97/32`and `192.168.2.101/32` as IP selectors for the ESG.  
* Create an `App-2` ESG and configure `192.168.2.98/32`and `192.168.2.102/32` as IP selectors for the ESG. 
  * Note: This setup makes use of two dedicated egress nodes:
    * `egress-cilium-0` is configured with `192.168.2.97/32` and `192.168.2.98/32`
    * `egress-cilium-1` is configured with `192.168.2.101/32` and `192.168.2.102/32`
  * Add contract(s) to provide connectivity for pods egressing the cluster via the Egress IPs.
* Create a new L3Out called `services`. This will be configured using floating SVIs.
  * Create a new `Node Profile` and add the anchor leaf nodes to it. This example makes use of two anchor nodes.
  * Configure a floating interface profile with 2 floating SVI interfaces, one for each leaf node.
  * Add a BGP peer connectivity profile for the `192.168.2.16/28` subnet. Select a BGP remote AS for your cluster. This example makes use of AS number `65111`.
    * Enabled `Bidirectional Forwarding Detection`
    * Repeat this step for all the interface profiles

  * (Optional) Create a “default_svc” external EPG for the load-balancer services pool: `192.168.5.0/28`, and configure this subnet with `External Subnets for External EPG`. This could be useful if you want to have a default set of permit/deny rules for the services that are exposed from your Kubernetes Cluster.
  * Create an `app1_svc` external EPG for the load-balancer services: `192.168.5.2/32`, and configure the subnet with `External Subnets for External EPG`.
  * Create an `app2_svc` external EPG for the load-balancer services: `192.168.5.1/32`, and configure the subnet with `External Subnets for External EPG`.
  
    **Note**: The IP addresses are allocated dynamically by Cilium, even if you are following this example step by step you might need to update these IPs depending on the actual `LoadBalancer` IP allocation. 
  * Add the required `contracts` to `external EPGs` to provide connectivity towards your clients.

* Enable Next Hop Propagate and Ignore IGP Metric (Requires ACI 6.1(2) or newer)

  * Create a `BGP Best Path Policy`
    * Enable `Ignore IGP metric when choosing multipaths`
    * **Note:** This config is applied to the whole VRF and will apply to any other L3Outs in this VRF.
  * Create a `BGP Protocol Profile` by right clicking on the `Logical Node Profile`
    * Select the `BGP Best Path Policy` we just created for the `Bestpath Control Policy`
  * Create a `Match Rule`
    * Create a Match Prefix for the Load-balancer services pool: `192.168.5.0/28` and set `Aggregate` to `True`
  * Create a `Set Rule`
    * Enable `Next Hop Propagation`
  * Create a `Route Maps for Route Control`
    * Create a `Context`
    * Add a `Match Rule` and select the `Match Rule` created previously 
    * Select as `Set Rule` and select the `Set Rule` created previously 
  * Apply the `Route Map` to every `BGP Peer Connectivity Profile` under `Route Control Profile`
* Enable BFD:
  * Create a `BFD Interface Policy` and configure it to match the Cilium Default BFD Timers
    * Detection Multiplier: 3
    * Minimum Transmit Internal: 300
    * Minimum Receive Internal: 300
    * Echo Receive Internal: 50
    * Echo Admin State: Enabled
  * Create a `BFD Interface Profile` under the `Logical Interface Profile` and select the `BFD Interface Policy`

  {: .warning }
  If BFD is configured for sub-second failure detection and you are running virtual machines, a live migration may cause BFD to perceive the node as down. Consequently, the BGP session will be reset, leading to the withdrawal of all routes advertised by the affected node from the routing table.

## Kubernetes

Cilium can be installed on any Kubernetes distribution and its configuration is mostly identical regardless of the distribution of choice. This design has been tested with:
* [Talos Linux](https://www.talos.dev/)

This guide does not include the steps for installing a Kubernetes cluster. You can use any distribution that supports running Cilium.

### Ingress Nodes

The `ingress nodes` needs to be configured with an additional Interface and route table as explained at the beginning of the [Cilium BGP Control Plane](#cilium-bgp-control-plane) section.
To do this in `Talos` linux we can:

* Modify the node `machine config` to add a new interface and address.

```yaml
network:
  interfaces:
    - interface: eth1
      addresses:
        - 192.168.2.19/28
```

* Run the [Cilium Secondary Interface Route Manager](https://github.com/camrossi/cilium-secondary-interface-route-manager) This example daemonset manipulates the route table on the ingress nodes, which is required until this [issue](https://github.com/siderolabs/talos/issues/7184) is addressed. Please be advised that this repository is available as is, and no support is provided. Alternatively it is also possible to manually patch the ingress nodes with the required routes.

### Egress Nodes

The `egress nodes` must be configured with additional addresses on the primary interface. These addresses will be used to source traffic initiated by pods.
Cilium provides a capability to manage these addresses via the `IsovalentEgressGatewayPolicy` CRD however as of Cilium 1.16.6 there is a bug in the `Egress gateway IPAM` and such addresses need to be configured manually on each of the nodes.

To do this in `Talos linux` we can simply add additional `/32 addresses` under the primary interface:

```yaml
network:
  interfaces:
    - interface: eth0
      dhcp: true
      addresses:
        - 192.168.2.97/32
```

We continue to use DHCP on the primary node interfaces and manually add additional addresses for egress functionality. If ACI is set up with a secondary subnet that includes the `/32` addresses, no extra configuration is needed, even if the Egress Subnet is outside the Node Subnet.

## Isovalent Networking for Kubernetes

### Installation

Cilium can be installed with Helm.
Some parameters are specific to the Kubernetes distribution being used, while others are necessary to enable the features utilized in this example.

* Generic Config

```yaml
#Kube Proxy Replacement
kubeProxyReplacement: true
k8sServiceHost: 127.0.0.1
k8sServicePort: 7445

# Direct Routing 
routingMode: native
autoDirectNodeRoutes: true
ipv4NativeRoutingCIDR: "10.56.0.0/16"
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
    - 10.56.0.0/16
    clusterPoolIPv4PodCIDRMaskSize: 24

bpf:
  masquerade: true
nodePort:
  enabled: true
enterprise:
  featureGate:
    approved:
    - EnterpriseBGPControlPlane
    - BFD
    - EgressGatewayHA
    - EgressGatewayIPv4
  bgpControlPlane:
    enabled: true
  bfd:
    enabled: true
  egressGatewayHA:
    enabled: true
hubble:
  relay:
    enabled: true
hubble:
  metrics:
    enabled:
      - "dns:query;ignoreAAAA;labelsContext=source_namespace,source_workload,destination_namespace,destination_workload;sourceContext=workload-name|pod-name|dns|reserved-identity;destinationContext=workload-name|pod-name|dns|reserved-identity"
      - "drop:labelsContext=source_namespace,source_workload,destination_namespace,destination_workload;sourceContext=workload-name|pod-name|dns|reserved-identity;destinationContext=workload-name|pod-name|dns|reserved-identity"
      - "flow:labelsContext=source_namespace,source_workload,destination_namespace,destination_workload;sourceContext=workload-name|pod-name|dns|reserved-identity;destinationContext=workload-name|pod-name|dns|reserved-identity"
      - "flows-to-world:labelsContext=source_namespace,source_workload;sourceContext=workload-name|pod-name|dns|reserved-identity"
      - "httpV2:exemplars=true;labelsContext=source_namespace,source_workload,source_app,destination_namespace,destination_workload,destination_app,traffic_direction;sourceContext=workload-name|pod-name|dns|reserved-identity;destinationContext=workload-name|pod-name|dns|reserved-identity"
      - "icmp:labelsContext=source_namespace,source_workload,destination_namespace,destination_workload;sourceContext=workload-name|pod-name|dns|reserved-identity;destinationContext=workload-name|pod-name|dns|reserved-identity"
      - "kafka:labelsContext=source_namespace,source_workload,source_app,destination_namespace,destination_workload,destination_app,traffic_direction;sourceContext=workload-name|pod-name|dns|reserved-identity;destinationContext=workload-name|pod-name|dns|reserved-identity"
      - "policy:labelsContext=source_namespace,source_workload,destination_namespace,destination_workload;sourceContext=workload-name|pod-name|dns|reserved-identity;destinationContext=workload-name|pod-name|dns|reserved-identity"
      - "tcp:labelsContext=source_namespace,source_workload,destination_namespace,destination_workload;sourceContext=workload-name|pod-name|dns|reserved-identity;destinationContext=workload-name|pod-name|dns|reserved-identity" 
  tls:
    auto:
      enabled: true
      method: helm
  # If you use Hubble UI Enterprise set this to false
  ui:
    enabled: false
```

* Talos Specific

```yaml
cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup
securityContext:
  capabilities:
    ciliumAgent:
    - CHOWN
    - KILL
    - NET_ADMIN
    - NET_RAW
    - IPC_LOCK
    - SYS_ADMIN
    - SYS_RESOURCE
    - DAC_OVERRIDE
    - FOWNER
    - SETGID
    - SETUID
    cleanCiliumState:
    - NET_ADMIN
    - SYS_ADMIN
    - SYS_RESOURCE
  cleanCiliumState:
    - NET_ADMIN
    - SYS_ADMIN
    - SYS_RESOURCE
```

### Configuration - Ingress and BGP

The main goals for this design are the following:

* Ensure that the `ingress nodes` are not running any pods by applying a node taint. Using the `NoExecute` taint will terminate any pods running on the `ingress nodes`, and prevent scheduling. This is important because these nodes may have been provisioned during cluster bootstrap and could already have pods running on them.

```bash
kubectl  taint node <nodes> dedicated=ingress:NoExecute
```

* `ingress-cilium-bgp-0` and `ingress-cilium-bgp-1` are configured to peer with ACI leafs 201 and 202. This can be achieved by labeling the nodes and creating an `IsovalentBGPClusterConfig` that selects nodes based on this label, ensuring each node only peers with Leaf201 and 202.

```bash
kubectl label node <nodes> rack: rack0
```

* Services with an `advertise: bgp` label are going to be advertised. This requirement can be met by creating a `IsovalentBGPAdvertisement` that selects the services based on this label.
* BFD is enabled for Cilium: This requirement can be met by creating a `IsovalentBFDProfile` and referencing it in the `IsovalentBGPPeerConfig`

A complete configuration can be seen here:

```yaml
apiVersion: isovalent.com/v1alpha1
kind: IsovalentBGPClusterConfig
metadata:
  name: cilium-bgp
spec:
  nodeSelector:
    matchLabels:
      rack: rack0
  bgpInstances:
  - name: "instance-65111"
    localASN: 65111
    peers:
    - name: "leaf201"
      peerASN: 65002
      peerAddress: 192.168.2.27
      peerConfigRef:
        name: "cilium-peer"
    - name: "leaf202"
      peerASN: 65002
      peerAddress: 192.168.2.28
      peerConfigRef:
        name: "cilium-peer"
---
apiVersion: isovalent.com/v1alpha1
kind: IsovalentBGPPeerConfig
metadata:
  name: cilium-peer
spec:
  ebgpMultihop: 1
  gracefulRestart:
    enabled: true
    restartTimeSeconds: 120
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: "bgp"
  bfdProfileRef: tor-bfd-profile
---
apiVersion: isovalent.com/v1alpha1
kind: IsovalentBGPAdvertisement
metadata:
  name: bgp-advertisements
  labels:
    advertise: bgp
spec:
  advertisements:
    - advertisementType: "Service"
      service:
        addresses:
          - LoadBalancerIP
      selector:
        matchLabels:
          advertise: bgp
---
apiVersion: isovalent.com/v1alpha1
kind: IsovalentBFDProfile
metadata:
  name: tor-bfd-profile
spec:
  detectMultiplier: 3
  receiveIntervalMilliseconds: 300
  transmitIntervalMilliseconds: 300
  echoFunction:
    directions:
      - Receive
      - Transmit
    receiveIntervalMilliseconds: 50
    transmitIntervalMilliseconds: 50
```

Once this configuration is completed, the BGP peering should be established. You can verify this directly from the Kubernetes cluster using the `cilium` CLI utility or by checking ACI.

```bash
 ~ cilium bgp peers
Node                   Local AS   Peer AS   Peer Address   Session State   Uptime     Family         Received   Advertised
ingress-cilium-bgp-0   65111      65002     192.168.2.27   established     2h1m4s     ipv4/unicast   0          4
                       65111      65002     192.168.2.28   established     1h59m16s   ipv4/unicast   0          4
ingress-cilium-bgp-1   65111      65002     192.168.2.27   established     2h0m46s    ipv4/unicast   0          4
                       65111      65002     192.168.2.28   established     1h59m33s   ipv4/unicast   0          4

  ~ cilium bgp routes
(Defaulting to `available ipv4 unicast` routes, please see help for more options)

Node                   VRouter   Prefix           NextHop   Age     Attrs
ingress-cilium-bgp-0   65111     192.168.5.0/32   0.0.0.0   4m37s   [{Origin: i} {Nexthop: 0.0.0.0}]
                       65111     192.168.5.1/32   0.0.0.0   4m37s   [{Origin: i} {Nexthop: 0.0.0.0}]
                       65111     192.168.5.2/32   0.0.0.0   4m37s   [{Origin: i} {Nexthop: 0.0.0.0}]
ingress-cilium-bgp-1   65111     192.168.5.0/32   0.0.0.0   4m37s   [{Origin: i} {Nexthop: 0.0.0.0}]
                       65111     192.168.5.1/32   0.0.0.0   4m37s   [{Origin: i} {Nexthop: 0.0.0.0}]
                       65111     192.168.5.2/32   0.0.0.0   4m37s   [{Origin: i} {Nexthop: 0.0.0.0}]

fab2-apic1# fabric 201 show ip bgp summary vrf common:default
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.2.19    4 65111     256     240     3602    0    0 01:58:03 3
192.168.2.20    4 65111     255     240     3602    0    0 01:57:45 3

fab2-apic1# fabric 202 show ip bgp summary vrf common:default
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.2.19    4 65111     259     244     4463    0    0 01:59:57 3
192.168.2.20    4 65111     260     245     4463    0    0 02:00:15 3
```

### Configuration - Egress Config

The egress configuration is straightforward. After configuring the nodes with the dedicated egress IP, you can create an `IsovalentEgressGatewayPolicy` that:

* Selects one or more `destinationCIDRs` to enable egress gateway `SNAT` logic. (Any IP belonging to these ranges which is also an internal cluster IP is excluded from the egress gateway SNAT logic.)
  * Optionally select some `excludedCIDRs` 
* Selects wich nodes are part of the `egressGroups` with a `nodeSelector`
* Specify which `egressIP` the node has to use. Note: The `egressIP` needs to be pre-configured on the node. Commonly as a secondary IP on the interface. This should be done prior to deploying the policy.
* Select which pods the `IsovalentEgressGatewayPolicy` should apply to by using a `podSelector`

For example the configuration below will NAT all traffic (`destinationCIDRs`) initiated from pods in the `egress-1` namespace (`io.kubernetes.pod.namespace`) using two nodes (`kubernetes.io/hostname`) and two IPs (`egressIP: 192.168.2.97 and 192.168.2.101`)

```yaml
apiVersion: isovalent.com/v1
kind: IsovalentEgressGatewayPolicy
metadata:
  name: namespace-egress-1
spec:
  destinationCIDRs:
  - 0.0.0.0/0
  egressGroups:
  - nodeSelector:
      matchLabels:
        kubernetes.io/hostname: egress-cilium-bgp-0
    egressIP: 192.168.2.97
  - nodeSelector:
      matchLabels:
        kubernetes.io/hostname: egress-cilium-bgp-1
    egressIP: 192.168.2.101
  selectors:
  - podSelector:
      matchLabels:
        io.kubernetes.pod.namespace: egress-1
```

These two `egressIPs` can now easily be mapped by using an IP selector on the ESG. This allows for administrators to control access per application by using contracts. This example shows access control per namespace, but this can also be narrowed down further to a subset of pods. Pods in the `egress-1` namespace will egress the cluster via the `egressIPs` specified in the `IsoValentEgressGatewayPolicy`.

[Next](/cilium-dc-design/docs/aci/examples/advanced/){: .btn }
{: .text-right }