---
title: Fabric Agnostic Features
layout: default
parent: Isovalent and Cisco DC Fabrics
---
## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Common Features

### Native Routing and Auto Direct Node Routes

By enabling these 2 features we can implement an efficient packet forwarding between PODs without requiring additional encapsulation or overlay networks. By having direct routes to each pod's IP address subnet, PODs can communicate directly over the existing Layer 2 infrastructure, reducing latency and potential overhead associated with tunneling protocols.

The Auto Direct Node Routes feature thus leverages the existing L2 topology to streamline pod-to-pod communication, ensuring that packet delivery is both efficient and straightforward. It eliminates the need for exposing the POD Subnet to the broader network fabric, thereby maintaining a cleaner and simple network architecture.

This configuration aligns with Cilium's ethos of providing high-performance, scalable, and simple networking for Kubernetes environments. By integrating closely with the Linux kernel's routing capabilities, Cilium can offer robust networking solutions without necessitating complex configurations or additional network infrastructure.

{: .warning}
This design is intended to address 90% of common use cases. However, if your cluster scales beyond 1,000 nodes, adjustments to the design may be necessary. In such cases, we strongly encourage you to contact Isovalent for guidance and support.

### Cilium BGP Control Plane

Cilium BGP Control Plane [Enterprise](https://docs.isovalent.com/configuration-guide/networking/bgpv2/index.html) or [OSS](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane/) provides a way for Cilium to advertise routes to connected routers by using the Border Gateway Protocol (BGP). Cilium BGP Control Plane makes pod networks and/or load-balancer services of type LoadBalancer reachable from outside the cluster for environments that support BGP. In Cilium, the BGP Control Plane does not program the Linux host data path, so it cannot be used to establish IP reachability within the cluster or to external IPs.

### Cilium Egress Gateway

The [Egress Gateway](https://docs.cilium.io/en/stable/network/egress-gateway/egress-gateway) features allows for redirecting traffic originating from pods and destined to specific CIDRs outside the cluster to be routed through particular nodes.

When the egress gateway feature is enabled and egress gateway policies are in place, packets leaving the cluster are masqueraded with selected, predictable IPs (`egressIP`) associated with the gateway nodes. As an example, this feature can be used with legacy firewalls to allow traffic to legacy infrastructure only from specific pods within a given namespace. These pods typically have ever-changing IP addresses. 

{: .warning}
Only Cilium Enterprise supports [Egress Gateway High Availability](https://docs.isovalent.com/configuration-guide/networking/egress-gateway/index.html)

### XDP Acceleration

[The XDP Acceleration](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#loadbalancer-nodeport-xdp-acceleration) supports NodePort, LoadBalancer services and services with externalIPs for the case where the arriving request needs to be forwarded and the backend is located on a remote node. This feature was introduced in Cilium version 1.8 at the XDP (eXpress Data Path) layer where eBPF is operating directly in the networking driver instead of a higher layer.
The majority of drivers supporting 10G or higher rates also support native XDP on a recent kernel and this feature should be enabled.

## Advanced Design Only Features

### Maglev

Incorporating advanced load balancing mechanisms into Kubernetes clusters is essential for maintaining optimal performance and efficient resource utilization. The integration of Cilium with Maglev hashing provides a robust solution for consistent and efficient load distribution, particularly in scenarios involving Equal-Cost Multi-Path (ECMP) routing.

Maglev consistent hashing minimizes such disruptions by ensuring that each load balancing node has a consistent view and ordering for the backend lookup table such that selecting the backend through the packet's 5-tuple hash will always result in forwarding the traffic to the very same backend without having to synchronize state with the other nodes. This not only improves resiliency in case of failures but also achieves better load balancing properties since newly added nodes will make the same consistent backend selection throughout the cluster.

![cilium-maglev](../images/cilium-maglev.gif)

Aside from that, the Maglev consistent hashing algorithm ensures even balancing among backends as well as minimal disruptions in the case where backends are added or removed. Specifically, a flow is highly likely to choose the same backend after adding or removing a backend for a service as it did before the operation. Upon backend removal, the backend lookup tables are reprogrammed with minimal changes for unrelated backends, that is, typical configurations provide the upper bound of at most 1% tolerable difference in the reassignments.

This is particularly important for any network fabric that don't support ECMP Resilient Hashing.

To successfully implement Maglev hashing within a Kubernetes environment using Cilium, a key requirement is that Kubernetes nodes can select the Pod IP as the destination IP for traffic routing. This capability is crucial for maintaining consistent and efficient load balancing across the network. Additionally, for optimal performance and reduced latency, the node where the Pod resides should be able to reply directly back to the network. This direct response mechanism requires the implementation of Direct Server Return (DSR).

[This](https://cilium.io/blog/2020/11/10/cilium-19/) blog details explore Maglev in details.

### Direct Server Return

When Direct Server Return (DSR) mode is enabled in Cilium, the efficiency of handling node-external traffic is significantly improved. In this mode, once a request reaches a backend pod located on a remote node, the response is sent directly back to the client from the pod, bypassing the original node that received the request. This eliminates the additional hop needed for reverse SNAT, reducing latency and improving overall network performance.

The DSR mode provides two key benefits:

* Preservation of Source IP: Since the backend pod sends the response directly to the client, the original source IP of the client is preserved. This is particularly beneficial for security and monitoring purposes, as it allows network policies and logging mechanisms on the backend node to accurately identify and match the client's IP address. This can be crucial for implementing fine-grained security policies and for troubleshooting.
* Reduced Latency and Load: By removing the need for the response to pass back through the node that initially received the request, the latency is reduced. This also decreases the processing load on that node, as it does not have to handle reverse SNAT for responses, allowing it to manage incoming requests more efficiently.

Overall, enabling DSR in Cilium optimizes network traffic flow, enhances security, and ensures more efficient resource usage within the Kubernetes cluster. It's a strategic choice for environments where performance and accurate client identification are priorities.

[Next](/cilium-dc-design/docs/aci/aci_designs/){: .btn }
{: .text-right }
