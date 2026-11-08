---
title: "Never trust the agent. How Axiom, Muse, and OpenShell draw the line"
subtitle: "Part 1: Axiom, Meta's Muse and NVIDIA OpenShell, compared"
date: 2026-11-07
draft: true
description: "A short comparison, for security leaders, of three agent security designs: my learning project Axiom, Meta's Muse personal agent, and NVIDIA's OpenShell with Sentry. What each assumes, where each draws the boundary, and what each leaves open."
tags: [ai-agents, security, threat-modeling, agent-harness, governance]
---

> **Part 1 of 2.** This is the short version: one table and the takeaways. Part 2 will go system by
> system in detail. A disclosure up front: [Axiom](https://github.com/bharathcmbox/axiom) is my own
> learning project, not a product, and it has run only a handful of real tasks. Muse and OpenShell
> are described from their public documentation as of November 2026.

<!--more-->

## The shared starting point

All three designs start from the same assumption: **the agent will eventually be tricked, so the
security can't depend on the model behaving.** Prompt injection is still an open problem, and none of
them claims otherwise. Where they differ is in what they protect, where they draw the boundary, and who
gets to say yes.

- **Axiom** runs a small team of coding agents against one GitHub repository. Its job is to keep a
  misled agent from shipping or stealing anything. [Repo](https://github.com/bharathcmbox/axiom).
- **Meta's Muse** is a consumer personal agent with access to a user's email, calendar and payments.
  Its job is to protect one person's data and money.
  [Meta's write-up](https://research.meta.ai/blog/security-and-safety-for-ai-agents-our-approach-with-muse).
- **NVIDIA OpenShell** is an open-source (Apache 2.0) sandbox runtime and policy layer for running
  fleets of agents. **Sentry** is NVIDIA's companion running on BlueField-4 DPU hardware that watches
  from outside the host. Its job is general-purpose containment for whatever agent you run.
  [Repo](https://github.com/NVIDIA/OpenShell), [Sentry](https://www.nvidia.com/en-au/networking/products/software/sentry/).

## Side by side

| | Axiom | Meta Muse | NVIDIA OpenShell + Sentry |
|---|---|---|---|
| **What it protects** | A code repository and the keys to it | One user's data, accounts and money | Whatever infrastructure the agent runs on |
| **Treated as untrusted** | The model, task text, repository files | All external content entering the model's context | Everything inside the sandbox; Sentry also assumes the host may be compromised |
| **Isolation** | Fresh container per run; only a separate launcher service can start containers | `systemd-nspawn` cell on a per-user VM; sensitive services run outside it | Containers, microVMs or Kubernetes, plus Landlock, seccomp and a separate network namespace |
| **Can the agent see a real credential?** | No. Model access goes through a proxy using short-lived, budgeted run tokens; the GitHub token stays with a separate worker | No. Surrogate tokens get swapped for real ones at the network boundary | No. Opaque placeholders get swapped by the proxy, only for authorized hosts and paths |
| **Network egress** | Allowlist proxy; the sandbox has no route to GitHub at all | Every request decided by "Sentinel" at the connection and HTTP layers; reading user data removes auto-allow | Deny by default; rules tied to a specific binary; optional HTTP-level rules (these only log by default) |
| **Who says yes** | Deterministic code, independent reviewer agents, CI, then a human merges | Sentinel grants scoped, expiring capabilities; the user approves in the app UI, not in chat | Operator-written YAML policy; agents can *propose* new network rules, and a human reviews them by default |
| **Prompt-injection stance** | Assume the model is compromised; verify its output (allowed paths, secret scan, re-run tests) | Defense in depth: model training, labeling untrusted input, classifiers, and human approval for anything leaving the VM | Contain the damage; no content filtering |
| **Audit** | Append-only event log; costs metered by the proxy | Not described for today's product | Exports in OCSF format to your SIEM ("best-effort"); Sentry adds telemetry the host can't tamper with |
| **Biggest stated gap** | GitHub itself: a compromised worker could push to any repository its token reaches | Meta can still access user data under policy; the confidential-VM version isn't out yet | Each allowed endpoint is an exfiltration path; the sandbox doesn't scrub what the app outputs |

## What security leaders should take from this

**1. "The agent never holds the real key" is now the baseline.** All three reached the same pattern on
their own: the agent gets a stand-in token, and something it can't touch swaps in the real credential
at the boundary. If a vendor's agent can run `env` and see a live API key, it is behind.

**2. The real differences are in who approves.** Axiom puts approval in deterministic code and a human
merge. Muse turns approvals into scoped capabilities granted in a UI the agent can't reach. OpenShell
puts it in version-controlled policy the operator owns. Ask any vendor: *can the agent approve its own
request, or talk the approver into it?* Muse is explicit here: approval prompts don't appear in the chat,
so the agent can't answer them.

**3. Containment beats detection, but containment has holes you allowed.** OpenShell says plainly that
every allowed endpoint is a possible exfiltration path. Axiom's biggest remaining gap is GitHub, the one
destination it *must* allow. Your agent's risk is roughly the sum of what you allowlisted. Review that
list like a firewall ruleset.

**4. The harness itself is the high-value target.** In Axiom, the most serious finding wasn't the model
misbehaving. It was a path out of the sandbox through git configuration the agent could write, which
our own trusted code then executed. Muse moves its sensitive services out of the agent's cell, and
Sentry watches from separate hardware for the same reason. Whatever reads the agent's output is part
of your attack surface.

**5. Watch the defaults.** OpenShell's HTTP-level rules only log violations by default. Muse lets
low-risk reads through without a prompt. Axiom lets the sandbox reach PyPI. Each is a reasonable choice,
and each is where an assessment should start.

**6. Nobody has solved shared state between agents.** All three support multiple agents or subagents.
None of them publishes much about how trust works when agents share memory, files or tools. That's the
gap I'd watch most closely as self-improving harnesses (see my [Exo post](/posts/exo_analysis/)) let
agents rewrite the tools other agents load.

## Three questions to ask any agent vendor

1. Where exactly is the boundary, and what runs on each side of it? Ask for the diagram.
2. Show me the path a credential takes. At what point, if any, could the model read it?
3. Who or what approves a new destination, a new permission or a merge, and can the agent reach that approver?

## Coming in part 2

A deeper look at each design: how credentials flow, how each one fails, what they log, the drills I ran on
Axiom, and a threat-by-threat matrix covering injection, exfiltration, privilege escalation, supply chain
and shared state.
