---
title: "AI Gateway Survey: Cloudflare AI Gateway"
date: 2024-08-26 03:03:34
tags:
    - AI
    - Gateway
    - Cloudflare
---

With the rising popularity of AI, I noticed that many API gateway products rebrand themselves as AI Gateways. This prompted me to explore what these so-called AI Gateways actually offer. My research includes those that started with API management or cloud-native Ingress Controllers and later integrated AI features, such as: [Kong](https://konghq.com/products/kong-ai-gateway), [Gloo](https://www.solo.io/products/gloo-ai-gateway/), and [Higress](https://higress.io/en/). It also covers gateways that were AI-native from the start, like [Portkey](https://portkey.ai/features/ai-gateway) and [OneAPI](https://github.com/songquanpeng/one-api), as well as the public cloud Serverless-based [Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/) discussed in this blog.

Generally, current AI Gateways excel in three main areas:

**Standard API gateway features applied to AI APIs** such as monitoring, logging, rate limiting, reverse proxy, request or response rewriting, and user system integration. These features, although crucial, are not specifically AI-related; they treat LLM APIs like any standard API Service.

**API gateway features optimized for AI** include enhancements like token-based rate limiting, prompt-based caching, firewall filtering based on prompts and LLM responses, load balancing across multiple LLM API keys, and API translation among different LLM providers. These functionalities extend existing API gateway concepts for AI scenarios.

**New features added for AI applications** include embedding and RAG capabilities, exposing vector and text database functionalities via APIs. There are also cost optimizations related to token usage, such as prompt simplification and semantic caching. Additionally, some gateways offer application-layer features, like scoring the outputs of LLMs.

This blog post highlights the features of the [Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/).

# Basic Principle

Cloudflare's AI Gateway mainly functions as a reverse proxy. After reviewing it, I realized that I could potentially replicate similar functionalities using Cloudflare Worker. If you're currently using OpenAI's API, you simply need to change the SDK's baseURL to `https://gateway.ai.cloudflare.com/v1/${accountId}/${gatewayId}/openai`. Through this setup, Cloudflare can offer monitoring, logging, and caching as traffic goes through their platform.

This approach has several advantages:
- Easy integration by changing the baseURL, with no change in API format. It is entirely Serverless and free, effectively giving away monitoring capabilities.
- Leveraging Cloudflare's global network can accelerate user access, though this is minimal compared to the latency of LLMs themselves. The most noticeable speed improvement may be in the latency of the first token response.
- It can obscure the source IP, useful for bypassing regional restrictions on certain OpenAI API accesses.

However, there are also drawbacks:
- All request data, including API keys, pass through Cloudflare, posing potential security risks.
- The gateway lacks a plugin mechanism, making it challenging to extend functionalities without additional external layers.
- Constantly changing IP addresses through Cloudflare's network might trigger security measures from OpenAI.

# Key Features

## Multi-provider Support

Since Cloudflare AI Gateway simply acts as a reverse proxy without altering the LLM API, it can support almost any mainstream LLM API by changing the baseURL to the format: `https://gateway.ai.cloudflare.com/v1/${accountId}/${gatewayId}/{provider}`.

It also offers a [Universal Endpoint](https://developers.cloudflare.com/ai-gateway/providers/universal/) for simple fallbacks, allowing multiple provider queries in one API request to automatically call the next provider if the previous one fails.

## Observability

In addition to basic monitoring like QPS and error rates, Cloudflare provides specific dashboards for tokens, costs, and cache hit rates for LLM scenarios.

The logging is similar to that of Workers, focusing only on real-time logs without historical data access. This limitation makes it difficult for AI applications that rely on analyzing request and response logs for optimizations or fine-tuning.

## Caching

Cloudflare’s caching is still based on exact text matches, likely implemented using [Cloudflare Workers KV](https://developers.cloudflare.com/kv/). Custom cache keys and settings, including TTL and cache bypass, are possible, but semantic caching is not yet available, although promised for the future.

## Rate Limiting

Cloudflare's rate limiting still follows traditional QPS-based methods without any AI-specific enhancements, like token-based limiting, which could be improved in the future.

## Custom Metadata

Custom headers can be added to requests, such as user information, retrievable through logging.

# Conclusion

Overall, Cloudflare AI Gateway excels in simplicity and ease of use. New users can integrate within minutes, providing essential monitoring and caching capabilities. Despite its straightforward implementation, it lacks depth in more advanced features, and extending functionalities is cumbersome, requiring additional setups with Workers. A potential improvement could be open-sourcing the AI Gateway as a template, allowing users to modify code or create plugins to build a new ecosystem, considering it likely operates similarly to a Worker template.