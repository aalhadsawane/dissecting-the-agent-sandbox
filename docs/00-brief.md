# 00 — Original brief (preserved verbatim in substance)

> This file preserves the brief as handed to the agent on 2026-11-08, so later
> documents can be checked against it. Edits to the *plan* go in `02-critique.md`
> and `10-decision-log.md`, never here. Where later research contradicts this
> brief, the contradiction is recorded in `02-critique.md` — the brief text is
> left as-is on purpose.

**What this is:** a research + engineering brief for an autonomous agent. It states *what to find out and why it matters*. It deliberately does not prescribe the build. Methods are the agent's call; the invariants in §7 are not.

**Owner:** final-year undergrad, BITS Goa. Systems / distributed systems. Ships a paper on scheduling and attestation for untrusted, churning, heterogeneous compute.
**Assets:** a bare-metal server (HDD), unlimited GitHub Actions minutes (public repo), Claude cloud session container, a domain + blog, a modest AWS student credit balance, no cash budget.
**Horizon:** ~8 weeks of build, then artifact + blog + workshop paper.

---

## 1. Thesis

Agentic serving has moved the cost center. Inference is no longer the expensive part — tool execution, sandboxes, and idle-but-live session state are (AgentSysBench, 2026). Every sandbox vendor now competes on cold start, density, and isolation strength, and every one of them measures itself with its own workload on its own host. So the industry has a large, expensive, load-bearing number that nobody can compare across vendors, and a cost center nobody has opened up. **Open the sandbox.** Decompose where its time actually goes, at the isolation boundary, per workload phase, across the four isolation mechanisms in production use — and make the numbers comparable for the first time.

## 2. What is already known (as stated in the brief)

**Agent workloads are characterized — at the application level.**
- AgentSysBench, arXiv:2608.15127 — 10 instrumented agentic apps, 178k production sessions. Finds tools and sandboxes dominate cost. Instrumentation is Prometheus-level.
- *Agentic AI Workload Characterization*, arXiv:2605.26297 — ReAct traces across serving and tool execution, Claude Code + Harbor + vLLM, five agent benchmarks.
- MLSys 2026 cluster (KV-cache for coding agents, agent workloads on unified-memory workstations).

**Isolation cost is measured — within one mechanism at a time.**
- Manco et al., SOSP'17. Agache et al., Firecracker, NSDI'20. Young et al., HotCloud'19. CVM serverless study, arXiv:2609.04478.

**Guest↔host correlated tracing exists.** DORSAL "hypertracing" (LTTng guest+host, KVM tracepoints). *Does Virtualization Really Incur a 5% Overhead?* (<1% to >80% by workload). Zhang et al. I/O virtualization (2010); Hofstra multi-threaded overhead diagnosis.

**Benchmark suites exist.** SeBS, vHive, ServerlessBench, FaaSdom.

**Industry has commoditized the explanation, not the measurement.** Vendor blogs (PandaStack, Northflank, Fly, Synteq). GKE Agent Sandbox ~300 sandboxes/s on gVisor. "gVisor costs 10–30% on I/O, microVMs cost boot time" is folk knowledge with weak provenance.

## 3. The gap

Nobody has:
1. A **unit of account** comparing a container syscall, a gVisor seccomp trap, and a Firecracker VM exit as the same phenomenon.
2. **Per-sandbox attribution** of that cost (not host-aggregate).
3. A mapping from **agent workload phase** (clone / install / build / test) to boundary cost.
4. A **reproducible, vendor-neutral** isolation-cost comparison on student-affordable hardware.

## 4. Objectives (as stated)

| ID | Objective | Claim as stated |
|----|-----------|-----------------|
| O1 | Reproduce app-level finding locally | sandbox time > 50% of e2e wall-clock for a coding-agent trace with local or stubbed model |
| O2 | Decompose sandbox time at the boundary | boundary crossings > cold start once workload > ~2 s |
| O3 | Make four runtimes comparable | host-time per crossing orders runtimes consistently; raw throughput does not |
| O4 | Attribute cost to phase | runtime ranking flips between phases |
| O5 | Predict tail latency under co-tenancy | p99 under N neighbors predicted better by host-time-per-crossing than by aggregate CPU util |
| O6 | Release what makes it checkable | artifact, dataset, runbook, blog |

## 5. Definition of done
- Stranger with same hardware class reproduces headline table from repo alone.
- Every number traces to a committed raw file + pinned environment.
- Probe overhead quantified.
- ≥1 hypothesis disconfirmed, stated plainly.
- A vendor engineer learns something new about their own product.

## 6. Non-goals
Not a platform, scheduler, security evaluation (no escape hunting), UI, new benchmark suite, or "serverless on volunteers". Never benchmark a runtime without naming platform/config.

## 7. Invariants
1. Bare metal (nested virt distorts exit cost by ~10×). If host is virtualized, stop.
2. Never fabricate, never interpolate. Missing → `TBD` + issue.
3. Pin environment (kernel, runtimes, toolchain, governor) and publish it.
4. Quantify the observer.
5. One variable at a time; median + CI, never bare means.
6. Validate against an independent tool before trusting a new probe.
7. Three strikes: surprising result → three independent runs before belief.

## 8. Constraints
- HDD is a confound → images, repos, wheel caches on tmpfs; storage held constant; no IOPS headlines.
- AWS credits → short bare-metal validation, optionally one CVM comparison; else artifact hosting.
- Blog is a deliverable: publish method + first result before the paper.

## 9. Open questions for the human
- Default gVisor platform; one alternative in scope?
- CVM comparison wanted or distraction?
- Workshop first (APSys) or arXiv + blog first? Default: blog + arXiv, then APSys.
- Can the home server run overnight sweeps?

## Appendix — starting citations
AgentSysBench; Agentic AI Workload Characterization; CVM serverless; SeBS; vHive; ServerlessBench; FaaSdom; Firecracker NSDI'20; Manco SOSP'17; Young HotCloud'19; DORSAL hypertracing; "Does Virtualization Really Incur a 5% Overhead?"; Zhang 2010; `perf kvm stat`; LTTng; bpftrace/libbpf.

**Instruction attached to the brief (2026-11-08):** no code yet; planning, research and brainstorming only; be critical and evidence-grounded; document heavily; ambitious but realistic.
