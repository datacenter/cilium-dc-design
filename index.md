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


Welcome to the Best Practices for the Modern Datacenter Documentation Site covering Kubernetes deployments with Cisco ACI and Isovalent Networking for Kubernetes! This resource is designed to provide guidance and insights into the Networking best practices for deploying Kubernetes Clusters using Isovalent Networking for Kubernetes (based on Cilium) as CNI. While we strive to cover a wide range of scenarios and design considerations, please note that not all options are covered. Alternative designs exist, and we encourage users to conduct their own research and tailor their designs to meet specific needs and requirements.

These Best Practices for the Modern Datacenter Documentation Site serves as a comprehensive guide to help you navigate through various concepts and options to strike a balance between features and complexity when deploying Kubernetes Clusters in the Datacenter. Our goal is to equip you with the knowledge and tools needed to implement effective and efficient solutions.

We welcome contributions from the community! If you have insights or improvements you'd like to share, please feel free to submit a pull request or open an issue on our GitHub repository.

## Document Structure

This design document is composed by the following chapter/sections

- [Fabric Agnostic Features](docs/fabric_agnostic_features/): This chapter outlines the Cilium features that are common across all designs.
- **ACI Designs:**
  - [Simplicity Design](docs/aci/simplicity_design/): The most straightforward design to get you started, ideal for testing basic functionalities.
  - [Advanced Design](docs/aci/advanced_design/): An optimized, high-scale design that trades configuration simplicity for increased scalability. (Recommended)
  - [ACI BGP Design](docs/aci/aci_bgp_design/): This section provides details on the ACI BGP configuration, which remains consistent regardless of the chosen design.
  - [Scale Testing](docs/aci/aci_tested_scale/): A collection of scale metrics and tests conducted for the Advanced design.
  - [Config Examples](docs/aci/examples/examples/): Complete configuration examples for both ACI and Kubernetes/OpenShift.


## Kubernetes 

Kubernetes has become the de facto standard for container orchestration in today's cloud-native ecosystem, providing a robust framework for deploying, scaling, and managing containerized applications. As enterprises increasingly adopt Kubernetes, they are often faced with the challenge of ensuring seamless network connectivity and service discovery across diverse and dynamic environments.
Traditionally, implementing BGP peering to all the nodes in a Kubernetes cluster has been the go-to strategy to provide security and visibility into the workload running on the cluster as well as handling external services routing and load-balancing requirements.

BGP peering enables Kubernetes nodes to directly exchange routing information, promoting efficient network traffic flow, and reducing latency.
However when designing BGP peering for Kubernetes clusters, especially as they grow in size and complexity, several key considerations can help ensure efficient, scalable, and manageable network configurations. 

This white paper proposes two design solutions:

- A Simplicity-First Approach Design trading some features and performance for ease of configuration and operations.
- A slightly more Advanced Design that trades some simplicity for better performance and resilient ECMP hashing towards services.

At present, these designs does not necessitate the use of supplementary components from Cisco or other third-party vendors to meet its objectives. We encourage users to programatically deploy or even automate the configuration on ACI. Such efforts can aid in the repeatability, reduce risk, and accelerate deployments. However, such efforts are optional and are beyond the scope of this design document.

This white paper will thoroughly examine the technical underpinnings, implementation considerations, and operational benefits of this approach. It will guide readers through the process of implementing this hybrid model, ensuring that network reliability, performance, and security are not only preserved but significantly enhanced.

## Isovalent Networking for Kubernetes (Cilium)

Isovalent Networking for Kubernetes provides networking, security, and observability capabilities for containerized applications, particularly in Kubernetes environments. It leverages eBPF (extended Berkeley Packet Filter), a powerful Linux kernel technology, to efficiently manage network policies, service connectivity, and security at the application layer. It is the enterprise offering of the Cilium open source project, which was created by Isovalent. 

Key features of Isovalent Networking for Kubernetes include:
* **Advanced Network Security**: Isovalent Networking for Kubernetes enables fine-grained security policies that can be defined based on application-layer identities rather than just IP addresses. This allows for more precise control over communication between microservices.
* **Scalability and Performance**: By using eBPF, Isovalent Networking for Kubernetes achieves high performance and scalability, as it can run network policies directly in the Linux kernel without the need for complex user-space processing.
* **Service Mesh Integration**: Isovalent Networking for Kubernetes can be integrated with service mesh architectures to provide transparent, high-performance networking and security features that complement service mesh capabilities.
* **Observability**: Isovalent Networking for Kubernetes offers robust observability tools, allowing users to monitor and troubleshoot network traffic and security policies with detailed metrics and visibility into application communication patterns.
* **Kubernetes Native**: Designed with Kubernetes in mind, Isovalent Networking for Kubernetes seamlessly integrates with Kubernetes to provide native support for network policies and service discovery

Isovalent Networking for Kubernetes is widely adopted in cloud-native environments for its ability to enhance security and network performance while providing comprehensive observability and integration with existing Kubernetes infrastructure.

## Isovalent 

As the creator of eBPF, Cilium, and Tetragon, Isovalent has become a cornerstone of innovation in cloud-native ecosystems. All major cloud providers have adopted Isovalent’s technologies, making it the de-facto standard for cloud-native networking and security.  

The Isovalent Enterprise Platform delivers advanced Networking, Security, and Observability solutions for Kubernetes and cloud-native environments. This is combined with a [global customer support offering](https://isovalent.com/blog/post/isovalent-enterprise-cilium-support-customer-environments/). 

Isovalent joined Cisco via acquisition in April 2024. 

The Isovalent Platform is trusted by enterprises such as Goldman Sachs, J.P. Morgan Chase, UBS, and S&P Global, as well as technology leaders like Databricks, Adobe, and Confluent, and some of the leading AI Large Language Model (LLM) providers. 


{: .note }
Disclaimer: This document has been developed exclusively with **Isovalent Networking for Kubernetes** version in mind. No support or guidance is provided for Cilium open-source version. 
Cisco TAC does not provide support for Cilium and Isovalent Networking for Kubernetes support can be purchased separately 

## How to Request Design Assistance

We understand that every network environment is unique, and we're here to help tailor solutions to meet your specific needs. If you require design assistance, our team of experts is ready to support you through the process. Here's how you can get in touch with us:

### Contact Information

* [Contact Isovalent](mailto:isovalentsales@cisco.com)
* Contact your Cisco account manager ([Global Contact Page](https://www.cisco.com/site/us/en/about/contact-cisco/index.html))
* [Request a demo](https://isovalent.com/request-demo/)

Our team is committed to providing personalized support and guidance to ensure your network design aligns with your operational requirements and future growth plans. We look forward to working with you to achieve your goals!

## Goals

The design goals for our Kubernetes networking solution focus on addressing the critical challenges faced by administrators when managing traffic flow and ensuring secure connectivity within and outside the Kubernetes cluster. Our approach is to implement a robust and scalable load-balancing strategy coupled with a secure networking model that bridges the gap between containerized applications and external non-containerized systems.

{: .note }
With both of the designs discussed in this document, the role of the network fabric (ACI in this case) is to provide **North-South** traffic filtering. ACI can selectively permit or deny traffic ingressing or egressing the Kubernetes or Openshift cluster using a combination of security contracts and [ESGs](https://www.cisco.com/c/en/us/td/docs/dcn/whitepapers/cisco-aci-esg-design-guide.html). ACI **cannot** regulate intra-cluster East-West traffic using the approaches proposed here. This role is left to [Cilium's rich security feature set](https://docs.cilium.io/en/latest/overview/intro/). Cilium Enterprise ships with a powerful yet easy to use [GUI tool](https://docs.cilium.io/en/stable/observability/hubble/#hubble-intro) that lets security administrators create, test, validate and monitor intra-cluster security policies. Combining ACI with Cilium provides a very complementary layered approach to securing the data center.  


## Two Designs
This design document outlines two approaches to achieving our networking objectives. Each option has been crafted to cater to different priorities and operational considerations. 

Regardless of the options you choose **both** designs can provide you with the following outcomes:

* High Performance
  * By using [Native Routing](https://docs.cilium.io/en/stable/network/concepts/routing/#native-routing) it is possible to increase the networking stack performance by removing the the Node-to-Node overlay as well as enabling advanced features like [BigTCP](https://docs.cilium.io/en/stable/operations/performance/tuning/#ipv4-big-tcp).
  
  {: .note }
  BigTCP is currently in beta and should not be used for production workloads. However, this design is prepared for its future stable release, allowing it to be enabled easily with a simple configuration flag once it becomes available.

* Efficient load balancing:
  * Load-balancing that effectively distributes external client traffic to services within the Kubernetes cluster
  * Utilize cluster nodes to perform BGP peering, enabling Layer-3/Layer-4 Equal-Cost Multi-Path ECMP-based load balancing

* External service security:
    * The load balancer IP is mapped to an ACI external Endpoint Group (external EPG) to facilitate the enforcement of security policies through ACI contracts

* Pod identity
  * With the [Egress Gateway](https://docs.cilium.io/en/stable/network/egress-gateway/egress-gateway/) feature it's possible to masquerade POD traffic leaving the cluster with predictable IPs associated with the gateway nodes. As an example, this feature can be used in combination with legacy firewalls to allow traffic to legacy infrastructure only from specific pods within a given namespace.

* DHCP relay support:
  * This will ensure it is easy to bootstrap and horizontally scale a given cluster.

  {: .warning } 
  Please be aware of the DHCP relay limitations for L3outs. See: [DHCP Limitations](https://www.cisco.com/c/en/us/td/docs/dcn/aci/apic/6x/basic-configuration/cisco-apic-basic-configuration-guide-61x/provisioning-core-aci-fabric-services-61x.html#guidelines-and-limitations-for-a-dhcp-relay-policy)

### Simplicity-First Approach Design 

This design serves as an excellent starting point for quickly getting up and running, allowing you to explore the capabilities of Cilium with ease.

* Objectives: 
  * Ability to run on any ACI Software version
  * Provide a straightforward and easily maintainable solution.
  * Utilize ACI for External Service load-balancing
  * Utilize ACI Contracts to control North-South traffic
* Approach: 
  * All Cluster Nodes are deployed in an EPG and are mapped into an ESG
  * Native routing mode: Since all the nodes share the same L2 Domain (the Node BD) POD to POD traffic can be directly routed
  * Dedicated Egress nodes, with the Egress IP(s) mapped into an ESG
  * Dedicated Ingress nodes peering with BGP for External Services
* Benefits: Quick deployment, reduced complexity, and lower operational overhead.
* Limitations: 
  * Cilium selects service backends randomly and ensures that the traffic remains sticky to the backend. However, this method can cause issues if a node fails or if there is a change in the ECMP Next Hops. This can result into traffic being sent to a different Kubernetes node, which lacks context on which backend is currently serving the connection. This can lead to unexpected disruptions on connection-oriented protocols like TCP, as client connections are being reset by the newly selected backends.
  * May not fully optimize network throughput in scenarios with complex traffic patterns
  
### Advanced Design with Maglev for ECMP Flow Consistency (Recommended)

This design offers a more comprehensive solution, ideal for production environments. It incorporates advanced features and optimizations that are recommended for leveraging the full potential of Cilium in a stable and scalable manner. While it may require more initial setup and configuration, this design ensures robust performance and resilience, making it well-suited for enterprise-level deployments.

* Objectives: 
  * Maximize network performance and ensure consistent flow distribution in ECMP scenarios with [Maglev](docs/fabric_agnostic_features/#maglev).
  * Utilize ACI Contracts to control North-South traffic
  * Approach: 
    * All Nodes are deployed behind a L3Out
      * Use a `Node` External EPG to secure Kubernetes Nodes traffic
      * Use one External EPG per exposed External Service for North-South traffic accessing Kubernetes services.
      * Leverage [Maglev hashing](docs/fabric_agnostic_features/#maglev) to maintain ECMP flow consistency. It is suited for environments requiring optimal load distribution and performance. Maglev adds the following requirements:
      * Direct Server Return: the response is sent directly back to the client from the pod, bypassing the original node that received the request.
      * Native routing mode: The native routing mode leverages the routing capabilities of the network Cilium runs on instead of performing encapsulation for POD-to-POD traffic
  * Dedicated Egress nodes, with the Egress IP(s) mapped into an ESG
* Benefits: Improved flow consistency, enhanced network utilization, and better handling of dynamic traffic patterns.
* Limitations:
  * ACI 6.1(2) or later is required
  * Node IP visibility is lost as they are placed behind an L3Out


[Next](/cilium-dc-design/docs/fabric_agnostic_features/){: .btn }
{: .text-right }



