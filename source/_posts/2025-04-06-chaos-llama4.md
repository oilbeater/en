---
title: Chaos in Llama 4
date: 2025-04-06 15:57:23
tags: 
- LLM
- Llama
---

A while ago, the departure of Meta AI's head raised suspicions about issues with the progress of Llama 4. However, just two days later, Meta released Llama 4, seemingly to dispel rumors. Yet, after reviewing the basic information of the models that have been published, I feel that the Llama project is now in extreme disarray. Below is my analysis based on the available information, and I welcome any corrections.

## Model Basic Information

Llama has officially released a 109B and a 400B parameter model this time, along with information about an unreleased 2T parameter model. I have summarized the key architectural details below. All information is sourced from Llama's blog and the huggingface model page:

| Name     | Parameters | Activated Parameters | Experts | Context | Corpus Size | GPU Time |
| -------- | ---------- | -------------------- | ------- | ------- | ----------- | -------- |
| Scout    | 109B       | 17B                  | 16      | 10M     | 40T         | 5.0M     |
| Maverick | 400B       | 17B                  | 128+1   | 1M      | 22T         | 2.38M    |
| Behemoth | 2T         | 288B                 | 16      | -       | -           | -        |

There's no need to look at the model's scoring performance anymore. The comparison of these models' architectures reveals many issues.

## Strange MoE Architecture

Llama 4 has fully transitioned from Dense models to MoE this time, but the bizarre part is that they have adopted two different MoE architectures for the three models. The largest Behemoth and the smallest Scout use the traditional MoE, with 16 experts—a number traditionally considered conventional. The middle Maverick, however, uses a new fine-grained expert plus shared expert model proposed by DeepSeek MoE, with a 128 experts plus 1 shared expert architecture.
Generally, models in the same generation use the same architecture, with adjustments made to the number of layers and the width of each layer. Having two models with significant differences in the same generation is odd. Moreover, even if there are changes, it shouldn't be the largest and smallest models that remain consistent, while the middle-sized one is switched. It feels like the middle Maverick was hastily retrained under the impact of DeepSeek, but there wasn't enough time to redo all three, so they were just released together.

## Strange Cost Investment

Typically, the larger the model's parameter scale, the higher the cost required. On one hand, larger models can accommodate more knowledge and require more corpus for larger models; on the other hand, with the same corpus, larger models require more parameters to be trained, leading to higher GPU costs. Therefore, usually, the larger the model scale, the higher the required cost.
However, in Llama 4, both indicators are reversed. Maverick's parameter scale is nearly four times that of Scout, but Maverick's training corpus size is only half of Scout's, and the consumed GPU time is also only half. Considering that the activated parameters of these two models are the same, the GPU time can be understood, but the corpus size being only half is very strange. It feels like either this is a trial of the new MoE architecture without intending to do a full training, or the training broke down later, and they had to release from an intermediate snapshot.

## Strange Context Length

Generally, larger models have stronger capabilities. However, in this generation of Llama, the most shocking 10M context is given to the smallest Scout, while the larger Maverick has only 1M context. Considering the mainstream method for expanding context is still post-training fine-tuning, the larger Maverick's post-training investment is less than the smaller Scout.

## Conclusion

It feels like Llama 4 was initially intended to follow the traditional MoE path but was influenced by DeepSeek and halfway started to look at DeepSeek MoE. However, training might have already begun, and stopping it would have been difficult, so a middle-sized Maverick was inserted. Judging from the parameter selection, the goal seems to be to achieve similar performance to DeepSeek V3 with a smaller parameter count. However, with 17B activated parameters, it seems challenging to match DeepSeek V3's 39B activated parameters. Nevertheless, the fact that this generation's models were released in such a chaotic form, along with a futures model, suggests that there are significant issues within the Llama project.

![](../images/llama4.png)