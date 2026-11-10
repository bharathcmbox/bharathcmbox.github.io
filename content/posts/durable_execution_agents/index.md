---
title: "Long-horizon agents are distributed systems"
date: 2026-11-10
draft: false
description: "A short commentary on 'Long-horizon agents and durable execution' on martinfowler.com, and the specifics worth focusing on once you treat an agent as a distributed system."
tags: [agents, durable-execution, distributed-systems, architecture]
---

> **Short version.**
> [Long-horizon agents and durable execution](https://martinfowler.com/articles/durable-execution-agents.html)
> on martinfowler.com is one of the clearest things I've read on what happens when an agent's task takes
> weeks instead of minutes. Its core idea: once an agent waits days for a person or a supplier, it is no
> longer a chat loop. In the author's own words, *"These agents are distributed systems,"* and the old
> distributed-systems patterns apply. Below
> are the parts I think deserve the most attention. Read the original; this is only a guide to it.

<!--more-->

## The setup

The article follows one example the whole way through: **Stock Replenisher**, an agent that notices a
part is below safety stock, gets quotes, asks a procurement manager to approve the price, places the
order and tracks it to delivery. That can take days or weeks. In that time processes crash, APIs fail,
people take their time, and code gets deployed.

The article sorts the resulting failures into a table, and makes one point that sets up the rest:
*"The first four failures are the same in any domain."* A crashed process, a failed call, a repeated
retry and two runs competing for the same work have nothing to do with AI. Runtimes such as Temporal,
Restate, Inngest, DBOS and Cloudflare Workflows already handle them. The model is what makes an agent
different, and the harness's job is to give the model the right input when something goes wrong.

## What to focus on

**1. Decide where the step boundaries are.** Durable execution means *"the runtime keeps a record of the
result of each step that interacts with the outside world."* A step is any outside call, and that
includes the LLM call. Code between steps only reads and writes local state. After a crash the program
re-runs from the top: recorded steps return their logged result, the code in between rebuilds the
state, and the first step with no record runs for real. This is a write-ahead log plus event sourcing.
In practice it means the most important design choice is which calls are steps. In the article, the
model's decision and each tool's side effect are steps; the business rules and state updates are not.

**2. Treat waiting as normal.** Thousands of parts means thousands of agents, nearly all of them idle
and waiting for an approval or a reply. The runtime must not keep a process in memory or use compute
while they wait, and the timers have to survive crashes too. I liked how timeouts are handled: when a
supplier hasn't replied after seven days, the timeout updates the state, and that makes a
"chase supplier" tool available to the model. A timeout is something the model reasons about, not an
exception.

**3. Retries are at-least-once, so idempotency is your job.** *"What the runtime can't do is make a
retry safe."* If the supplier accepted the order and only the reply was lost, a retry places a second
order. The fix is an idempotency key that stays the same on every attempt **and on every replay**.
In the example that key is the PO number. This is the one I would underline: a harness that generates
a fresh request ID per attempt quietly undoes everything the runtime gives you.

**4. The log records what the agent saw, not what happened.** I think this line is the most important
in the piece. When a call times out, the agent can't tell "failed" from "succeeded but I never heard
back." In the article's demo the agent gives up on supplier A, reorders from supplier C, then later sees
A's confirmation and cancels the duplicate using a `CancelOrder` tool. That is a saga: each risky step
needs a compensating step. Durable execution makes the agent recoverable. It does not make it correct.

**5. Separate the entity from the process.** There are two kinds of workflow: a long-lived **entity**
(one per part, forever) and a short-lived **process** (one replenishment, then done). The entity has a
stable address for incoming messages such as the manager's approval, and it makes sure only one
replenishment runs at a time. This is the actor model, made durable. It also answers concurrency
without locks scattered through the agent's code.

**6. Keep rules in code, not in the prompt.** Each tool has an `allowed` check. `PlaceOrder` is only
offered if the price is within what the manager approved. The model picks among the tools it is
allowed to use; it never decides whether a rule applies. Because the world changes during a long wait,
decisions need an expiry date, and facts need re-checking before anything hard to undo: the
optimistic-lock idea, applied to an approval that may be days old. The article adds a neat third
pattern: approve a range, not a value. The manager approves a price and up to 5% above it, and the
order tool re-fetches the quote and enforces that ceiling itself, whatever the model decides. The same
code-level checks are the agent's defence against injected text. Durable execution does nothing
against prompt injection. But a supplier email claiming *"the manager has already approved $6.00"*
approves nothing, because only the manager can approve and the model can only pick allowed tools.

**7. The log is a published interface, so version your code against it.** Deploying the harness
changes the code that thousands of paused workflows will replay. The article's experiment is
striking: a rule change (four quotes before drafting an order, up from three) was deployed while an
order waited for approval. On Temporal the order went ahead under the old rule, which the new rule
doesn't allow; on Cloudflare it got stuck as a draft. Same change, different failure. The fix is
Parallel Change: each run records the rules version it started under, old and new rules live side
by side until no running workflow needs the old one, and you learn how your runtime behaves *before*
you deploy.

**8. History grows, and so does replay cost.** Replay gets slower as the log gets longer. The article
reports measured wake-up times on both Temporal and Cloudflare, and points to snapshots and Temporal's
`continue_as_new` for long-lived entities; short process workflows avoid the problem because they end.
What the model sees is a separate choice from what the runtime logs. An agent that runs for years
needs tiered memory (the MemGPT/Letta idea), and every read and write to that memory is itself a step.

## Why this matters

Most agent discussion is about the model: which one, which prompt, how many tokens. This piece is a
useful correction. For work that runs for weeks, most of the hard problems are old ones: logs, retries,
idempotency, compensation, actors, stale reads and versioning. The conclusion goes further: in
practice most of the effort goes into systems of record that disagree with each other and were never
built for an agent to write to. The article's sidebar on essential versus accidental
complexity says it well: durable execution removes a class of accidental problems, but the uncertainty
of the real world stays, and the design has to make that uncertainty visible to the model.

Kiran Prakash's original has working code for both Temporal and Cloudflare, a table that maps the primitives
between the two, and many more details than I have covered.
[Read it in full](https://martinfowler.com/articles/durable-execution-agents.html).
