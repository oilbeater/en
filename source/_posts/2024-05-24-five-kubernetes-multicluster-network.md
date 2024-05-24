---
title: "5 Solutions for Multi-Cluster Communication in Kubernetes"
date: 2024-05-24 08:25:30
tags:
    - kubernetes
    - network
keywords:
    - kubernetes
    - multicluster
    - network
---

As enterprises scale their operations, the use of Kubernetes extends from single-cluster to multi-cluster deployments. In a multi-cluster environment, communication between clusters becomes a critical area of research. This article introduces the basic principles, advantages, and limitations of five solutions for cross-Kubernetes cluster communication.

## 1. Underlay Network

This type of network plugin includes [macvlan](https://www.cni.dev/plugins/current/main/macvlan/), [ipvlan](https://www.cni.dev/plugins/current/main/ipvlan/), [Kube-OVN underlay](https://kubeovn.github.io/docs/stable/start/underlay/), and various cloud VPC CNIs.

**Basic Principle**:

From the CNI (Container Network Interface) perspective, the underlay network is the simplest approach. This method relies on the underlying infrastructure to achieve connectivity at the network level, such as using VPC Peering on public clouds or configuring routing and large Layer 2 networks in physical networks. Once the underlying network is connected, cross-cluster container networks naturally connect.

**Advantages**:

- Simplest from a CNI perspective, requiring no additional operations.
- Architecturally clear, delegating the responsibility of cross-cluster communication to the underlying network.

**Limitations**:

- Depends on specific CNI; underlay networks have limited use cases, and some scenarios can only use overlay networks.
- Difficulties in heterogeneous environments, such as between multiple clouds or between public and private clouds.
- Provides only basic container network communication without higher-level functionalities like service discovery, domain names, and network policies.
- Lacks fine-grained control by connecting all cluster container networks at once.

## 2. Overlay CNI Providing Cross-Cluster Communication

When underlay networks cannot be used, some specific CNIs achieve cross-cluster communication at the overlay level, such as [Cilium Cluster Mesh](https://cilium.io/use-cases/cluster-mesh/), [Antrea Multi-Cluster](https://antrea.io/docs/v2.0.0/docs/multicluster/quick-start/), and [Kube-OVN with ovn-ic](https://kubeovn.github.io/docs/stable/en/advance/with-ovn-ic/).

**Basic Principle**:

These CNIs typically select a set of nodes within the cluster as gateway nodes, which then establish tunnels between each other. Cross-cluster traffic is forwarded through these gateway nodes.

**Advantages**:

- CNIs include cross-cluster functionality without needing additional components.

**Limitations**:

- Dependent on specific CNIs, cannot achieve communication between different CNIs.
- Cannot handle CIDR overlap; network segments need to be pre-planned.
- Except for Cilium, which implements a complete solution including cross-cluster service discovery and network policies, others only provide basic container network communication.
- Lacks fine-grained control by connecting all cluster container networks at once.

## 3. Submariner

Given the common need for cross-cluster network interconnectivity, similar implementations lead to duplicated efforts among various CNIs. [Submariner](https://submariner.io/), an independent, cross-cluster network plugin, offers a generic solution capable of connecting clusters with different CNIs into one network. Initially created by engineers at Rancher and now a CNCF Sandbox project with active participation from Red Hat engineers.

**Basic Principle**:

Submariner selects gateway nodes within the cluster that communicate via VXLAN. Cross-cluster traffic is transmitted through VXLAN. Submariner relies on the CNI to send egress traffic to the host network, which then forwards it. Additionally, Submariner deploys a set of CoreDNS for cross-cluster service discovery and uses a Globalnet Controller to address CIDR overlap issues.

**Advantages**:

- CNI-agnostic to some extent, allowing the connection of clusters with different CNIs.
- Implements cross-cluster service discovery, supporting service and domain name resolution.
- Supports communication between clusters with overlapping CIDRs, avoiding IP conflicts post-deployment.

**Limitations**:

- Not compatible with all CNIs, especially those like macvlan or short-circuiting Cilium where the host cannot see the traffic.
- Gateway operates in active-passive mode, lacking horizontal load balancing, potentially causing performance bottlenecks in high-traffic scenarios.
- Lacks fine-grained control by connecting all cluster container networks at once.

## 4. Skupper

[Skupper](https://skupper.io/index.html) is considered the most interesting among the solutions, enabling service-layer network connectivity on demand, avoiding the control issues of complete connectivity. It innovatively uses a layer 7 message queue to achieve this, being entirely independent of the underlying network and CNI, making it very easy to get started. Currently, most contributors are engineers from Red Hat.

**Basic Principle**:

Unlike the previous solutions that use tunnels to connect container IPs directly, Skupper introduces the concept of VAN (Virtual Application Networks) to connect networks at layer 7. Simply put, instead of directly connecting IPs, it connects services. Conceptually similar to ServiceExporter and ServiceImporter, but predating these community concepts, it was quite innovative at its inception.

Skupper uses a message queue implementation, forming a large message queue between multiple clusters. Cross-cluster communication packets are sent to this message queue and consumed on the other end. This approach is similar to reverse proxy but implemented with a message queue, making it a very novel idea. Services become message subscription points for consumption, allowing the establishment of a message queue between the server and client as needed, managing and controlling this message path through the message queue concept.

**Advantages**:

- Excellent CNI compatibility, completely independent of CNI behavior at the application layer for packet connectivity.
- Easy to get started, requiring no complex upfront network planning and no CIDR non-overlap requirements. Provides CLI for temporary testing and quick demonstrations.
- Connects services on demand instead of entire container networks, allowing fine-grained control with lower infrastructure requirements.

**Limitations**:

- Currently supports only TCP protocol; UDP and lower-level protocols like ICMP have issues.
- IP information is lost as messages are forwarded through the message queue.
- Using a message queue for forwarding might lead to performance issues like latency and throughput loss.
- Given TCP's stateful nature, converting it entirely to a message queue form and ensuring compatibility is questionable.

## 5. KubeSlice

[KubeSlice](https://kubeslice.io/documentation/open-source/1.3.0) is a project that recently entered the CNCF Sandbox. Solutions based on tunnels cannot achieve fine-grained control and full CNI compatibility, while those based on the application layer cannot fully support network protocols. KubeSlice offers a new approach, attempting to solve both issues simultaneously.

**Basic Principle**:

KubeSlice's basic idea is simple and straightforward: dynamically insert a network card into a pod as needed, creating an overlay network for cross-cluster communication on this card. This overlay network then implements service discovery and network policies. Users can dynamically create and join networks across namespaces or pods, achieving flexible fine-grained control. Since it operates on a second network card, it is highly compatible with network protocols.

**Advantages**:

- High CNI compatibility, as it involves adding an extra network card, regardless of the original CNI and avoiding original network address conflicts.
- High protocol compatibility, as it uses an additional network card for traffic forwarding, compatible with all network protocols.
- High flexibility, providing CLI tools for dynamically creating and joining networks, allowing users to create multiple cross-cluster virtual networks as needed.
- Comprehensive functionality, implementing service discovery, DNS, QoS, NetworkPolicy, and monitoring on the additional overlay network.

**Limitations**:

- Applications need to be aware of the additional network card and choose the appropriate network, potentially requiring application modifications.
- Since cross-cluster traffic uses a different network card, not the pod's primary IP, external systems like monitoring and tracing might require adjustments.
- Documentation appears to be an internal project made open source, with many usage methods and API descriptions not well explained, requiring users to guess parameter meanings from references. Despite seeming comprehensive, the documentation needs significant improvement for external users to use it effectively.

## Summary

In a multi-cluster Kubernetes environment, there are multiple solutions for efficient communication. Each solution has its advantages and limitations, allowing users to choose the most suitable one based on specific needs and environments.

> This post was originally written in Chinese and I translated it to English with the help of GPT4. If you find any errors, please feel free to let me know.