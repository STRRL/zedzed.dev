---
title: "AI Agents Are Not Microservices"
description: "Platform teams adopting AI agent infrastructure keep making the same mistake: treating agents like stateless HTTP services. They're not."
pubDate: "2026-03-08"
heroImage: "../../assets/blog-placeholder-2.jpg"
---

Every few years, a new computing paradigm arrives and the industry spends two years trying to fit it into existing mental models before accepting that it needs its own.

We did it with containers ("just VMs but lighter"), with Kubernetes ("just Mesos but from Google"), and we're doing it again with AI agents.

The mistake: **treating AI agents like microservices**.

## How microservices think

A microservice is stateless, request-scoped, and embarrassingly parallel. You can scale it horizontally. You can kill any instance and another one picks up the next request. The unit of work is a single HTTP request with a response time measured in milliseconds.

This model is so deeply embedded in how platform teams think about infrastructure that it shapes everything: how we do load balancing, how we think about retries, how we size compute, how we write SLOs.

## Why agents are different

An AI agent isn't completing a request. It's pursuing a goal across an unbounded sequence of steps. It has:

- **Multi-step execution** — A single "task" might involve dozens of tool calls, each potentially side-effectful, over minutes or hours.
- **Accumulated context** — The agent's working memory grows throughout the task. Killing and restarting an agent mid-task isn't a free operation — it's like losing your train of thought mid-sentence.
- **Non-deterministic duration** — You cannot predict how long an agent task will take. An agent debugging a flaky test might take 2 minutes or 45 minutes. Your p99 latency SLO doesn't apply.
- **Irreversibility** — The agent may have written files, called APIs, or sent messages. Unlike stateless request handling, partial completion leaves real state in the world.

## The infrastructure implications

If you design agent infrastructure like microservices, you get:

**Timeout-triggered failures.** Your load balancer has a 30-second timeout. Your agent is on step 23 of 40, right in the middle of a complex refactor. Timeout. Work lost. Agent restarted. This loop is expensive and demoralizing.

**Broken retry semantics.** Microservice retries are safe because the operation is idempotent. Agent retries may replay side effects. If the agent already created the GitHub branch and sent the Slack message, retrying from the start creates duplicates.

**Inappropriate scaling models.** Horizontal scale-out works for stateless services because all instances are equivalent. For long-running agents, you often want to scale the *compute allocated to a single task* (give it more CPU/memory) rather than adding more agents in parallel.

**Missing durability.** Microservices survive pod restarts because state lives in databases. Agents carry critical state in their context window — a prompt buffer that lives in process memory and disappears on kill.

## What agent infrastructure actually needs

**Durable task queues, not HTTP routing.** Each agent task is a unit of work that should be claimed by exactly one worker, have its state checkpointed, and be resumable after interruption.

**Long-polling or WebSocket connections.** Not short-lived HTTP request/response cycles. The connection lifetime should match the task lifetime.

**Context persistence.** If an agent is interrupted, its accumulated context (tool outputs, intermediate reasoning, file states) needs to be recoverable. This is closer to database transaction logging than HTTP session cookies.

**Outcome-based SLOs.** Measure task completion rate and success rate, not latency percentiles. An agent that takes 20 minutes and succeeds is better than one that times out in 25 seconds.

**Explicit idempotency design.** Before executing any side-effectful tool call, the agent infrastructure should record the intent. Replays can then skip already-completed steps rather than duplicating them.

## The organizational implication

This isn't just an infrastructure design problem. It's a skills gap problem. Platform engineers who've spent their careers thinking about stateless HTTP services need to update their mental models.

The primitives they need to learn are closer to job queues, distributed transactions, and event sourcing than to REST API gateway configuration.

The good news: all of this has been solved before. Workflow engines, durable execution systems, and long-running job frameworks have existed for years. The challenge is adapting those patterns to the new constraints of LLM-powered agents — particularly the context window as a first-class resource.

That adaptation is where the interesting engineering work is happening right now.

---

*This is the first in a series on AI infrastructure patterns for platform engineers.*
