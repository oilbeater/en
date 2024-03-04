---
title: "The Single-Node Kubernetes Showdown: minikube vs. kind vs. k3d"
date: February 22, 2024
tags: [kubernetes, ci]
---

As a developer in the cloud-native ecosystem, a common challenge is the need to frequently test applications within a Kubernetes environment. In CI, this often extends to quickly trialing configurations across various Kubernetes clusters, including single-node, high-availability, dual-stack, multi-cluster setups, and more. Thus, the ability to swiftly create and manage Kubernetes clusters on a local machine, without breaking the bank, has become a must-have. This post will dive into three popular single-node Kubernetes management tools: minikube, kind, and k3d.Highlighting their unique features, use cases, and potential pitfalls.

> TL;DR
> 
> If speed is your only concern, k3d is your best bet. If you're after compatibility and a simulation close to reality, minikube is your safest bet. kind sits comfortably in the middle, offering a balance between the two.

- [Technical Comparison](#technical-comparison)
  - [minikube](#minikube)
  - [kind](#kind)
  - [k3d](#k3d)
- [Performance Showdown](#performance-showdown)
  - [Methodology](#methodology)
  - [Results](#results)
- [Conclusion](#conclusion)


# Technical Comparison

At their core, these three tools serve a similar function: managing Kubernetes on a single machine. However, their differing historical backgrounds and technical choices have led to unique nuances and use cases.

## minikube

[minikube](https://minikube.sigs.k8s.io/docs/) is the Kubernetes community's OG tool for quickly setting up Kubernetes locally, a first love for many Kubernetes novices. Initially, it simulated multi-node clusters via VMs on your local machine, offering a high-fidelity emulation of real-world scenarios, down to the OS and kernel module level. The downside? It's a resource hog, and if your virtualization environment doesn't support nested virtualization, you're out of luck, not to mention it's slow to start. Recently, the community introduced a Docker Driver to mitigate these issues, though at the cost of losing some VM-level emulation capabilities. On the bright side, minikube comes with a plethora of add-ons, like dashboards and nginx-ingress, for easy community component installation.

## kind

[kind](https://kind.sigs.k8s.io/) is a more recent favorite for local Kubernetes deployment, using Docker containers to simulate nodes and focusing purely on Kubernetes standard deployments, with community components requiring manual installation. It's the go-to for Kubernetes' own CI processes. The upside? Quick starts and a familiar environment for Docker veterans. The downside? Container simulation lacks OS-level isolation, sharing the host's kernel, which can complicate OS-specific testing. I once had a kernel module test fail because the host's netfilter tweaks caused havoc in a kind-managed cluster.

## k3d

[k3d](https://k3d.io/stable/), a featherweight in local Kubernetes deployment, shares a similar approach to kind but opts for deploying a lightweight [k3s](https://k3s.io/) instead of standard Kubernetes. This means it inherits k3s's pros and cons, boasting incredibly fast setup times—don't worry about correctness; just marvel at the speed. The trade-offs include a super-slimmed-down OS (sans glibc), complicating certain OS-level operations, and a unique installation approach that might puzzle those accustomed to kubeadm's standard deployment features.

# Performance Showdown

While the minikube community provides some [performance benchmarks](https://minikube.sigs.k8s.io/docs/benchmarks/timetok8s/v1.32.0/), comparing the startup times of our three contenders, I was curious about other aspects like image size, memory footprint, and bare-minimum setup times, prompting another round of tests.

## Methodology

Testing was straightforward, given each tool's one-liner setup, with a few caveats:

1. minikube used the Docker Driver to keep the speed test fair.
2. All tests assumed pre-downloaded images to exclude network delays.
3. The latest versions were tested, though Kubernetes versions varied, making this more of a qualitative than quantitative analysis.
4. Tests focused on basic component starts without additional plugins, ensuring essentials like CNI, CoreDNS, and CSI were included.
5. `docker image` and `docker stat` commands were used to measure image sizes and memory usage, respectively.

Commands used:

```bash
#minikube
time minikube start --driver=docker --force

#kind
time kind create cluster

#k3d
time k3d cluster create mycluster --k3s-arg '--disable=traefik,metrics-server

@server:*' --no-lb
```

## Results

| Name | Software Version | Kubernetes Version | Image Size | Start Time | Memory Usage |
| ---- | ---------------- | ------------------ | ---------- | ---------- | ------------ |
| minikube | v1.32.0 | v1.28.3 | 1.2GB | 29s | 536MiB |
| kind | v0.22 | v1.29.2 | 956MB | 20s | 463MiB |
| k3d | v5.6.0 | v1.27.4 | 263MB | 7s | 423MiB |

Evidently, k3d takes the crown in startup performance, boasting significant advantages in image size, startup time, and memory usage. It's a godsend for those running CI on a shoestring budget.

# Conclusion

If speed and resource efficiency are your top priorities, k3d is a no-brainer. For OS-level isolation tests, minikube's VM Driver is unbeatable. For everything in between, kind offers a balanced compromise between compatibility and performance.