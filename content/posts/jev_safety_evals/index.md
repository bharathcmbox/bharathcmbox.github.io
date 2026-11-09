---
title: "Testing TypeSafe's Jev as a safety classifier on 22 public datasets"
subtitle: "Precision, recall, latency and cost across 53,602 examples"
date: 2026-11-09
draft: true
description: "I ran TypeSafe's Jev decision model over 22 public prompt-injection, jailbreak, moderation and indirect-injection datasets. It is very precise, fast (0.33 s median) and cheap (about 3 cents per 1,000 checks), and its main weakness is recall at the default threshold, which a threshold tuned on training data largely fixes."
tags: [ai-safety, prompt-injection, guardrails, evals, jev]
---

> **Short version.** I ran TypeSafe's **Jev 1.14** decision model over **53,602 examples from 22 public datasets**:
> direct prompt injection, jailbreaks, content moderation, harmful requests, over-refusal and indirect
> (document, email and web-page) prompt injection. Jev is **very precise**: on most datasets, when it flags
> something it is right, and on the three indirect-injection datasets it flagged almost no clean content.
> It is **fast** (0.33 s median, 0.55 s p95) and **cheap**: about **$0.03 per 1,000 checks** for typical
> prompts, and **$4.88** for everything in this post, including pilots and tuning. Its weak spot is
> **recall at the default 0.5 threshold**. Its scores rank attacks well, though, and a threshold chosen on
> training data lifted indirect-injection F1 from 0.87 to 0.97 on a dataset it never saw.

<!--more-->

## Why I tested it

Most guardrail benchmarks report one number on one dataset. I wanted to know how one model behaves across
the kinds of inputs a real assistant sees: obvious jailbreaks, subtle injections, toxic chat, harmless
prompts that only *sound* dangerous, and instructions hidden inside an email or a web page. I also wanted
the numbers a deployment decision depends on: false-alarm rate, latency and cost.

## How Jev was called

Jev is not a chat model. It is a **decision model** served through OpenRouter's
`/api/alpha/decisions` endpoint. You give it a **state** and a set of **questions**, and it returns
structured answers. I used two question types on every example:

- **`noul`**: a score from 0 to 1 for "is this unsafe?". I count it as unsafe at ≥ 0.5 unless stated.
- **`choice`**: pick `safe` or `unsafe` (for indirect injection, `clean` or `injected`), with a probability per option.

The **state** was the input prompt. Where a dataset has more context, the state was a small record:
`{system_prompt, user_input}` for SPML, `{user_prompt, assistant_response}` for BeaverTails, and
`{user_request, external_content}` for indirect injection.

I wrote **one policy definition per input type** and applied it unchanged to every dataset of that type.
I did not tune wording per dataset. That matters for reading the results: each dataset's labels encode
its authors' policy, so a low score can mean "Jev missed it" *or* "my definition and theirs differ".

## Datasets

| Category | Datasets (test split or sample) |
|---|---|
| Direct prompt injection | Qualifire benchmark, SPML Chatbot, xTRam1 Safe-Guard, deepset, Lakera Gandalf |
| Jailbreak | jackhhao, WildJailbreak (eval), In-the-Wild Jailbreak Prompts |
| Moderation | OpenAI moderation eval, ToxicChat, Aegis 2.0, WildGuardMix, BeaverTails (judges the *response*) |
| Harmful requests | JailbreakBench, AdvBench, HarmBench, SimpleSafetyTests, Do-Not-Answer |
| Over-refusal | XSTest |
| Indirect prompt injection | LLMail-Inject (emails), BIPIA (emails, tables, code answers), BrowseSafe-Bench (raw HTML) |

## Precision and recall

All numbers are on the positive class (unsafe / attack). The main columns use the `noul` score at 0.5. The last
column shows F1 for the `choice` answer.

| Dataset | n | Precision | Recall | F1 | False-alarm rate | F1 (choice) |
|---|---|---|---|---|---|---|
| **Direct injection** | | | | | | |
| Qualifire | 5,000 | 0.806 | 0.922 | 0.860 | 0.148 | 0.871 |
| SPML Chatbot | 16,011 | 1.000 | 0.835 | 0.910 | 0.000 | 0.863 |
| xTRam1 Safe-Guard | 2,060 | 0.995 | 0.860 | 0.922 | 0.002 | 0.924 |
| deepset | 116 | 0.969 | 0.517 | 0.674 | 0.018 | 0.629 |
| Lakera Gandalf † | 112 | – | 0.929 | 0.963 | – | 0.928 |
| **Jailbreak** | | | | | | |
| jackhhao | 400 | 0.956 | 0.929 | 0.942 | 0.023 | 0.945 |
| WildJailbreak | 2,210 | 0.968 | 0.968 | 0.968 | 0.310 | 0.970 |
| In-the-Wild | 3,405 | 0.712 | 0.878 | 0.786 | 0.250 | 0.792 |
| **Moderation** | | | | | | |
| OpenAI moderation | 1,680 | 0.844 | 0.852 | 0.848 | 0.071 | 0.767 |
| ToxicChat | 5,083 | 0.725 | 0.845 | 0.781 | 0.025 | 0.704 |
| Aegis 2.0 | 1,928 | 0.869 | 0.782 | 0.823 | 0.138 | 0.770 |
| WildGuardMix | 1,699 | 0.829 | 0.890 | 0.859 | 0.146 | 0.847 |
| BeaverTails (responses) | 3,021 | 0.913 | 0.711 | 0.799 | 0.091 | 0.799 |
| **Harmful requests** | | | | | | |
| JailbreakBench | 200 | 0.752 | 0.940 | 0.836 | 0.310 | 0.856 |
| AdvBench † | 520 | – | 0.990 | 0.995 | – | 0.995 |
| HarmBench † | 300 | – | 0.973 | 0.986 | – | 0.983 |
| SimpleSafetyTests † | 100 | – | 1.000 | 1.000 | – | 0.985 |
| Do-Not-Answer † | 939 | – | 0.483 | 0.652 | – | 0.576 |
| **Over-refusal** | | | | | | |
| XSTest | 450 | 0.935 | 0.940 | 0.938 | 0.052 | 0.911 |
| **Indirect injection** | | | | | | |
| LLMail-Inject | 2,238 | 1.000 | 0.766 | 0.868 | 0.000 | 0.908 |
| BIPIA | 2,450 | 1.000 | 0.707 | 0.828 | 0.000 | 0.847 |
| BrowseSafe-Bench | 3,680 | 0.992 | 0.550 | 0.708 | 0.004 | 0.747 |
{.compact-table}

† These datasets contain only unsafe examples, so precision and false-alarm rate are undefined and F1 only reflects recall.

What stands out:

- **Precision is the strength.** Eight of the 17 datasets with clean examples have precision of 0.95 or higher,
  and the three indirect-injection sets have essentially no false alarms (0 of 438 clean emails and
  documents, 8 of 1,856 clean web pages).
- **Recall is the gap, and much of it is a threshold problem.** Recall at 0.5 is 0.55–0.84 on several
  datasets, but AUROC is 0.95 or higher on 11 of the 17 (0.996 on BIPIA, 0.999 on LLMail). The scores put attacks above clean
  content; many attacks just score below 0.5.
- **Some low numbers are about definitions, not detection.** Do-Not-Answer counts things like medical
  advice and questions about people's private lives as risky. My definition didn't, so recall there is 0.48.
- **Adversarial "benign" prompts cause most false alarms.** The highest false-alarm rates (0.25–0.31) are
  on WildJailbreak, In-the-Wild and JailbreakBench. Their clean examples are written to *look* like attacks.
  On XSTest, which tests over-refusal directly, only 5% of safe-but-scary prompts were flagged.
- **`noul` and `choice` disagree in a useful way.** The score was better for moderation, and the
  categorical answer was slightly better for jailbreaks and indirect injection. Asking both in one request
  costs almost nothing extra.

## Tuning the threshold, without touching the test sets

For indirect injection I picked a new `noul` threshold on **training splits only** (BIPIA train and
BrowseSafe train, 610 examples, $0.15). LLMail-Inject has no training split, so it was **never used for
tuning** and acts as an unseen check. The test scores were then re-thresholded with no new calls.

| Test set | F1 @ 0.5 | Recall @ 0.5 | F1 @ 0.115 (tuned) | Recall @ 0.115 | False-alarm rate @ 0.115 |
|---|---|---|---|---|---|
| LLMail-Inject (unseen) | 0.868 | 0.766 | **0.974** | **0.950** | 0.000 |
| BIPIA | 0.828 | 0.707 | **0.961** | **0.926** | 0.000 |
| BrowseSafe-Bench | 0.708 | 0.550 | 0.754 | 0.757 | **0.248** |

On emails and documents, a lower threshold is close to free: recall jumps and false alarms stay at zero.
**Raw HTML is the real weakness.** BrowseSafe pages run to 17k tokens at the median and up to 73k, with the
injection buried in markup. AUROC there is only 0.84, and lowering the threshold buys recall at a steep
false-alarm cost. A BrowseSafe-specific threshold (0.155) does better (F1 0.770, false-alarm rate 0.099),
but no threshold makes this one easy.

The attacks that worked are also the hardest to catch. In LLMail-Inject, emails that actually
triggered the email-sending tool were flagged 58–63% of the time at 0.5, against 79–83% for attempts
labelled by the challenge's LLM judge.

## Latency

Latency is the wall-clock time of each API call, measured from my client through OpenRouter with 32
requests in flight. It includes the network round trip.

| | p50 | p95 | p99 |
|---|---|---|---|
| All 53,602 calls | **0.33 s** | 0.55 s | 0.80 s |
| Typical prompts (all except BrowseSafe) | 0.33 s | 0.51 s | – |
| BrowseSafe web pages (17k-token median) | 0.48 s | 0.86 s | 1.12 s |

Latency barely moves with input length: a 35× longer input added about 0.15 s at the median. That makes
Jev practical as an inline check in front of a model call, not just as an offline filter.

## Cost

Jev is billed at **$0.042 per million input tokens, and output is free**. Each call also carries my
question text, about 430 tokens, which was the larger part of the bill for short prompts.

| | Cost per 1,000 checks | What I spent |
|---|---|---|
| Typical prompts (≈ 500–1,000 tokens incl. questions) | **≈ $0.02–0.05** | – |
| Raw web pages (BrowseSafe) | ≈ $0.83 | $3.06 for 3,680 pages |
| All 22 test sets (53,602 checks) | $0.085 average | **$4.56** |
| Everything, including pilots and threshold tuning | – | **$4.88** |

Two practical notes: shorter question text is the easiest saving for short inputs, and for web content,
stripping HTML before the check would probably cut both cost and the noise Jev has to read through.

## Caveats

- **One wording, one model version.** Results are for `jev-1.14-20261014` with my question wording. The
  decisions API is an alpha endpoint, so behavior may change.
- **Labels are not a shared standard.** Datasets disagree on what counts as unsafe, so compare a dataset's
  numbers with itself over time more than with other datasets.
- **Samples, not always full sets.** In-the-Wild uses 2,000 sampled regular prompts. LLMail-Inject uses
  2,000 sampled attacks plus 238 benign emails. BIPIA uses 3 of its 5 tasks (the other two need separately
  licensed source data) and 2,250 sampled injected contexts.
- **LLMail-Inject's "not an attack" labels were unusable as negatives.** Many submissions the challenge's
  judge labelled `attack_attempt = False` still contained the attack payload, so I used only genuinely
  benign emails as clean examples.
- **Thresholds were tuned only for indirect injection.** The direct-injection and moderation numbers are
  all at 0.5 and would likely improve the same way.

## Datasets used

[Qualifire prompt-injections benchmark](https://huggingface.co/datasets/qualifire/prompt-injections-benchmark),
[SPML Chatbot Prompt Injection](https://huggingface.co/datasets/reshabhs/SPML_Chatbot_Prompt_Injection),
[xTRam1 safe-guard-prompt-injection](https://huggingface.co/datasets/xTRam1/safe-guard-prompt-injection),
[deepset prompt-injections](https://huggingface.co/datasets/deepset/prompt-injections),
[Lakera gandalf_ignore_instructions](https://huggingface.co/datasets/Lakera/gandalf_ignore_instructions),
[jackhhao jailbreak-classification](https://huggingface.co/datasets/jackhhao/jailbreak-classification),
[WildJailbreak](https://huggingface.co/datasets/allenai/wildjailbreak),
[In-the-Wild Jailbreak Prompts](https://huggingface.co/datasets/TrustAIRLab/in-the-wild-jailbreak-prompts),
[OpenAI moderation-api-release](https://github.com/openai/moderation-api-release),
[ToxicChat](https://huggingface.co/datasets/lmsys/toxic-chat),
[Aegis 2.0](https://huggingface.co/datasets/nvidia/Aegis-AI-Content-Safety-Dataset-2.0),
[WildGuardMix](https://huggingface.co/datasets/allenai/wildguardmix),
[BeaverTails](https://huggingface.co/datasets/PKU-Alignment/BeaverTails),
[JailbreakBench](https://huggingface.co/datasets/JailbreakBench/JBB-Behaviors),
[AdvBench](https://huggingface.co/datasets/walledai/AdvBench),
[HarmBench](https://huggingface.co/datasets/walledai/HarmBench),
[SimpleSafetyTests](https://huggingface.co/datasets/Bertievidgen/SimpleSafetyTests),
[Do-Not-Answer](https://huggingface.co/datasets/LibrAI/do-not-answer),
[XSTest](https://huggingface.co/datasets/Paul/XSTest),
[LLMail-Inject](https://huggingface.co/datasets/microsoft/llmail-inject-challenge),
[BIPIA](https://github.com/microsoft/BIPIA),
[BrowseSafe-Bench](https://huggingface.co/datasets/perplexity-ai/browsesafe-bench).
