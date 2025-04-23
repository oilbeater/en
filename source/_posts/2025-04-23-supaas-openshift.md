---
title: How to Surpass OpenShift
date: 2025-04-23 10:40:43
tags:
- OpenShift
- Product
---

OpenShift, as a benchmark product in the container platform space and a model of open-source commercialization, has always been a target for others to chase or surpass. However, if you merely imitate OpenShift's product, the only scenario where you might catch up is if OpenShift stops developing. After a year, you might finally catch up, only to be phased out for the same reasons.

So, is there a way to catch up with or even surpass OpenShift? I believe the key lies in examining OpenShift's business model choices and technical roadmap, identifying the inevitable shortcomings that arise from these choices, and then differentiating yourself. Only then is surpassing OpenShift possible.

## OpenShift Is Not Open

The "Open" in OpenShift might be the same as the "Open" in OpenAI. If you've ever tried to deploy OpenShift without going through RedHat's sales team, you’ll know exactly what I mean.

All of OpenShift's components are indeed open-source, but if you're purely a community user, you’ll face obstacles at every step. You might struggle to find the corresponding component for a feature, locate its source code, or even find documentation on how to compile and use it. These issues are commonplace for components exclusively used by OpenShift. It seems OpenShift’s "community" is only open to customers and partners.

For non-OpenShift-specific components, OpenShift’s strategy is to heavily invest once a choice is made, aiming to gain dominance in the corresponding component’s community. As a result, you’ll notice that OpenShift contributors are entirely absent from some well-known community projects, while others that have been steadily developing for years suddenly see an influx of OpenShift contributors.

These are the trade-offs OpenShift has made between commercialization and open-source—there’s no right or wrong here. However, this leaves room for more open projects. If a new product can lower the barrier to entry, gather broader feedback, and enable more contributors to participate in innovation, I believe its potential will surpass OpenShift's.

## OpenShift’s Technology Isn’t Cutting-Edge

Due to the first point, OpenShift cannot widely adopt the latest advancements from the broader ecosystem. When a component in the ecosystem overlaps with OpenShift’s proprietary components, OpenShift’s internal developers have little incentive to switch to another community or an open-source component led by another company.

Take networking, for example—an area I’m familiar with. Early on, OpenShift implemented Route using HAProxy to enable external traffic to access services inside the cluster. At the time, when Ingress was still immature, Route was a far more advanced solution than what the community offered, and OpenShift’s approach was undoubtedly leading. However, as Ingress matured and various gateways in the ecosystem rapidly evolved, OpenShift, constrained by its early implementation and user practices, took a long time to support Ingress, which had already become a standardized feature in the community. Now, the Ingress specification has progressed to GatewayAPI, and many new AI Gateway scenarios are extending through GatewayAPI. Yet, OpenShift still doesn’t support GatewayAPI and is only recently planning to add support for Route, Ingress, and GatewayAPI on its existing HAProxy.

There are many similar cases within OpenShift. Early solutions might have been excellent, but because they were OpenShift-specific and closed, they eventually became obstacles to progress as the community advanced. Today, no one building an Ingress Gateway on Kubernetes would reference OpenShift’s implementation. In many niche areas, OpenShift is no longer the most advanced solution. To me, OpenShift now feels like a broad but mediocre and uninspiring platform.

The production of a container platform is similar to manufacturing a smartphone—it involves selecting components from hundreds of suppliers and assembling them into a final product. If you can remain open, choose the most cutting-edge components from the supply chain, or quickly assemble a product tailored to specific scenarios, your technical competitiveness should far surpass OpenShift’s.

## Conclusion

To truly catch up with or surpass OpenShift, the key isn’t to follow its existing features step by step but to be more open and more advanced. Only then can you break free from the follower’s path, establish a differentiated advantage over OpenShift, and become a leader in the next wave of technology.
