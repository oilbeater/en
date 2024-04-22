
---
title: "k8gb: The Best Open Source GSLB Solution for Cloud Native"
date: 2024-04-18 10:01:10
tags: [kubernetes, network]
---

Balancing traffic across multiple Kubernetes clusters and achieving automatic disaster recovery switching has always been a headache. We have explored public clouds and [Karmada Ingress](https://github.com/karmada-io/multi-cluster-ingress-nginx), and have also tried manual DNS solutions, but these approaches often fell short in terms of cost, universality, flexibility, and automation. It was not until we discovered [k8gb](https://www.k8gb.io/), a project initiated by South Africa's Absa Group to provide banking-level multi-availability, that we realized the ingenuity of using various DNS protocols to deliver a universal and highly automated GSLB solution. This blog will briefly discuss the problems with other approaches and how k8gb cleverly uses DNS to implement GSLB.

## What is GSLB

GSLB (Global Service Load Balancer) is a concept in contrast to single-cluster load balancing, which mainly serves as an entrance to a cluster to distribute traffic within the cluster, whereas GSLB typically acts as an entrance for traffic across multiple clusters, handling load balancing and fault management. On one hand, GSLB can set geographic affinity rules to route traffic closer to users, enhancing overall performance; on the other hand, it can automatically redirect traffic to a functioning cluster when one fails, minimizing the impact on users.

## Problems with Other Solutions

### Commercial Load Balancers

GSLB is not a new concept, so many commercial companies have mature products, such as [F5 GSLB](https://www.f5.com/solutions/use-cases/global-server-load-balancing-gslb). These products generally have the following drawbacks:

1. They do not integrate well with cloud-native systems, often requiring deployment outside Kubernetes clusters, which complicates unified management.
2. They are costly and pose risks of vendor lock-in.

### Public Cloud Global Load Balancers

Public clouds offer multi-cluster load balancing products to solve the issue of traffic distribution across multiple regions, such as AWS's [Global Accelerator](https://aws.amazon.com/global-accelerator/) and GCP's [External Application Load Balancer](https://cloud.google.com/load-balancing/docs/https). GCP even offers custom [Multi Cluster Ingress](https://cloud.google.com/kubernetes-engine/docs/concepts/multi-cluster-ingress) resources that integrate well with Kubernetes Ingress. However, they also have several problems:

1. Although they support multi-cluster load balancing, all clusters must be within the same public cloud, preventing inter-cloud traffic scheduling.
2. They are not usable in private clouds.

### Karmada Multi Cluster Ingress

Karmada is a multi-cluster orchestration tool that also provides its own multi-cluster traffic routing solution through [Karmada Multi Cluster Ingress](https://karmada.io/docs/userguide/service/multi-cluster-ingress/). This solution involves deploying an ingress-nginx provided by the Karmada community and defining `MultiClusterIngress` within a cluster, but it has several issues:

1. It relies on inter-cluster container network connectivity and CRDs like ServiceImporter and ServiceExporter, which have high requirements.
2. It requires additional management of the ingress-nginx instance that provides GSLB services, including its deployment location, quantity, and distribution, which are operational considerations.
3. The [multi-cluster-ingress-nginx](https://github.com/karmada-io/multi-cluster-ingress-nginx) modified by the community has had little code submission in recent years, raising concerns about its reliability.

### Simple DNS Approach

A simple, manually configured DNS-based solution can work initially. Most DNS providers offer health check features, allowing us to add multiple cluster exit IP addresses to DNS records and configure health checks for fault switching. However, this simple approach has clear limitations as it scales:

1. It lacks good automation; a single cluster might have multiple domain names and Ingress IP combinations, and manually managing their mappings becomes increasingly challenging with scale.
2. DNS providers' health checks are typically based on TCP and ICMP, so they can detect and switch during complete cluster outages. However, partial failures are undetectable, e.g., when multiple Ingresses share an ingress-controller and route traffic by domain name. If the backend instances of one service fail, but other services on the ingress-controller operate normally, the health check will still pass, preventing traffic from being redirected to another cluster.
3. DNS inherently involves caching at various levels, which can delay updates.
4. DNS health checks are a capability provided by the vendor and are not guaranteed to be available from all vendors, especially in private cloud scenarios.

## k8gb's Solution

k8gb's solution also utilizes DNS but addresses the deficiencies of the simple DNS approach through a series

 of clever designs.

The essence of the simple DNS approach's problem is that it does not integrate well with Kubernetes. Dynamic information within Kubernetes, such as new Ingress additions, new domain names, and service health status, cannot be effectively synced to upstream DNS servers. Upstream DNS servers' simple health checks are also inadequate for handling the complex changes within Kubernetes. Thus, a core change in k8gb is that the upstream DNS records are no longer A or AAAA records pointing to a cluster's exit address but are forwarded to a CoreDNS configured within the cluster for DNS resolution, pushing the complex DNS logic down into the cluster for internal control. In this way, the upstream DNS only needs to serve as a simple proxy without needing to configure health checks or dynamically adjust multiple address mappings.

The user's DNS request process is illustrated below:

- ![K8GB multi-cluster interoperability](https://www.k8gb.io/docs/images/gslb-basic-multi-cluster.svg)

1. The user requests a domain name's IP record from an external DNS provider.
2. The external DNS forwards this request to a CoreDNS managed by k8gb within the cluster.
3. k8gb analyzes available Ingress IPs based on the Ingress Spec's domain name, the Ingress Status's Ingress IP, and the health status of backend pods within the cluster, as well as load balancing policies, and returns a usable Ingress IP to the user.
4. The user can then directly access the corresponding Ingress Controller via this IP.
5. If a k8gb-managed CoreDNS in one cluster fails, since the upstream DNS concurrently proxies DNS requests to multiple clusters, another cluster can also return its Ingress IP, allowing the user to choose from multiple available IPs.

This method requires administrators to register several domain name suffixes with the upstream DNS and proxy them to each cluster's CoreDNS. k8gb also provides automation capabilities; once certificates are properly configured, it can automatically analyze the domain names used in Ingress and automatically register them with the upstream DNS, significantly simplifying administrators' tasks.

Another important aspect to note is that each cluster's CoreDNS must not only record its Ingress IPs but also those of other clusters for the same Ingress. If a cluster's corresponding service's pods are Not Ready, CoreDNS will return NXDomain. If the client receives this response, it will treat it as if the domain cannot be resolved, even though another cluster's service might still be operational. Therefore, k8gb must also sync all clusters' Ingress IPs for the same domain name and their health status.

As is well known, syncing data across multiple clusters is a global challenge, but k8gb cleverly achieves data syncing through DNS.

The syncing process is illustrated below:

![k8gb multi-cluster interoperability](https://www.k8gb.io/docs/images/k8gb-multi-cluster-interoperabililty.svg)

The general idea is that each cluster's k8gb registers its CoreDNS's Ingress IP with the upstream CoreDNS, allowing each cluster to directly access another cluster's CoreDNS. Then, each cluster's CoreDNS maintains its own `localtargets-app.cloud.example.com` Ingress IP and health status for `app.cloud.example.com`, allowing each cluster to obtain other clusters' Ingress IPs for the same domain name and include them in their return results, achieving multi-cluster domain name resolution syncing.

## Summary

k8gb, as an open-source GSLB, achieves seamless integration with Kubernetes, enabling effective cross-cluster domain name and traffic management with minimal external dependencies—only requiring an API to add DNS records. Although this project is not yet widely popular, it is already the best solution in this field in my view.

> This post was originally written in Chinese and I translated it to English with the help of GPT4. If you find any errors, please feel free to let me know.