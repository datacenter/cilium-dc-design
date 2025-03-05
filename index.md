---
title: Isovalent and Cisco DC Fabrics
layout: default
nav_order: 1
---

{: .warning }
Disclaimer:
This design document is currently under review and is subject to change. The information contained herein is preliminary and may be updated or modified as the review process progresses. 

# Introduction
{: .no_toc }


Welcome to the Isovalent Best Practices for the Modern Datacenter Documentation Site! This resource is designed to provide guidance and insights into the Networking best practices for deploying Kubernetes Clusters using Isovalent as CNI. While we strive to cover a wide range of scenarios and design considerations, please note that the information provided here addresses the most common designs and situations. We encourage users to conduct their own research and tailor their designs to meet specific needs and requirements.

The Isovalent Best Practices for the Modern Datacenter Documentation Site serves as a comprehensive guide to help you navigate through various concepts and options to strike a balance between feature and complexity when Deploying Kubernetes Cluster in the Datacenter. Our goal is to equip you with the knowledge and tools needed to implement effective and efficient solutions.

We welcome contributions from the community! If you have insights or improvements you'd like to share, please feel free to submit a pull request or open an issue on our GitHub repository.

## Kubernetes 

Kubernetes has become the de facto standard for container orchestration in today's cloud-native ecosystem, providing a robust framework for deploying, scaling, and managing containerized applications. As enterprises increasingly adopt Kubernetes, they are often faced with the challenge of ensuring seamless network connectivity and service discovery across diverse and dynamic environments.
Traditionally, implementing BGP peering to all the nodes in a Kubernetes cluster has been the go-to strategy to provide security and visibility into the workload running on the cluster as well as handling external services routing and load-balancing requirements.

BGP peering enables Kubernetes nodes to directly exchange routing information, promoting efficient network traffic flow, and reducing latency.
However when designing BGP peering for Kubernetes clusters, especially as they grow in size and complexity, several key considerations can help ensure efficient, scalable, and manageable network configurations. 

This white paper proposes two design solutions:

- A Simplicity-First Approach Design trading some features and performances for ease of config and operation
- A slightly more Advanced Design that trades some simplicity for high performance and resilient ECMP hashing for exposed services.

At present, these designs does not necessitate the use of supplementary components from Cisco or other third-party vendors to meet its objectives. While there is the potential for users or the open-source community to create tools that automate the Cisco ACI configuration, such developments are optional and beyond the scope of this design document.

This white paper will thoroughly examine the technical underpinnings, implementation considerations, and operational benefits of this approach. It will guide readers through the process of implementing this hybrid model, ensuring that network reliability, performance, and security are not only preserved but significantly enhanced.

## Cilium

Cilium is an open-source software project that provides networking, security, and observability capabilities for containerized applications, particularly in Kubernetes environments. It leverages eBPF (extended Berkeley Packet Filter), a powerful Linux kernel technology, to efficiently manage network policies, service connectivity, and security at the application layer.

Key features of Cilium include:
* **Advanced Network Security**: Cilium enables fine-grained security policies that can be defined based on application-layer identities rather than just IP addresses. This allows for more precise control over communication between microservices.
* **Scalability and Performance**: By using eBPF, Cilium achieves high performance and scalability, as it can run network policies directly in the Linux kernel without the need for complex user-space processing.
* **Service Mesh Integration**: Cilium can be integrated with service mesh architectures to provide transparent, high-performance networking and security features that complement service mesh capabilities.
* **Observability**: Cilium offers robust observability tools, allowing users to monitor and troubleshoot network traffic and security policies with detailed metrics and visibility into application communication patterns.
* **Kubernetes Native**: Designed with Kubernetes in mind, Cilium seamlessly integrates with Kubernetes to provide native support for network policies and service discove* 
Cilium is widely adopted in cloud-native environments for its ability to enhance security and network performance while providing comprehensive observability and integration with existing Kubernetes infrastructure.

## Isovalent 

Isovalent (now part of Cisco) is a company that specializes in providing advanced networking and security solutions for cloud-native environments. It is the primary developer and maintainer of Cilium.

## Cilium vs Cilium Enterprise

Cilium and Cilium Enterprise both provide advanced networking, security, and observability solutions for cloud-native environments, particularly those using Kubernetes. However, there are key differences between the two, primarily in terms of features, support, and target audience.

### Cilium:

* Open Source: Cilium is an open-source project available to anyone. It is maintained by a community of contributors, with Isovalent as the primary maintainer.
* Core Features: It provides essential functionalities such as network security policies, load balancing, and observability using eBPF (extended Berkeley Packet Filter) technology.
* Community Support: Users rely on community forums, GitHub issues, and public documentation for support.
* Flexibility and Innovation: As an open-source project, Cilium benefits from continuous innovation and contributions from a broad community of developers and users.

### Cilium Enterprise:

* Commercial Offering: Cilium Enterprise is the commercial version offered by Isovalent, built on top of the open-source Cilium project.
* Enhanced Features: It includes additional enterprise-grade features and capabilities that are not available in the open-source version. These may include advanced security features, compliance tools, extended observability, and integrations with other enterprise systems.
* Professional Support: Customers receive professional support, including SLAs, access to a dedicated support team, and potentially custom engineering services.
* Enterprise Integration: Designed to integrate seamlessly into large-scale enterprise environments, offering improved reliability, scalability, and ease of management.

This design best practice document is written with enterprise customers in mind and as such is taking advantages of Cilium Enterprise features.
Where Enterprise features are used they will be called out and if possible an alternative for the Open Source version will be presented.

## Goals

The design goals for our Kubernetes networking solution focus on addressing the critical challenges faced by administrators when managing traffic flow and ensuring secure connectivity within and outside the Kubernetes cluster. Our approach is to implement a robust and scalable load-balancing strategy coupled with a secure networking model that bridges the gap between containerized applications and external non-containerized systems.

## Two Designs
This design document outlines two potential approaches to achieving our networking objectives. Each option has been crafted to cater to different priorities and operational considerations. Regardless of the options you choose both design can provide you with the following outcome:

* High Performance
  * By using [Native Routing](https://docs.cilium.io/en/stable/network/concepts/routing/#native-routing) it is possible to increase the networking stack performances by removing the the Nodep-to-Node overlay as well as enabling advanced features like [BigTCP](https://docs.cilium.io/en/stable/operations/performance/tuning/#ipv4-big-tcp) or `endpoint routes` to bypass the `cilium_host` interface for added performance.

* Efficient load balancing:
  * load-balancing mechanism that effectively distributes external client traffic to services within the Kubernetes cluster
  * Utilize cluster nodes to perform BGP peering, enabling Layer-3/Layer-4 Equal-Cost Multi-Path ECMP-based load balancing

* External service security:
    * The load balancer IP is mapped to an ACI external Endpoint Group (external EPG) to facilitate the enforcement of security policies through ACI contracts

* Pod identity
  * With the [Egress Gateway](https://docs.cilium.io/en/stable/network/egress-gateway/egress-gateway/) feature it's possible to masqueraded POD traffic leaving the cluster with predictable IPs associated with the gateway nodes. As an example, this feature can be used in combination with legacy firewalls to allow traffic to legacy infrastructure only from specific pods within a given namespace.

* DHCP relay support:
  * This will ensure it is easy to bootstrap and horizontally scale a given cluster.

  {: .warning } 
  DHCP Replay has limitation when used on an ACI L3OUT See: [DHCP Limitations](https://www.cisco.com/c/en/us/td/cilium-dc-design/docs/dcn/aci/apic/6x/basic-configuration/cisco-apic-basic-configuration-guide-61x/provisioning-core-aci-fabric-services-61x.html#guidelines-and-limitations-for-a-dhcp-relay-policy)

### Simplicity-First Approach Design
* Objectives: 
  * Ability to run on any ACI Software version
  * Provide a straightforward and easily maintainable solution.
  * Utilize ACI for External Service LoadBalancing
  * Utilize ACI Contract for macro segmentation for North-South Traffic
* Approach: 
  * All Nodes are deployed in an EPG and are mapped into an ESG
  * Native routing mode: Since all the nodes share the same L2 Domain (the Node BD) POD to POD traffic can be directly routed
  * Dedicated Egress nodes, with the Egress IP(s) mapped into an ESG
  * Dedicated Ingress nodes peering with BGP for External Services
* Benefits: Quick deployment, reduced complexity, and lower operational overhead.
* Limitations: 
  * Cilium selects service backends randomly and ensures that the traffic remains sticky to the backend. However, the issue with this scheme is that in case of a node failure (or even a change in the ECMP Next Hops), ECMP can select different Kubernetes node which has no prior context on which backend is currently serving the connection. This eventually leads to unexpected disruptions on connection-oriented protocols like TCP as client connections are being reset by the newly selected backends.
  * May not fully optimize network throughput in scenarios with complex traffic patterns
  
### Advanced Design with Maglev for ECMP Flow Consistency

* Objectives: 
  * Maximize network performance and ensure consistent flow distribution in ECMP scenarios with [MagLev](docs/fabric_agnostic_features/#maglev).
  * Utilize ACI Contract for macro segmentation for North-South Traffic
  * Approach: 
    * All Nodes are deployed in a L3OUT
      * Use a `Node` External EPG to secure Kubernetes Nodes traffic
      * Use one External EPG per exposed External Service for North-South traffic accessing Kubernetes services.
      * Leverage [Maglev hashing](docs/fabric_agnostic_features/#maglev) to maintain ECMP flow consistency. It is suited for environments requiring optimal load distribution and performance. Maglev adds the following requirements:
      * Direct Server Return: the response is sent directly back to the client from the pod, bypassing the original node that received the request.
      * Native routing mode: The native routing mode leverages the routing capabilities of the network Cilium runs on instead of performing encapsulation for POD-to-POD traffic
  * Dedicated Egress nodes, with the Egress IP(s) mapped into an ESG
* Benefits: Improved flow consistency, enhanced network utilization, and better handling of dynamic traffic patterns.
* Limitations:
  * ACI 6.1.2 or newer is required
  * Node IP visibility is lost as they are placed in an L3OUT


[Next](/cilium-dc-design/docs/fabric_agnostic_features/){: .btn }
{: .text-right }
