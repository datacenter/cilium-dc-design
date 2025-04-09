---
title: ACI Scale Testing
layout: default
parent: ACI Designs
nav_order: 4
---

# Scale Testing - Work In Progress


The [Advanced Design](../../advanced_design) outlined in this document has undergone scalability testing to ensure its effectiveness and reliability under various conditions. However, it is important to note that these tests were conducted based on general scenarios and assumptions. As each organization's needs and architecture are unique, we strongly recommend conducting your own testing to validate the design's performance and suitability within your specific environment.

Custom testing will help identify any potential issues that may arise due to unique architectural elements or specific use cases pertinent to your organization. By doing so, you can ensure that the solution meets your performance expectations and integrates seamlessly with your existing systems.


The Advanced design has been currently tested with:

- 350 Node Kubernetes Cluster
- Each Node is advertising 250 BGP /32 Service Routes. 
- All Nodes peers to 2 ACI Boarder Leaves
- ACI Fabric is composed by 4 leaves
- Clients are accessing the service via: 
  - An L3OUT
  - An EPG/ESG

## Generic ACI Scale 

In the context of the Isovalent and Cisco DC Fabrics design these are the metrics we need to keep in mind:

{: .note }
These are metric for ACI 6.1.2, if you are using a different version refer to the [Verified Scalability Guide](https://www.cisco.com/c/en/us/support/cloud-systems-management/application-policy-infrastructure-controller-apic/tsd-products-support-series-home.html) for your version


- Floating L3Out: Max of 6 anchors and 32 non-anchor
- IPs per MAC = 4096
- BFD neighbors: 2,000 sessions using these minimum BFD timers: minTx:300, minRx:300, multiplier:3
- Number of BGP neighbors (2000 per leaf with up to 70000 external prefixes with a single path). 20k per fabric scale
- Shared L3Out (when leaking between VRFs) 2000 IPv4 prefixes
- External EPGs per L3out (250 per L3out), 600 fabric wide
- Number of ESGs per Fabric = 10000
- Number of ESGs per VRF = 4000
- Number of ESGs per tenant = 4000
- Number of L3 IP Selectors per leaf = 5000
- Number of IP Longest Prefix Matches (LPM) entries: 20000 IPv4 with default dual stack profile. Worth noting that changing the profile reduces the amount of supported ECMP paths.

## Conducted Tests:

### Adding/Removing BGP Peers

**Test:**

Removing and then adding the label that enables the node for BGP peering.

**Impact:**

None. This is expected thanks to Maglev even if the nodes still receives the traffic will just be able to forward it on to the correct POD.

### Reloading a Kubernetes Node

**Test:**

This test is conducted by gracefully reloading a node.

**Impact:**

Minimal. Traffic can be dropped during Routing Table Re Convergence. I.E. if traffic sent to the node that is reloading before is removed as a valid Next Hop. 
This issue can be minimized by first removing the node from BGP Peering. 

### Isovalent Networking for Kubernetes Upgrade

The BFD and BGP Process are running on the Cilium POD. Restarting the Cilium POD for any reason will result in the BFD adjacency to go down however, thanks to BGP Graceful Restart the impact is minimal.

**Test:**

Restart the Cilium POD (or Upgrading Cilium)

**Impact:**

Minimal to none. A Cilium Agent POD restart triggers Graceful Restart.

## Reloading an Anchor Node
**Test:**

Reloading an Anchor Node from the CLI without Sending BFD Down Messages

**Impact**

None aside for potential in flight packets. 
The routing tables are not impacted by this as all the next hops are the Kubernetes Nodes IP.

[Next](/cilium-dc-design/docs/aci/examples/examples/){: .btn }
{: .text-right }