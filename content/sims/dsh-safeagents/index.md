---
title: "Simulator: one attack, four agent setups"
description: "Step through the same prompt-injection attack on a typical harness, DeepSeek Harness with default settings, DSH with Hugging Face SafeAgents, and a hardened DSH deployment."
date: 2026-11-08
draft: true
layout: simulator
build:
  publishResources: false
---

The same attack, run against four agent setups. Pick a setup, then step through with **Next** or the
arrow keys. The step stays the same when you switch setups, so you can compare one moment across all four.

> This is an illustrative simulation based on a code review of
> [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness/tree/fcc0ad462172522aaf5c93a7b2e01f6b9f0b0159)
> (commit `fcc0ad46`) and [Hugging Face SafeAgents](https://github.com/huggingface/safeagents/tree/0de8e22d2345b2379779161abbbffa72738a78a2)
> (commit `0de8e22`). It doesn't run either tool. The full write-up is in
> [DeepSeek Harness: an architecture and security review](/posts/dsh_secreview/).
