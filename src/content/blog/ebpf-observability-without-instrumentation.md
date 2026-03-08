---
title: "eBPF: Observability Without Instrumentation"
description: "Why eBPF changes the economics of Kubernetes observability, and what platform teams need to understand before adopting it."
pubDate: "2026-03-08"
heroImage: "../../assets/blog-placeholder-4.jpg"
---

The traditional observability stack has a dirty secret: it requires your application to cooperate.

You add a metrics library, configure exporters, instrument your HTTP handlers, propagate trace context through every service boundary, and ship logs in the right format. If a service doesn't cooperate — legacy code, third-party dependencies, a team that hasn't gotten to it yet — you get a blind spot.

eBPF changes this.

## What eBPF actually is

eBPF (extended Berkeley Packet Filter) is a technology that lets you run sandboxed programs inside the Linux kernel in response to events — without modifying kernel source code or loading kernel modules.

From an observability perspective, the key capability is: **you can observe any system call, network packet, or kernel event from any process, without modifying that process.**

Your application doesn't need to know it's being observed. No SDK. No sidecar with application cooperation. No code changes.

## The Kubernetes observability problem eBPF solves

In a Kubernetes cluster, you're running tens to hundreds of containerized services. Getting consistent observability across all of them with traditional instrumentation requires:

1. Every team adopting the same metrics library
2. Every service exporting in a compatible format
3. Trace context being propagated correctly through every inter-service call
4. Log formats being standardized across services in different languages

In practice, this never fully happens. There's always a service that uses the wrong library version, a team that hasn't migrated yet, or a third-party container you don't control.

eBPF-based observability tools (Cilium, Tetragon, Hubble, Pixie) solve this by attaching to the kernel layer. Every TCP connection, every DNS query, every system call is visible — regardless of what's running in the container.

## What you get for free

With an eBPF-based observability layer on your cluster, you get without any application changes:

**Network flow visibility.** Which pods are talking to which, what protocols, how much bandwidth, latency at the connection level. Useful for both debugging and security policy.

**HTTP/gRPC golden signals.** For HTTP/2 and gRPC traffic, some eBPF tools can parse the protocol layer and give you request rates, error rates, and latency — the three pillars of the RED method — per service, per endpoint.

**DNS resolution tracing.** Which services are doing DNS lookups, to where, and whether they're succeeding. Surprisingly useful for debugging connection failures.

**System call auditing.** Which processes are executing what binaries, opening which files, making which network connections. Foundational for security monitoring.

## What eBPF doesn't replace

It's worth being clear about the limits.

**Application-level tracing.** eBPF can see the network boundary, but it can't see inside your application's business logic. If you want distributed traces that span your application's internal function calls, you still need instrumentation. eBPF gives you the service mesh layer; it doesn't give you jaeger traces through your codebase.

**Custom business metrics.** Things like "number of orders processed" or "queue depth by customer tier" require application-level instrumentation. eBPF can't infer business semantics from kernel events.

**Deep application profiling.** CPU profiling at the function level, memory allocation tracking, goroutine analysis — these still require language-specific profilers or instrumented runtimes.

The right mental model: eBPF handles the infrastructure and network layer; your instrumentation handles the application layer. Together, they cover everything.

## Platform team adoption considerations

Before rolling out eBPF-based observability, a few things worth knowing:

**Kernel version requirements.** eBPF capabilities have improved significantly with each kernel version. Most modern distributions (4.19+, ideally 5.4+) support the features observability tools need, but verify your node kernel versions before assuming compatibility.

**Privilege requirements.** Loading eBPF programs requires elevated privileges. This is handled by the DaemonSet that runs the observability agent, but it means your nodes need to allow it. In locked-down environments (managed Kubernetes with restrictive PSP/PSA policies), this requires explicit configuration.

**Performance overhead.** eBPF is designed to be low overhead, and in practice it usually is. But "low" isn't "zero". Budget for 1-3% CPU overhead on nodes and measure your specific workload before and after deployment.

**Data volume.** If you start capturing every network flow and system call, you generate a lot of data. Be intentional about what you export vs. what you keep local, and what sampling rate makes sense for your workload.

## Where to start

If you're running Kubernetes and want to experiment without a full production rollout:

1. **Cilium** — The most widely adopted eBPF networking and observability solution for Kubernetes. Replaces kube-proxy and adds network policy, observability, and load balancing.
2. **Pixie** — Specifically focused on observability, with an auto-instrumentation approach that requires only a DaemonSet deployment. Good for quick experimentation.
3. **Tetragon** — Security-focused eBPF from the Cilium team. Good if your first use case is security observability (detecting unexpected syscalls, file access patterns).

The operational model is genuinely different from traditional APM. Worth the learning curve.

---

*Next up: how to build an alert routing system that doesn't turn your on-call rotation into alert fatigue.*
