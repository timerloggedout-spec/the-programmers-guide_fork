# Claude Models

## About

Claude is not one fixed model. Anthropic releases a family of Claude models, each with a different balance of capability, speed, context length, and cost.

That matters because we do not need the most powerful model for every task. A quick classification or rewrite may need a fast, lower-cost model. A difficult coding task or a long-running agent may justify a more capable one.

## The Usual Trade-Off

When we pick a model, we are balancing four things:

| We care about  | What it means in practice                                                 |
| -------------- | ------------------------------------------------------------------------- |
| **Capability** | How well the model handles difficult reasoning, code, and multi-step work |
| **Speed**      | How quickly we get a useful answer                                        |
| **Cost**       | What we spend on input and output tokens when using the API               |
| **Context**    | How much information the model can work with in one request or session    |

There is no universally “best” model. There is only the best fit for the job in front of us.

## The Model Families

Anthropic commonly offers models in three practical categories:

* **Frontier or premium models** are for the hardest work: complex reasoning, deep code changes, high-stakes analysis, and long-running agent tasks.
* **Balanced models** aim to give us strong quality without paying the highest cost or waiting the longest.
* **Fast models** are useful when we need quick responses at scale, such as classification, extraction, routing, short summaries, and simple transformations.

The exact model names, prices, and limits change over time. We should check the official model table before hard-coding a model name or cost into a production system.

{% hint style="info" %}
A useful default: start with a balanced model. Move up only when quality or reliability is not good enough; move down when the task is simple and cost or latency matters more.
{% endhint %}

## What All Current Claude Models Have in Common

The capabilities available to a specific model can vary, but current Claude models generally support:

* text input and text output;
* image input and visual analysis;
* multilingual work;
* long-context tasks;
* tool use when we provide tools through the API or a supported product.

We should still check the documentation for a particular model before relying on a feature. Context windows, output limits, thinking modes, availability, and pricing are model-specific.

## Model Versions Matter

Model names can look interchangeable, but versions are important.

A named model version is a specific snapshot with its own behaviour, limits, and pricing. If we build an application around a model, pinning to a documented version helps us test and release with predictable behaviour. If we use a moving alias instead, we should expect improvements—but we should also retest the workflows that matter.

For personal use in Claude.ai or Claude Code, we can usually choose a model based on the task. For an API integration, we should make the choice explicit and keep it under review.

## A Few Practical Examples

| Task                                                         | A sensible starting point                                |
| ------------------------------------------------------------ | -------------------------------------------------------- |
| Summarise meeting notes or extract fields from a document    | Fast model                                               |
| Draft a technical explanation or review a small pull request | Balanced model                                           |
| Investigate a tricky production bug across several files     | Premium model                                            |
| Run a workflow that makes many simple requests               | Fast model, with evaluation checks                       |
| Build an autonomous coding or research workflow              | Balanced or premium model, depending on the failure cost |

## What We Should Learn Next

This page gives us the vocabulary. **Model Selection** turns it into a decision process: how we choose a model for a specific workload, measure whether it is good enough, and avoid paying for more capability than we need.

## Official Documentation

* [Anthropic: Models overview](https://platform.claude.com/docs/en/about-claude/models/overview)
* [Anthropic: Choosing a model](https://platform.claude.com/docs/en/about-claude/models/overview#choosing-a-model)
* [Anthropic: Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
