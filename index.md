---
title: Cilium and Cisco DC Fabrics
layout: default
nav_order: 1
---
# Introduction
{: .no_toc }


Kubernetes has become the de facto standard for container orchestration in today's cloud-native ecosystem, providing a robust framework for deploying, scaling, and managing containerized applications. As enterprises increasingly adopt Kubernetes, they are often faced with the challenge of ensuring seamless network connectivity and service discovery across diverse and dynamic environments.
Traditionally, implementing BGP peering to all the nodes in a Kubernetes cluster has been the go-to strategy to provide security and visibility into the workload running on the cluster as well as handling external services routing and load-balancing requirements.

BGP peering enables Kubernetes nodes to directly exchange routing information, promoting efficient network traffic flow, and reducing latency.
However when designing BGP peering for Kubernetes clusters, especially as they grow in size and complexity, several key considerations can help ensure efficient, scalable, and manageable network configurations. 

This white paper proposes two design solutions:

- A Simplicity-First Approach Design trading some features and performances for ease of config and operation
- A slightly more Advanced Design that trades some simplicity for high performance and resilient ECMP hashing for exposed services.

At present, these designs does not necessitate the use of supplementary components from Cisco or other third-party vendors to meet its objectives. While there is the potential for users or the open-source community to create tools that automate the Cisco ACI configuration, such developments are optional and beyond the scope of this design document.

This white paper will thoroughly examine the technical underpinnings, implementation considerations, and operational benefits of this approach. It will guide readers through the process of implementing this hybrid model, ensuring that network reliability, performance, and security are not only preserved but significantly enhanced.

---


## Goals

The design goals for our Kubernetes networking solution focus on addressing the critical challenges faced by administrators when managing traffic flow and ensuring secure connectivity within and outside the Kubernetes cluster. Our approach is to implement a robust and scalable load-balancing strategy coupled with a secure networking model that bridges the gap between containerized applications and external non-containerized systems.

## Two Designs
This design document outlines two potential approaches to achieving our networking objectives. Each option has been crafted to cater to different priorities and operational considerations. Regardless of the options you choose both design can provide you with the following outcome:

* High Performance
  * By using [Native Routing](https://docs.cilium.io/en/stable/network/concepts/routing/#native-routing) it is possible to increase the networking stack performances by removing the the Nodep-to-Node overlay as well as enabling advanced features like [BigTCP](https://docs.cilium.io/en/stable/operations/performance/tuning/#ipv4-big-tcp) or `endpoint routes` to bypass the `cilium_host` interface.

* Efficient load balancing:
  * load-balancing mechanism that effectively distributes external client traffic to services within the Kubernetes cluster
  * Utilize cluster nodes to perform BGP peering, enabling Layer-3/Layer-4 Equal-Cost Multi-Path ECMP-based load balancing

* External service security:
    * The load balancer IP is mapped to an ACI external Endpoint Group (external EPG) to facilitate the enforcement of security policies through ACI contracts

* Pod identity
  * With the egress gateway feature it's possible to masqueraded POD traffic leaving the cluster with predictable IPs associated with the gateway nodes. As an example, this feature can be used in combination with legacy firewalls to allow traffic to legacy infrastructure only from specific pods within a given namespace.

* DHCP relay support:
  * This will ensure it is easy to bootstrap and scale a given cluster.
    * Consideration: DHCP Replay has limitation when used on an L3 OUT See: [DHCP Limitations](https://www.cisco.com/c/en/us/td/docs/dcn/aci/apic/6x/basic-configuration/cisco-apic-basic-configuration-guide-61x/provisioning-core-aci-fabric-services-61x.html#guidelines-and-limitations-for-a-dhcp-relay-policy)

### Simplicity-First Approach Design
* Objectives: 
  * Ability to run on any ACI Software version
  * Provide a straightforward and easily maintainable solution.
  * Utilize ACI for External Service LoadBalancing
  * Utilize ACI Contract for macro segmentation for North-South Traffic
* Approach: 
  * All Nodes are deployed in an EPG and are mapped into an ESG
  * Dedicated Egress nodes, with the Egress IP(s) mapped into an ESG
  * Dedicated Ingress nodes peering with BGP for External Services
  * Native routing mode: Since all the nodes share the same L2 Domain (the Node BD) POD to POD traffic can be directly routed
* Benefits: Quick deployment, reduced complexity, and lower operational overhead.
* Considerations: 
  * Cilium selects service backends randomly and ensures that the traffic remains sticky to the backend. However, the issue with this scheme is that in case of a node failure (or even a change in the ECMP Next Hops), ECMP can select different Kubernetes node which has no prior context on which backend is currently serving the connection. This eventually leads to unexpected disruptions on connection-oriented protocols like TCP as client connections are being reset by the newly selected backends.
  * May not fully optimize network throughput in scenarios with complex traffic patterns
  
### Advanced Design with Maglev for ECMP Flow Consistency
* Objectives: 
  * Utilize ACI for External Service LoadBalancing
    * Maximize network performance and ensure consistent flow distribution in ECMP scenarios with [MagLev](docs/cilium/#maglev).
  * Utilize ACI Contract for macro segmentation for North-South Traffic
  * Approach: 
    * All Nodes are deployed in a L3OUT
      * Use a `Node` External EPG to secure node initiated traffic
      * Use one External EPG per External Service for North-South traffic accessing services.
      * Leverage [Maglev hashing](docs/cilium/#maglev) to maintain ECMP flow consistency. It is suited for environments requiring optimal load distribution and performance. Magles adds the requiremets for the following features:
      * Direct Server Return: ensuring the return traffic follows the optimal traffic path increasing performances
    * Native routing mode: Since all the nodes share the same L2 Domain (the Floating SVI) POD to POD traffic can be directly routed
  * Dedicated Egress nodes, with the Egress IP(s) mapped into an ESG
* Benefits: Improved flow consistency, enhanced network utilization, and better handling of dynamic traffic patterns.
* Considerations:
  * Increased complexity in setup and management, with a steeper learning curve for configuration and maintenance.
  * ACI 6.1.2 or newer is required see [Pre ACI 6.1(2) Limitation](docs/cilium/#pre-aci-612-limitation)
* 
## Required knowledge

This document covers advanced configuration topics in both Cisco ACI and Cilium. It is expected the user is familiar with the following feature and that Cilium Enterprise is deployed. 

* Cisco ACI:
  * Access policies
  * EPG/ESGs/BDs
  * Contracts
  * L3Out floating SVI
  * BGP

* [Cilium](https://docs.cilium.io/)
  * Installation
  * BGP Control Plane
  * LoadBalancer IP Address Management
  * Egress Gateway
  * For the Advanced Design Only:
    * MagLev
    * Direct Routing
    * Endpoint Routes
    * DSR

Note: Cilium OSS can also be used however:
* The following features are not available: BFD, Egress Gateway High Availability
* The CRD for BGP are different, for example Cilium Enterprise  uses `IsovalentBGPClusterConfig` while OSS uses `CiliumBGPPeeringPolicy` and so on.

[Next](/docs/cilium/){: .btn }
{: .text-right }
