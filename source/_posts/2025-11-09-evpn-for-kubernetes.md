---
title: OpenPERouter -- Bringing EVPN to Kubernetes
date: 2025-11-09 04:29:02
tags:
  - EVPN
  - Kubernetes
  - Network
---

Currently, the mainstream choices for implementing multi-tenancy in Kubernetes networking within the community are [Kube-OVN](https://github.com/kubeovn/kube-ovn) and [ovn-kubernetes](https://github.com/ovn-kubernetes/ovn-kubernetes). Essentially, both solutions build an overlay virtual network on top of the physical network using the capabilities of OVS and OVN. This approach is less concerned with the underlying physical network architecture and offers good compatibility. However, the dual-layer network architecture also increases the overall complexity.

Recently, while researching EVPN as a multi-tenancy solution for physical networks, I discovered the open-source project [OpenPERouter](https://github.com/openperouter/openperouter). It introduces the concept of EVPN into container networking, providing a new approach to achieving multi-tenancy in Kubernetes. This solution not only unifies software and hardware network architectures but also offers some compatibility with existing CNIs like Calico, which advertise routes via BGP. I can even foresee the potential for Calico to gain multi-tenancy capabilities with minimal effort. Although this project is still in its early stages, I believe it represents a promising direction and could become a competitive option for building large-scale data centers in the future.

## Limitations of OVN-Based Solutions

OVN-based solutions have two main limitations: the scalability constraints of the centralized control plane and network complexity.

## Centralized Control Plane

Although Kube-OVN has been deployed in community cases with thousands of nodes at scale, the centralized control plane architecture of OVN places significant pressure on the control plane, making it a bottleneck for the entire cluster. Especially when control plane nodes experience power loss or failures, it may take a considerable amount of time for the network control plane to recover.

![ovn-limit](../images/ovn-limit.png)

This issue primarily stems from the centralized control plane architecture of OVN and cannot be entirely avoided. ovn-kubernetes addresses this bottleneck by deploying one OVN control plane per node and interconnecting multiple nodes via OVN-IC. However, this approach also increases architectural complexity and essentially abandons OVN's inherent cluster networking capabilities, rendering many OVN features unusable.

## Network Complexity

Another issue lies in the inherent complexity of the OVS/OVN system. Users need to build a new knowledge system around OVN specifications and flow tables to ensure they can handle practical problems effectively. Since this system often differs from the underlying physical network, it effectively results in two separate network systems. This not only complicates troubleshooting but also necessitates separate teams for physical and container networking, who often struggle to understand each other's work and collaborate efficiently.

The above limitations are more about trade-offs in solution selection. If the goal is centralized software control and a container network that is agnostic to the underlying physical network, these limitations are an inevitable consequence.

## EVPN

Parallel to the purely software-based approach of OVS, there exists another multi-tenancy networking solution in the hardware world: EVPN.

In the EVPN world, data plane switches encapsulate packets using VXLAN, and traffic from different tenants is isolated by setting the VNI in the VXLAN header.

Meanwhile, switches synchronize L2/L3 routes and VNI information via BGP in the control plane, enabling rapid learning of addresses, routes, and tenant information across the entire network topology.

![EVPN](../images/evpn.png)

This approach allows large data centers to automate distributed multi-tenancy networking. This solution has already been implemented in many large data centers, and mainstream switches currently support it. So, is there a networking solution in the container space based on this technical architecture? This is where OpenPERouter, which I recently discovered, comes in.

## OpenPERouter

The core concept of OpenPERouter is to bring the logic of EVPN switches down to the node level. It runs an FRR instance on each machine, establishing BGP and VXLAN tunnels directly with physical switches to seamlessly integrate container networking into the existing EVPN-based physical network architecture.

![](../images/openperouter.png)

Integrating existing container networks with OpenPERouter is not overly complex. For BGP-based CNIs, establishing a BGP peer with OpenPERouter is required. For CNIs based on veth and bridges, the veth on the host side simply needs to be connected to the net namespace where OpenPERouter resides.

This solution offers several unique advantages:
1. It unifies the underlying network and container network, adopting the same EVPN architecture. Although there are differences in specific operations and software usage, the overall approach is largely consistent.
2. It achieves Underlay and multi-tenancy—two challenging aspects in container networking—in an extremely lightweight manner. Since the main control and data planes reside at the hardware layer, and container networking only handles access, the CNI layer can theoretically be very lightweight.
3. It can transform existing CNIs into multi-tenancy CNIs. Although the project developers haven't explicitly mentioned this, I see the potential for quickly converting Calico into a multi-tenancy network by simply adding a VNI configuration to Calico's IPPool.

Of course, this solution currently has its limitations:
1. A unified network means that container networking fully intrudes into the underlying physical network. This requires a unified management team, which could lead to organizational politics in current environments.
2. OpenPERouter currently does not handle CNI-related tasks itself but instead attempts to integrate with other networking projects. This may be due to the availability of multiple mainstream solutions. However, from my perspective, integrating with other CNIs could make this otherwise simple EVPN solution more complex than the original, and incompatibilities and conflicts are likely to occur in the future. Building a lightweight CNI specifically designed for EVPN from the ground up would be a more elegant choice.

## Summary

In my view, EVPN will be an extremely competitive technical solution for container networking in the future. However, there are currently no mature open-source projects in this area. OpenPERouter is a commendable attempt, and I look forward to seeing a CNI specifically designed for EVPN architecture that can significantly simplify overall network design.