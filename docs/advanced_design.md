---
title: Advanced Design
layout: default
parent: Cilium Designs
nav_order: 2
---

# Advanced Design

The basic network infrastructure for our design will be composed of the following components:

* A tenant: The Kubernetes cluster can be placed in any dedicated, pre-existing between multiple clusters, etc.
* One Floating SVI L3Out where:
  * All the nodes are placed in this L3OUT
  * Dedicated Node Ext EPGs and Service EPGs are used for traffic classification and security with ACI Contracts
  * BGP Peering is established with all or a subset of the nodes for External Service advertisement and LoadBalancing
  * It provides a single L2 Broadcast domain for Node-to-Node Communication

* One Bridge Domain (BD) for the Kubernetes Egress Nodes
  * Create one EPG and Multiples ESG for the Kubernetes Egress Nodes interface. The ESG selector will be the `Egress IP` so that we can map different pod identities to different ACI ESGs.

This basic design gives us the following capabilities:

* Secure the traffic to/from the cluster with ACI contracts.
* DHCP relay support: This design allows the Kubernetes nodes to be bootstrapped without the need to manually configure their IP addresses easing the cluster bootstrap and horizontal scalability. 

  {: .warning } 
  DHCP relay on L3OUT has a few limitations see: [DHCP Limitations](https://www.cisco.com/c/en/us/td/docs/dcn/aci/apic/6x/basic-configuration/cisco-apic-basic-configuration-guide-61x/provisioning-core-aci-fabric-services-61x.html#guidelines-and-limitations-for-a-dhcp-relay-policy)

* The nodes can be of any type and can be mixed: you can have a cluster composed of bare-metal hosts and VMs running on any hypervisor as long as network connectivity is provided.
* Routing simplicity: the node default gateway is the ACI Floating SVI IP.
  * No need to advertise the POD subnet to ACI
* BGP based ECMP for External K8s service Load Balancing with Resilient Hashing
* Near optimal traffic flows thanks to [Direct Server Return](#direct-server-return)

## Cluster L3OUT physical connectivity

There is no strict requirement of the physical connectivity for the cluster EPG as long as it provides the required redundancy level. Most designs are likely to lean toward a vPC based design.

## BGP design
**Centralized BGP peering for service advertisement**

In order to keep the BGP configuration as simple as possible instead of peering with local leaf switches, all of the Kubernetes nodes will peer with one pair of switches (anchor leaf switches). This simplifies the configuration of the physical network and Cilium. At the time of writing ACI 6.1 [supports](https://www.cisco.com/c/en/us/td/docs/dcn/aci/apic/6x/verified-scalability/cisco-aci-verified-scalability-guide-612.html) up to 2000 BGP peers per leaf. It is unlikely that this will pose a scale issue for a single Kubernetes cluster. 
In case Multiple Clusters are running on the same fabric it is easier to spread then over differ anchor nodes as this will not have an impact on the configuration complexity of the cluster. 

![Centralized Routing](../images/centralized-routing.png)
Centralized Routing

### ECMP Considerations

* ACI installs up to 16 eBGP/iBGP ECMP paths. If more than 16 `nodes` are peering via BGP, ACI can be configured to install up to 64 ECMP paths.
* MagLev and DSR requires the `externalTrafficPolicy` set to `Cluster`: This means that that every node that peers with ACI over BGP will advertise itself as a valid next hop for every exposed Service.
* The ECMP selection algorithm will install up to the configured number of ECMP per exposed service. If there are more ECMP paths available ACI will randomly ***(Not sure need to triple check)*** select next-hops with the same Metric. This means that even if not all the nodes can be used for the same service there should still be a good distribution of the traffic load over all the nodes running BGP. Furthermore since DSR is used the response is sent directly back to the client from the pod, bypassing the original node that received the request. 

**Note:** This architecture requires ACI 6.1.2 or above as the Propagate Next Hop and Ignore AM features are both needed 

## Egress Nodes

These nodes will be configured with two interfaces a node interface and an egress one.:
* The node interface will be placed in the floating SVI, this it simplifies node-to-node communication.
  * It is not required for the `egress nodes` to peer over BGP if they are only used for Egress Traffic.
* The egress interface will be connected to an EPG and will be used for the egress gateway feature for POD initiated traffic. 

To ensure route symmetry we can:
* Create a new route-table ID, for example "100"
* In route table 100, add a default route pointing to the Egress BD Address
* Use ip rules so that traffic that is sourced from either of the following
  * the service interface IP address
  * the service IP pool 
  is going to use route table 100, thus ensuring that traffic will be sent back to the L3Out, which preserves routing symmetry.

## Cilium Egress design

When it comes to the Cilium Egress design, the only real decisions to make is how many `egress nodes` to deploy and whether to dedicate them only for this purpose.
Ideally, the design should have a minimum of two `egress nodes` distributed between two pairs of leaves. This will provide redundancy in case of `egress nodes` or ACI leaf failure or during upgrades.
Depending on the cluster scale and application requirements, dedicated `egress nodes` could be beneficial for the same reasons discussed in the [Cilium BGP design](#cilium-bgp-design) section.

## Design trade offs

This design aims to provide you with an easy and high scalable design; however, it comes with the following drawbacks:

1. External services can only be advertised as “Cluster Scope”: This requirement is imposed by MagLev. This drawback is however of minor consequence thanks for Direct Server Return. 
2. Potential bottle necks for egress traffic
3. Node IP is "hidden" behind a L3OUT so there is a small loss of visibility compared to the Simplicity first design.

For issue (1) there is no solution. Issue (2) can be easily addressed with either vertical or horizontal scaling.

[Next](/docs/aci_design/){: .btn }
{: .text-right }