---
title: Advanced Design Config Example
parent: Config Example
layout: default
---

# Example configuration - Advanced Design With OpenShit

This cluster will be composed

* 3x Control Plane nodes
* 3x worker nodes
* 2x egress nodes

The following subnets will be used:
* Node subnet and BGP Peering: `192.168.2.32/28`
* Egress Node subnet: `192.168.2.80/28`
* Load-balancer services pool: `192.168.2.48/28`


![alt text](../images/openshift-topology.png)
## ACI configuration

* Create a new `ocp-cluster-1` tenant. All of the subsequent config is placed inside this tenant.
* Create a new `egress` BD.
  * Note: In this example, this BD is mapped to a pre-existing VRF in the common tenant. You will need to create your own VRF.
  * Add the egress nodes subnet SVI `192.168.2.94/28` as the subnet.
* Create a new `k8s` application profile.
* Create an `egress` EPG and provision connectivity to your `K8s` node through VMM and/or static port binding.
* Create an `App-1` ESG and use the `192.168.2.81/32`and `192.168.2.82/32` IP subnet selector. 
  * Note: In this lab I have 2 egress gateways:
    * `egress-1` is configured with `192.168.2.81/32`
    * `egress-2` is configured with `192.168.2.82/32`
  * Add any required contracts
* Create a new L3Out called `openshift`. This will be configured using floating SVIs.
  * Create a new `Node Profile` and add your anchor nodes to it; in this example. I have 2 anchor nodes
  * Configure a floating interface profile with 2 floating-SVI interfaces, one for each node.
  * Add a BGP peer connectivity profile for the `192.168.2.32/28` subnet. Select a BGP remote AS for your cluster. In this example, we are going to use `65113`.
    * Enabled `Bidirectional Forwarding Detection`
    * Repeat this step for all the interface profiles
  * Create a `nodes` external EPG for the load-balancer services pool: `192.168.2.32/28`, and set this subnet as `External Subnets for External EPG`. This will be used to control who the OpenShift nodes can communicate with.
  * (Optional) Create a `default_svc` external EPG for the load-balancer services pool: `192.168.2.48/28`, and set this subnet as `External Subnets for External EPG`. This could be useful if you want to have a default set of permit/deny rules for the services that are exposed from your Kubernetes Cluster.
  * Create an `app1_svc` external EPG for the load-balancer services: `192.168.2.49/28`, and set this subnet as `External Subnets for External EPG`.
  * Create an `app2_svc` external EPG for the load-balancer services: `192.168.2.50/28`, and set this subnet as `External Subnets for External EPG`.
    * **Note**: The IP addresses are allocated dynamically by Cilium, even if you are following this example step by step you might need to update these IPs depending on the actual `LoadBalancer` IP allocation. 
  * Add the required `contracts` to `external EPGs` to provide connectivity toward your clients.

* Enable Next Hop Propagate and Ignore IGP Metric (Requires ACI 6.1(2) or newer)

  * Create a `BGP Best Path Policy`
    * Enable `Ignore IGP metric when choosing multipaths`
    * **Note:** This config is applied to the whole VRF
  * Create a `BGP Protocol Profile` by right clicking on the `Logical Node Profile`
    * Select the `BGP Best Path Policy` we just created for the `Bestpath Control Policy`
  * Create a `Match Rule`
    * Create a Match Prefix for the Load-balancer services pool: `192.168.2.48/28` and set `Aggregate` to `True`
  * Create a `Set Rule`
    * Enable `Next Hop Propagation`
  * Create a `Route Maps for Route Control`
    * Create a `Context`
    * Add a `Match Rule` and select the `Match Rule` created previously 
    * Select as `Set Rule` and select the `Set Rule` created previously 
  * Apply the `Route Map` to every `BGP Peer Connectivity Profile` under `Route Control Profile`
* Enable BFD:
  * Create a `BFD Interface Policy` and configure it to match the Cilium Default Timers
    * Detection Multiplier: 3
    * Minimum Transmit Internal: 300
    * Minimum Receive Internal: 300
    * Echo Receive Internal: 50
    * Echo Admin State: Enabled
  * Create a `BFD Interface Profile` under the `Logical Interface Profile` and select the `BFD Interface Policy`

## Cilium Enterprise

### Installation

Cilium can be installed with the [Cilium OLM](https://github.com/isovalent/olm-for-cilium) Operator and its config can be tuned by modifying the `cluster-network-07-cilium-ciliumconfig.yaml` file as show below. You wil need to customize this config to match your subnets, openshift domain name etc. This config is provided just as an example.

```yaml
apiVersion: cilium.io/v1alpha1
kind: CiliumConfig
metadata:
  name: cilium-enterprise
  namespace: cilium
spec:
  cilium:
    operator:
      prometheus:
        enabled: true
        serviceMonitor:
          enabled: true
    prometheus:
      enabled: true
      serviceMonitor:
        enabled: true
    securityContext:
      privileged: true
    ipam:
      mode: "cluster-pool"
      operator:
        clusterPoolIPv4PodCIDRList: ["10.56.0.0/16"]
        clusterPoolIPv4PodCIDRMaskSize: 24
    routingMode: native
    autoDirectNodeRoutes: true
    endpointRoutes: 
      enabled: true
    ipv4NativeRoutingCIDR: "10.56.0.0/16"
    bpf:
      masquerade: true
    nodePort:
      enabled: true
    loadBalancer:
      algorithm: maglev
      mode: dsr
      acceleration: native
    cni:
      binPath: "/var/lib/cni/bin"
      confPath: "/var/run/multus/cni/net.d"
      exclusive: false
    hubble:
      enabled: true
      metrics:
        enabled:
          - dns:labelsContext=source_namespace,destination_namespace
          - drop
          - tcp
          - flow
          - icmp
          - http
          - policy:sourceContext=app|workload-name|pod|reserved-identity;destinationContext=app|workload-name|pod|dns|reserved-identity;labelsContext=source_namespace,destination_namespace
        serviceMonitor:
          enabled: true
        dashboards:
          enabled: true
          annotations:
            grafana_folder: "Hubble"
      relay:
        enabled: true
      tls:
        auto:
          enabled: true
          method: helm
      # We use Hubble UI Enterprise
      ui:
        enabled: false
    enterprise:
      featureGate:
        approved:
        - EnterpriseBGPControlPlane
        - BFD
        - EgressGatewayHA
        - EgressGatewayIPv4
      bgpControlPlane:
        enabled: true
        secretsNamespace:
          create: false
          name: cilium
      bfd:
        enabled: true
      egressGatewayHA:
        enabled: true
    egressGateway:
      enabled: true
    k8sServiceHost: api-int.ocp-cilium-c1.cam.ciscolabs.com
    k8sServicePort: 6443
    kubeProxyReplacement: true
    devices: [enp+]
```

### BGP Config

The main goals for this design are the following:

* All OpenShift nodes will peer with a Pair of ACI Border Leaves. This is achieved by simply not using a `nodeSelector` in the `IsovalentBGPClusterConfig`
* Services with an `advertise: bgp` label are going to be advertised. This requirement can be met by creating a `IsovalentBGPAdvertisement` that selects the services based on this label.
* BFD is enabled for Cilium: This requirement can be met by creating a `IsovalentBFDProfile` and referencing it in the `IsovalentBGPPeerConfig`

A complete configuration can be seen here:

```yaml
apiVersion: isovalent.com/v1alpha1
kind: IsovalentBGPClusterConfig
metadata:
  name: cilium-bgp
spec:
  bgpInstances:
  - name: "instance-65113"
    localASN: 65113
    peers:
    - name: "leaf201"
      peerASN: 65002
      peerAddress: 192.168.2.43
      peerConfigRef:
        name: "cilium-peer"
    - name: "leaf202"
      peerASN: 65002
      peerAddress: 192.168.2.44
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
    restartTimeSeconds: 15
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

You can apply this config after the cluster is installed or at cluster installation time by creating an additional manifest file in the `openshift` folder to be consumed by the `openshift-installer`.
Once this configuration is applied the BGP Peering should be established. 

You can check directly from the OpenShift cluster by using the `cilium` cli utility or from ACI:

```bash
~ cilium bgp peers
Node       Local AS   Peer AS   Peer Address   Session State   Uptime      Family         Received   Advertised
egress-1   65113      65002     192.168.2.43   established     74h9m14s    ipv4/unicast   1          4
           65113      65002     192.168.2.44   established     74h7m31s    ipv4/unicast   1          4
egress-2   65113      65002     192.168.2.43   established     74h7m47s    ipv4/unicast   1          4
           65113      65002     192.168.2.44   established     74h8m30s    ipv4/unicast   1          4
m1         65113      65002     192.168.2.43   established     74h8m13s    ipv4/unicast   1          4
           65113      65002     192.168.2.44   established     74h10m9s    ipv4/unicast   1          4
m2         65113      65002     192.168.2.43   established     74h8m6s     ipv4/unicast   1          4
           65113      65002     192.168.2.44   established     74h8m48s    ipv4/unicast   1          4
m3         65113      65002     192.168.2.43   established     74h8m43s    ipv4/unicast   1          4
           65113      65002     192.168.2.44   established     74h10m13s   ipv4/unicast   1          4
w1         65113      65002     192.168.2.43   established     74h7m54s    ipv4/unicast   1          4
           65113      65002     192.168.2.44   established     74h9m1s     ipv4/unicast   1          4
w2         65113      65002     192.168.2.43   established     74h8m1s     ipv4/unicast   1          4
           65113      65002     192.168.2.44   established     74h9m12s    ipv4/unicast   1          4
w3         65113      65002     192.168.2.43   established     74h7m30s    ipv4/unicast   1          4
           65113      65002     192.168.2.44   established     74h10m48s   ipv4/unicast   1          4

~ cilium bgp routes  -n cilium
(Defaulting to `available ipv4 unicast` routes, please see help for more options)

Node       VRouter   Prefix            NextHop   Age         Attrs
egress-1   65113     192.168.2.48/32   0.0.0.0   77h27m44s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.49/32   0.0.0.0   77h27m44s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.50/32   0.0.0.0   1h8m8s      [{Origin: i} {Nexthop: 0.0.0.0}]
egress-2   65113     192.168.2.48/32   0.0.0.0   77h27m59s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.49/32   0.0.0.0   77h27m59s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.50/32   0.0.0.0   1h8m8s      [{Origin: i} {Nexthop: 0.0.0.0}]
m1         65113     192.168.2.48/32   0.0.0.0   77h27m56s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.49/32   0.0.0.0   77h27m56s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.50/32   0.0.0.0   1h8m8s      [{Origin: i} {Nexthop: 0.0.0.0}]
m2         65113     192.168.2.48/32   0.0.0.0   77h27m49s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.49/32   0.0.0.0   77h27m49s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.50/32   0.0.0.0   1h8m8s      [{Origin: i} {Nexthop: 0.0.0.0}]
m3         65113     192.168.2.48/32   0.0.0.0   77h27m59s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.49/32   0.0.0.0   77h27m59s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.50/32   0.0.0.0   1h8m8s      [{Origin: i} {Nexthop: 0.0.0.0}]
w1         65113     192.168.2.48/32   0.0.0.0   77h27m58s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.49/32   0.0.0.0   77h27m58s   [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.50/32   0.0.0.0   1h8m8s      [{Origin: i} {Nexthop: 0.0.0.0}]
w2         65113     192.168.2.48/32   0.0.0.0   77h28m6s    [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.49/32   0.0.0.0   77h28m6s    [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.50/32   0.0.0.0   1h8m7s      [{Origin: i} {Nexthop: 0.0.0.0}]
w3         65113     192.168.2.48/32   0.0.0.0   77h28m6s    [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.49/32   0.0.0.0   77h28m6s    [{Origin: i} {Nexthop: 0.0.0.0}]
           65113     192.168.2.50/32   0.0.0.0   1h8m8s      [{Origin: i} {Nexthop: 0.0.0.0}]

fab2-apic1# fabric 201 show ip bgp summary vrf common:default
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.2.35    4 65113    8911    8902     9704    0    0    3d02h 3
192.168.2.36    4 65113    8911    8902     9704    0    0    3d02h 3
192.168.2.37    4 65113    8912    8903     9704    0    0    3d02h 3
192.168.2.38    4 65113    8910    8901     9704    0    0    3d02h 3
192.168.2.39    4 65113    8913    8903     9704    0    0    3d02h 3
192.168.2.40    4 65113    8909    8900     9704    0    0    3d02h 3
192.168.2.41    4 65113    8913    8904     9704    0    0    3d02h 3
192.168.2.42    4 65113    8910    8901     9704    0    0    3d02h 3

fabric 202 show ip bgp summary vrf common:default 
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.2.35    4 65113    8916    8907    10522    0    0    3d02h 3
192.168.2.36    4 65113    8914    8905    10522    0    0    3d02h 3
192.168.2.37    4 65113    8916    8907    10522    0    0    3d02h 3
192.168.2.38    4 65113    8914    8905    10522    0    0    3d02h 3
192.168.2.39    4 65113    8914    8905    10522    0    0    3d02h 3
192.168.2.40    4 65113    8918    8909    10522    0    0    3d02h 3
192.168.2.41    4 65113    8911    8902    10522    0    0    3d02h 3
192.168.2.42    4 65113    8913    8904    10522    0    0    3d02h 3


# Note that thanks to Next Hop Propagate both the Anchor node 201 and the non anchor node 203 gets the OpenShift nodes IPs as next hops.

# Anchor Node 202
fab2-apic1# fabric 202 show ip route 192.168.2.49/32 vrf common:default  
----------------------------------------------------------------
 Node 202 (Leaf202)
----------------------------------------------------------------
IP Route Table for VRF "common:default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

192.168.2.49/32, ubest/mbest: 8/0
    *via 192.168.2.37%common:default, [20/0], 01:10:02, bgp-65002, external, tag 65113
         recursive next hop: 192.168.2.37/32%common:default
    *via 192.168.2.35%common:default, [20/0], 01:10:02, bgp-65002, external, tag 65113
         recursive next hop: 192.168.2.35/32%common:default
    *via 192.168.2.39%common:default, [20/0], 01:10:02, bgp-65002, external, tag 65113
         recursive next hop: 192.168.2.39/32%common:default
    *via 192.168.2.38%common:default, [20/0], 01:10:02, bgp-65002, external, tag 65113
         recursive next hop: 192.168.2.38/32%common:default
    *via 192.168.2.41%common:default, [20/0], 01:10:02, bgp-65002, external, tag 65113
         recursive next hop: 192.168.2.41/32%common:default
    *via 192.168.2.42%common:default, [20/0], 01:10:02, bgp-65002, external, tag 65113
         recursive next hop: 192.168.2.42/32%common:default
    *via 192.168.2.36%common:default, [20/0], 01:10:02, bgp-65002, external, tag 65113
         recursive next hop: 192.168.2.36/32%common:default
    *via 192.168.2.40%common:default, [20/0], 01:10:02, bgp-65002, external, tag 65113
         recursive next hop: 192.168.2.40/32%common:default

# Non-Anchor Node 203
fab2-apic1# fabric 203 show ip route 192.168.2.49/32 vrf common:default
----------------------------------------------------------------
 Node 203 (Leaf203)
----------------------------------------------------------------
IP Route Table for VRF "common:default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

192.168.2.49/32, ubest/mbest: 8/0
    *via 192.168.2.42%common:default, [200/0], 01:10:12, bgp-65002, internal, tag 65113
         recursive next hop: 192.168.2.42/32%common:default
    *via 192.168.2.36%common:default, [200/0], 01:10:12, bgp-65002, internal, tag 65113
         recursive next hop: 192.168.2.36/32%common:default
    *via 192.168.2.38%common:default, [200/0], 01:10:12, bgp-65002, internal, tag 65113
         recursive next hop: 192.168.2.38/32%common:default
    *via 192.168.2.39%common:default, [200/0], 01:10:12, bgp-65002, internal, tag 65113
         recursive next hop: 192.168.2.39/32%common:default
    *via 192.168.2.41%common:default, [200/0], 01:10:12, bgp-65002, internal, tag 65113
         recursive next hop: 192.168.2.41/32%common:default
    *via 192.168.2.35%common:default, [200/0], 01:10:12, bgp-65002, internal, tag 65113
         recursive next hop: 192.168.2.35/32%common:default
    *via 192.168.2.37%common:default, [200/0], 01:10:12, bgp-65002, internal, tag 65113
         recursive next hop: 192.168.2.37/32%common:default
    *via 192.168.2.40%common:default, [200/0], 01:10:12, bgp-65002, internal, tag 65113
         recursive next hop: 192.168.2.40/32%common:default

```

### Egress Config

The egress configuration is extremely simple: once the nodes are configured with the dedicated egress IP one can create an `IsovalentEgressGatewayPolicy` that:

* Select one or more `destinationCIDRs` to enable egress gateway `SNAT` logic. (Any IP belonging to these ranges which is also an internal cluster IP is excluded from the egress gateway SNAT logic.)
  * Optionally select some `excludedCIDRs` 
* Select wich node are part of the `egressGroups` with a `nodeSelector`
* Specify which `egressIP` a node has to use. Note: The `egressIP` needs to be pre-configured on the node.
* Select which pods are gonna be using this `IsovalentEgressGatewayPolicy` with a `podSelector`

For example the configuration below will NAT all traffic (`destinationCIDRs`) initiated from pods in the `egress-1` namespace (`io.kubernetes.pod.namespace`) using two nodes (`kubernetes.io/hostname`) and two IPs (`egressIP: 192.168.2.83 and 192.168.2.84`)

```yaml
apiVersion: isovalent.com/v1
kind: IsovalentEgressGatewayPolicy
metadata:
  name: egress-1
spec:
  destinationCIDRs:
  - 0.0.0.0/0
  egressGroups:
  - nodeSelector:
      matchLabels:
        kubernetes.io/hostname: egress-1
    egressIP: 192.168.2.83
  - nodeSelector:
      matchLabels:
        kubernetes.io/hostname: egress-2
    egressIP: 192.168.2.84
  selectors:
  - podSelector:
      matchLabels:
        io.kubernetes.pod.namespace: egress-1

```

It is easy now to add these 2 IP in an ESG selector and use ACI contracts to enforce security policy for all traffic initiated by PODs in the `namespace: egress-1`

[Next](/docs/examples/simple/){: .btn }
{: .text-right }