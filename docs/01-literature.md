# 01 — Literature and prior art (with verification status)

**TL;DR**
- The two 2026 agent-characterization papers **disagree** on whether the sandbox dominates. One (AgentSysBench) finds non-LLM components dominate in 5 of 10 apps; the other (2605.26297) finds LLM inference is ~71–98% of execution time for Claude-Code-driven benchmarks. The brief's thesis cites only the first.
- AgentSysBench sandboxes are **Docker only**, measured with **cAdvisor → Prometheus**. No gVisor, no microVM, no eBPF. The "inside the sandbox" gap is confirmed.
- No paper found (as of 2026-11-08) that compares runc / gVisor / Kata / Firecracker **at the boundary, per sandbox, under agent workloads**. The closest is a *security* comparison (arXiv:2606.08433, "Part 1 of 2"); its Part 2 was not found and is a scoop risk to watch.
- The most-cited folk numbers (Young HotCloud'19 for gVisor) **predate** gVisor's systrap platform (2023) and rootfs overlay (2023). They are stale, not merely unreproducible.

Status legend: **V** = verified against the primary source page this session · **S** = verified only via secondary sources/snippets · **U** = not verified; do not cite numbers until read.

---

## 1. Agent workload characterization

### 1.1 AgentSysBench — arXiv:2608.15127 — **V (abstract + HTML body, first ~100k chars)**
*From LLM Inference to Agentic Workloads: Characterization and Implications for Serving Systems.* Chang, Zhou, Fu, An, Feng, Lu, Yao, Guo, Yu, Shan, Li, Yuan, Wang (HKUST / Alibaba / ByteDance). Aug 2026.

What it establishes:
- 10 agentic apps (retrieval/search, coding/terminal, browser/GUI/MCP).
- "Non-LLM components dominating latency in 5 of 10 applications."
- GUIAgent: containerized desktop sandbox > 70% of execution time.
- Claude Code on MCP-Atlas: LLM dominates some tasks (up to 90%), tool execution others (up to 84%).
- Pi-AutoR: "sandbox charges constitute over 99% of the per-request cost" — a **billing** estimate using public sandbox pricing (E2B), not a CPU-time measurement.
- Idle sessions hold state for minutes–hours; sandbox working set peaks at 28 GB/session; Claude Code KV cache up to 11 GB.
- State offloading: 4.6× memory reduction; tool-result caching removes 35.2% of redundant search calls.

How it measures (the gap we exploit):
- Sandboxes are **Docker containers**; some black-box apps use SSH-based remote sandboxes.
- Per-operation start/end records via annotations or proxies/sandbox hooks.
- Resource metrics: **cAdvisor + DCGM exporter → Prometheus/Grafana**.
- No gVisor, Firecracker, Kata; no eBPF; no VM-exit or syscall-level view.

What it does **not** establish:
- Where time goes *inside* a sandbox.
- Any comparison across isolation mechanisms.
- That sandboxes dominate *latency* for coding agents in general — results are per-app and mixed.

Brief discrepancy: the brief says "178k production sessions". Not confirmed in the portion read. Status for that number: **U**.

### 1.2 Agentic AI Workload Characteristics — arXiv:2605.26297 — **S**
Yuan, Nayak, Kundu, Talati. May 2026, cs.DC. (Brief titles it "…Characterization"; actual title is "…Characteristics".)
- Claude Code as scaffold, Harbor as orchestrator, vLLM 0.20.0 on H100 NVL (TP=2), OpenTelemetry gateway.
- Models: Gemma, Qwen (thinking / instant). Benchmarks: ADE-Bench, DABStep, GAIA, SWE-bench Pro, Terminal-Bench 2.0.
- **LLM inference ≈ 71–98% of total execution time** (snippet-level; confirm in full text).
- Tools: Read/Bash/Edit dominate ADE; Bash dominates DABStep and Terminal-Bench.

Implication: directly in tension with O1 (see `02-critique.md` §1). This is the single most important paper to read in full before week 1.

### 1.3 MLSys 2026 cluster — **U**
Brief mentions KV-cache for coding agents and unified-memory workstations. Not looked up. Action: list exact titles before citing.

### 1.4 Industry trace/benchmark write-ups — **S**
- NVIDIA TensorRT-LLM blog: agentic serving evaluated with trace replay and job-level metrics. Relevant as a *method* precedent for trace replay.
- SemiAnalysis InferenceX agentic benchmark posts. Inference-side, not sandbox-side.

---

## 2. Isolation-mechanism cost (one mechanism at a time)

| Work | Venue | What it measured | Status | Staleness risk |
|------|-------|------------------|--------|----------------|
| Manco et al., *My VM is Lighter (and Safer) than your Container* | SOSP'17 | LightVM / unikernel boot + density vs Docker | known | Xen-based; not KVM |
| Agache et al., *Firecracker* | NSDI'20 | <125 ms boot, <5 MiB overhead per VM | known; numbers re-quoted in secondary sources (S) | boot = to guest init with minimal kernel; not to "agent ready" |
| Young et al., *The True Cost of Containing: A gVisor Case Study* | HotCloud'19 | gVisor syscall, file open (Gofer/9P), network | abstract V; numbers S only | **High.** Pre-systrap (2023), pre-directfs, pre-rootfs-overlay (2023). Numbers describe ptrace/KVM era |
| Niroula, Shan, Ding, *Performance Study of Serverless Workloads in CVMs*, arXiv:2609.04478 | 2026 | CVM memory overhead (no cross-VM dedup), warm-start slowdowns driven by VM exits, esp. idle (HLT) transitions; vCPU-to-memory coupling amplifies | V (abstract) | Platform (SEV-SNP vs TDX) not stated in abstract |
| Ustiugov et al., *Benchmarking, Analysis, and Optimization of Serverless Function Snapshots* (vHive + REAP) | ASPLOS'21 | Firecracker snapshot restore dominated by host page faults (lazy guest memory); REAP prefetch cuts cold start 3.7× | V (abstract/intro) | Not in brief's citation list — **add it**; directly relevant to O2 |

Key observation: the CVM paper already shows **exit type (idle transitions) drives slowdown**. That is evidence *for* the brief's unit-of-account idea, inside one mechanism. Our contribution must be the cross-mechanism version, not the idea that exits matter.

---

## 2b. Cross-mechanism comparisons that DO exist (correction to the brief, added round 2)

The brief says isolation cost is measured "within one mechanism at a time". That is wrong. Full numbers are in `11-stateful-fs-path.md` §2.

| Work | Mechanisms | Versions | Method | What it lacks for us |
|---|---|---|---|---|
| Anjali, Caraza-Harter, Swift, *Blending Containers and Virtual Machines: A Study of Firecracker and gVisor*, VEE'20 — **V (PDF text read)** | LXC, gVisor (KVM platform), Firecracker | gVisor release-20200127.0, FC v0.19.1, kernel 5.4.13, CloudLab xl170 | host-kernel **code coverage** (gcov) + micro-benchmarks. Both gVisor and FC execute more host kernel code than native | 2020 versions. Coverage, not time per crossing. No per-sandbox attribution. Bulk I/O only. No Kata |
| van Rijn & Rellermeyer, *A Fresh Look at the Architecture and Performance of Contemporary Isolation Platforms*, Middleware'21 (arXiv:2110.11462) — **V (PDF text read)** | Docker, LXC, QEMU/KVM, Firecracker, Cloud Hypervisor, Kata, gVisor, OSv | ~2021 | CPU, memory, fio 128 KiB block I/O (FC excluded), iperf3, attack surface | Block-level only, no metadata. gVisor on 9p/gofer era |
| Lee et al., *Impact of Secure Container Runtimes on File I/O Performance in Edge Computing*, Appl. Sci. 13:13329, 2023 — **V (PDF text read)** | runc, Kata (virtio-fs), gVisor, Firecracker | Kata 1.12.0-alpha0, gVisor 20201208.0, FC v0.21.0 | fio + mpstat + symbol-level profiling | Bulk I/O, 2020 versions |
| Modal, *VM sandbox filesystem*, 2026-10-16 — **V (vendor blog read)** | virtio-fs VM, gVisor, host, virtio-blk VM | unstated | 100k-op metadata microbench + bun/ripgrep/SQLite/pgbench | No methods. Own hosts, own FS stack |

## 2c. Stateful agent sandbox systems (2026 cluster, added round 2)

| Work | Group | What | Overlap risk |
|---|---|---|---|
| AgentCgroup, arXiv:2602.09345 — **V (HTML read)** | eunomia-bpf (Zheng, …, Quinn) | Characterizes Claude Code agents in Podman (144 tasks): OS-level 56–74% of time, init 31–48%, tests dominate Bash time. eBPF/sched_ext/memcg controller | **High on tooling** (eBPF + cgroup, tool-call level). Differs in goal: resource control, single runtime |
| Crab, arXiv:2604.28138 — **V (HTML read)** | HKUST (shares authors with AgentSysBench) | Semantics-aware C/R for agent sandboxes. runc+CRIU+ZFS, eBPF inspector | Medium. Single runtime. C/R design, not boundary cost |
| DeltaBox, arXiv:2605.22781 — **V (HTML read)** | SJTU IPADS + Huawei | ms-level checkpoint/rollback on Firecracker. Overlayfs layer-switch (DeltaFS) + CRIU template fork (DeltaCR) | Medium. Has FC-Diff / CHV / docker baselines for state ops |
| Xu, Zhou, Wu, Kaffes, *Toward Systems Foundations for Agentic Exploration*, arXiv:2510.05556 — **V (PDF text read)** | Columbia | Benchmarks 6 snapshot/restore tools. Argues for a native fork primitive | Low |
| Shepherd arXiv:2605.10913; TVCACHE arXiv:2602.10986 — **S** | — | fork/revert for meta-agents. Tool-value cache with sandbox forking | Low |
| DSec, arXiv:2609.22978 — **V (HTML, first 100k chars)** | DeepSeek | Production sandbox platform: ~380K concurrent, ≥3,200 containers or ≥800 microVMs/node, EROFS/3FS lazy loading, OverlayBD+ublk for microVMs, CPU QoS study | Medium for O5 (co-tenant interference). Strong source for lifecycle realism |
| SWE-MiniSandbox, arXiv:2602.11210 — **V (HTML read)** | — | Replaces containers with mount-ns + chroot for RL. Env setup 88.9 s → 23.6 s. 605 GB → 89 GB for SWE-bench Verified | Evidence that sandbox setup cost drives RL infra away from containers |

## 3. Practitioner / vendor cross-runtime comparisons (folk numbers)

All **S**, none peer-reviewed, mutually inconsistent:
- A blog "benchmark: gVisor 1.0 vs Kata 3.0 vs Firecracker 1.5" (12 metrics) — reports Firecracker lowest syscall overhead, ~120 ms cold start. Note "gVisor 1.0" is not how gVisor is versioned (it uses date-based release tags) — a provenance red flag.
- Another blog: Firecracker ~125 ms, gVisor ~200–600 ms, Kata-QEMU 500 ms–1 s.
- A prompt-injection benchmark post: Firecracker ~148 ms cold boot; gVisor "28% p99 syscall overhead".
- AWS EKS write-up: gVisor negligible launch latency; Kata+Firecracker ~5 s per sandbox.
- Vendor landscape (secondary): E2B and Vercel on Firecracker; Modal on gVisor; Northflank Kata and gVisor; Daytona Docker with optional Kata.
- GKE Agent Sandbox (Google Cloud blog, Next '26): "300 sandboxes per second, per cluster" at sub-second latency, ~90% of allocations in 200 ms, **from a warm pool**. GA vs preview status inconsistent across sources.

Use of these: they are the *dependent variable* of a "folk-number audit" (see `05-objectives-hypotheses.md`, H-F), not evidence.

---

## 4. Security-side cross-runtime comparisons (scoop risk)

- Andronchik & Lokhmakov, *AI Code Sandboxes: A Comparative Security Study. Part 1 of 2* — arXiv:2606.08433, June 2026 — **S**. Engine-level properties (attack surface, leakage, stackability, CVE history, patch cadence, fuzzing) across five AI-sandbox products. No ranking.
  - **Risk:** "Part 2" may be performance. Not found as of 2026-11-08. Re-check weekly.
- SandboxEscapeBench (arXiv:2603.02277) — Docker/OCI only — **S**.
- Sandlock (arXiv:2605.26298) — unprivileged Linux primitives for confining agent code — **S**. Possibly a fifth "mechanism" (Landlock/seccomp-only). Out of scope; mention.

---

## 5. Guest↔host correlated tracing

- **DORSAL lab (Polytechnique Montréal)**, 2014–2018 — **S** (slides, LPC 2017 abstract, lttng-dev threads):
  - LTTng in guest and host; synchronization via convex-hull method needing 1:1 guest/host event pairs (hypercall traced on host).
  - Benbachir's "hypertracing": guest events offloaded via hypercall, recorded with host timestamps; ftrace function tracing across virtualization layers. Reported overhead (nested): LTTng 18% vs hypertracing 3.6%.
  - Bastien 2018: problems = separate guest/host clocks + matching traces to VMs; proposed `ptp_kvm` + chronyd, VM UUID from KVM debugfs.
  - State of code is unclear; treat as *technique* precedent, not a dependency.
- Linux kernel selftest patch (2023, LKML): eBPF program on KVM exit tracepoint, per-vCPU / per-VM exit counts — **S**. Closest existing per-VM exit attribution; not per-container, not cross-runtime.
- `perf kvm stat` — standard, independent validator for exit counts and per-reason handling time.

## 6. eBPF observer cost

- Craun et al., *Eliminating eBPF Tracing Overhead on Untraced Processes*, eBPF '24 — **S**. Even untraced processes pay; memcached throughput −1.5% (pre-eBPF filter) and −2.7% (in-eBPF filter) per slides; more attached programs → worse. Relevant to invariant 4 and to O5 (neighbors are "untraced processes").

## 7. "Does Virtualization Really Incur a 5% Overhead?" — **U, possibly mis-titled**

A web search for this exact title found no matching paper. Results returned unrelated work (HP Labs 2007 Xen vs OS-level, ICPP 2015 virtualization overhead study at UVA, Waterloo DB-on-VM thesis). Action before citing: ask the owner for the source; candidate is the ICPP 2015 paper (cs.virginia.edu). **Do not cite the "<1% to >80%" range until the source is found.**

Also **U**: Zhang et al. I/O virtualization 2010; "Hofstra multi-threaded overhead diagnosis". Need exact bibliographic entries.

## 8. Benchmark suites (calibration, not agent workload)

SeBS (Copik et al., Middleware'21), vHive (Ustiugov et al., ASPLOS'21), ServerlessBench (Yu et al., SoCC'20), FaaSdom (Maissen et al., DEBS'20). These are **FaaS function** suites. They give comparability with prior isolation papers; they do **not** contain clone/install/build/test agent phases. See `04-methodology.md` §4 for how both are used.

## 9. gVisor platform facts (official docs) — **V**

- Default platform: **systrap** (replaced ptrace mid-2023). Uses seccomp `SECCOMP_RET_TRAP` to intercept syscalls.
- ptrace: deprecated, "expected to eventually be removed".
- KVM platform: recommended on bare metal outside GKE for best performance.
- systrap preferred inside VMs / nested virt.

Consequence: a bare-metal study **must** report systrap (the default users run) and should report KVM (the best-case bare-metal config). See `09-open-questions.md` Q1.

---

## 10. Reading queue (ordered by risk to the project)

1. 2605.26297 full text — confirms/denies O1 tension.
2. 2608.15127 remaining ~37k chars — "178k sessions", Figure 4 values, sandbox hooks.
3. arXiv:2606.08433 + watch for Part 2.
4. Young HotCloud'19 full text — exact numbers and gVisor version for the folk-number audit.
5. 2609.04478 full text — platform, exit-type breakdown method (reuse it).
6. REAP/vHive — UFFD/page-fault accounting method.
7. Firecracker NSDI'20 §boot methodology — what "125 ms" measures.
8. DORSAL slides — TSC-offset synchronization details.
9. Resolve §7 citations.

Sources are listed with URLs in `references.md`.
