---
title: Simplicity Design
layout: default
parent: Cilium Designs
nav_order: 1
---

# Simplicity Design
The basic network infrastructure for our design will be composed of the following components:

* A tenant: The Kubernetes cluster can be placed in any dedicated, pre-existing between multiple clusters, etc.
* One Bridge Domain (BD) for the Kubernetes cluster
  * Configure the bridge domain with a primary subnet in the node subnet. This will be the default gateway for the Kubernetes nodes.
* Create one EPG and ESG for the Kubernetes nodes primary (KubeAPI) interface. The ESG selector will be the `Node Subnet` so that all the nodes will be grouped in the same ESG.
* One Floating SVI L3Out L3Out running BGP for the ingress nodes.

This basic design gives us the following capabilities:

* Secure the traffic initiated by the Kubernetes nodes with ACI contracts.
* DHCP relay support: This design allows the Kubernetes nodes to be bootstrapped without the need to manually configure their IP addresses easing the cluster bootstrap and horizontal scalability. Note: Remember to set the node subnet as “Primary” as DHCP relay packets are forwarded only for this subnet.
  * This is an advantage compared to a design where the nodes are only connected to an L3OUT as DHCP relay on L3OUT has a few limitations see: [DHCP Limitations](https://www.cisco.com/c/en/us/td/docs/dcn/aci/apic/6x/basic-configuration/cisco-apic-basic-configuration-guide-61x/provisioning-core-aci-fabric-services-61x.html#guidelines-and-limitations-for-a-dhcp-relay-policy)
* Visibility of the node IP
* The nodes can be of any type and can be mixed: you can have a cluster composed of bare-metal hosts and VMs running on any hypervisor as long as network connectivity is provided.
* Routing simplicity: the node default gateway is the ACI node subnet SVI IP.
* BGP based ECMP for External K8s service Load Balancing 

## Cluster EPG physical connectivity

There is no strict requirement of the physical connectivity for the cluster EPG as long as it provides the required redundancy level. Most designs are likely to lean toward a vPC based design.

## Selective BGP peering for service advertisement

In this design, a dedicated Cisco ACI L3Out is created for external service advertisement for a subset of Kubernetes nodes that will be, from now on, called `ingress nodes`.

These nodes will be configured with two interfaces. It is convenient to keep the `ingress nodes` in the cluster EPG, because it simplifies node-to-node communication and allows the use of DHCP relay for node provisioning.

![Selective BGP peering design](../images/selective-bgp.png)
Selective BGP peering design

To ensure route symmetry we can:
* Create a new route-table ID, for example "100"
* In route table 100, add a default route pointing to the floating SVI secondary address or the BD
* Use ip rules so that traffic that is sourced from either of the following
  * the service interface IP address
  * the service IP pool 
  is going to use route table 100, thus ensuring that traffic will be sent back to the L3Out, which preserves routing symmetry.

![alt text](../images/BGP-Control-Plane-high-leve.png)
Cilium BGP Control Plane high-level cluster topology

![alt text](../images/BGP-Control-Plane-flow.png)
Cilium BGP Control Plane traffic flows

Refer to the [ACI BGP Design](../aci_design/) section for details on how to configure ACI.

## Cilium BGP design

When it comes to the Cilium BGP design, the only real decisions to make is how many `ingress nodes` to deploy and whether to dedicate them only for this purpose.

Ideally, the design should have a minimum of two `ingress nodes` distributed between two pairs of leaves. This will provide redundancy in case of `ingress nodes` or ACI leaf failure or during upgrades.

Depending on the cluster scale and application requirements, dedicated `ingress nodes` could be beneficial for the following reasons:

* Deterministic latency: if no Pods are running on the `ingress nodes` all the ingress traffic go through two hops ensuring a deterministic latency.
* Limit the number of `ingress nodes`: By having dedicated `ingress nodes`, we can ensure that all the bandwidth and compute are dedicated to the ingress function.

* Specialized hardware: not all the nodes in a Kubernetes clusters need to be the same; specialized high-performance hardware can be utilized for our `ingress nodes`. For example, thanks to [CiliumNodeConfig](https://docs.cilium.io/en/latest/configuration/per-node-config/#per-node-configuration), we can deploy bare-metal nodes with either Mellanox or Intel network interface cards and achieve 100+ Gbps throughput per ingress node, thanks to [Cilium Big TCP](https://docs.cilium.io/en/stable/operations/performance/tuning/#ipv4-big-tcp) capability.

Refer to the [Example configuration](../examples/examples/) section of this document for implementation details.

## Cilium Egress design

When it comes to the Cilium Egress design, the only real decisions to make is how many `egress nodes` to deploy and whether to dedicate them only for this purpose.
Ideally, the design should have a minimum of two `egress nodes` distributed between two pairs of leaves. This will provide redundancy in case of `egress nodes` or ACI leaf failure or during upgrades.
Depending on the cluster scale and application requirements, dedicated `egress nodes` could be beneficial for the same reasons discussed in the [Cilium BGP design](#cilium-bgp-design) section

## Design trade offs

This design aims to provide you with an easy and high scalable design; however, it comes with the following drawbacks:

1. External services can only be advertised as “Cluster Scope”: since all the incoming traffic has to pass through a subset of Kubernetes nodes, we need to be able to perform a second-stage load balancing to reach the “correct” node.
2. Potential bottle necks for ingress/egress traffic
3. BGP cannot be used to advertise the Pod subnets.

For issue (1) there is no solution; this is a trade-off we make to achieve a more scalable and simple design. Issue (2) can be easily addressed with either vertical or horizontal scaling.

Issue (3) can be addressed with Cilium Egress Gateway.

[Next](/docs/advanced_design/){: .btn }
{: .text-right }