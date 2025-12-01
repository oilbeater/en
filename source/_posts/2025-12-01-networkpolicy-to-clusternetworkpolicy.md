---
title: From NetworkPolicy to ClusterNetworkPolicy
date: 2025-11-30 09:45
created:
  - 2025-11-30
tags:
  - opensource
  - network
---
[NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/), as an early Kubernetes API, may seem promising, but in practice, it proves to be limited in functionality, difficult to extend, understand, and use. Therefore, Kubernetes established the [Network Policy API Working Group](https://network-policy-api.sigs.k8s.io/) to develop the next-generation API specification. [ClusterNetworkPolicy](https://network-policy-api.sigs.k8s.io/api-overview/#the-clusternetworkpolicy-resource) is the latest outcome of these discussions and may become the new standard in the future.

## Limitations of NetworkPolicy

NetworkPolicy is a Namespace-scoped resource, essentially representing an "application-centric" policy. It selects a group of Pods using `podSelector` and `namespaceSelector`, then restricts their Ingress and Egress traffic. This leads to several practical issues:

### Lack of Cluster-Level Control

Since the policy scope is limited to a Namespace, cluster administrators cannot define cluster-wide default network policies. Instead, they must create identical policies in every Namespace. This approach has drawbacks: updating a policy requires modifications across all Namespace resources, and it can easily conflict with network policies created by developers. Policies set by administrators can be easily bypassed by new policies from the application side.

The root cause is the conflict between the application-centric perspective of NetworkPolicy and the cluster-centric perspective of administrators. NetworkPolicy becomes difficult to apply when the user's security model is not application-centric, yet managing overall cluster security policies is a common real-world scenario for administrators.

### Unclear Semantics

NetworkPolicy semantics have several "pitfalls" that both newcomers and operators often encounter:

- **Implicit Isolation**
  Once any NetworkPolicy selects a Pod, traffic that is "not explicitly allowed" is implicitly denied. This implicit behavior requires mental deduction and is hard to understand at a glance.

- **Only Allow, No Explicit Deny**
  Standard NetworkPolicy only supports `allow`-type rules. To "deny a specific source," one typically has to indirectly achieve it by adding other `allow` rules or rely on vendor-specific extensions provided by certain CNI implementations.

- **No Priority**
  When multiple NetworkPolicies select the same set of Pods, their rules are additive rather than overriding. The final behavior often requires examining all policies together, making troubleshooting extremely difficult.

These characteristics combined make NetworkPolicy challenging to understand and even more challenging to debug.

## The ClusterNetworkPolicy Solution

To address the inherent issues of NetworkPolicy, the Network Policy API Working Group proposed a new API—ClusterNetworkPolicy (CNP). Its goal is to provide cluster administrators with clearer and more powerful network control capabilities without breaking existing NetworkPolicy usage.

The core idea is to introduce policy layering. Independent policy tiers are added before and after the existing NetworkPolicy layer, separating cluster administrator policies from application policies. This provides richer perspectives and more flexible usage.

![](../images/20251201104247.png)

An example is as follows:

```yaml
apiVersion: policy.networking.k8s.io/v1alpha2
kind: ClusterNetworkPolicy
metadata:
  name: sensitive-ns-deny-from-others
spec:
  tier: Admin
  priority: 10
  subject:
    namespaces:
      matchLabels:
        kubernetes.io/metadata.name: sensitive-ns
  ingress:
    - action: Deny
      from:
        - namespaces:
            matchLabels: {} 
      name: deny-all-from-other-namespaces
```

### Cluster Administrator Perspective

The newly introduced ClusterNetworkPolicy is a cluster-scoped resource. Administrators can directly select Pods across multiple Namespaces for policy control. Additionally, policies in the `Admin` tier can take effect before NetworkPolicies, allowing cluster administrators to control critical cluster-wide behaviors with just a few `Admin` tier policies.

Policies in the `Baseline` tier are executed after all NetworkPolicies have been evaluated, serving as a fallback policy.

In simple terms:
- Policies with `tier: Admin` define what **must absolutely not be done**.
- Policies with `tier: Baseline` define what is **not recommended by default**, which users can override using NetworkPolicies.

### Explicit Priority

ClusterNetworkPolicy introduces a new `priority` field. When multiple rules within the same tier have overlapping scopes, priority clearly determines which rule should take effect. This eliminates the implicit overriding behavior seen in NetworkPolicy, where rules had to be guessed and inferred.

### Clear Action Semantics: Accept / Deny / Pass

Unlike NetworkPolicy, which only has an "allow" semantic, each rule in ClusterNetworkPolicy has an explicit `action` field with possible values:

- `Accept`: Allows traffic selected by this rule and stops further policy evaluation.
- `Deny`: Denies traffic selected by this rule and stops further policy evaluation.
- `Pass`: Skips subsequent ClusterNetworkPolicies in the current tier and passes evaluation to the next layer.

The documentation emphasizes:
- ClusterNetworkPolicy no longer has the "implicit isolation" effect of NetworkPolicy.
- All behavior stems from the rules you write; what you see in the policy is exactly what you get.

Combined with priority configuration, rule interpretation no longer leads to ambiguity, making understanding significantly less difficult.

## Summary

ClusterNetworkPolicy, to some extent, returns to the traditional layered network policy architecture. It addresses the issues of NetworkPolicy without introducing disruptive changes, making it a well-designed solution. We look forward to seeing this specification mature and be widely adopted soon.