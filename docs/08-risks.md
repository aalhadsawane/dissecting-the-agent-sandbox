# 08 — Risk register

**TL;DR**
- Highest risks: (1) scoop by the "Part 2" of the AI-sandbox security study or a vendor, (2) the home server is unsuitable (virtualized, too little RAM, noisy), (3) probe overhead or mis-attribution invalidates per-sandbox numbers.
- Each risk has an early signal and a pre-decided response.

| # | Risk | Likelihood | Impact | Early signal | Response |
|---|---|---|---|---|---|
| R1 | Scooped: arXiv:2606.08433 Part 2 or a vendor publishes cross-runtime perf | M | H | Weekly arXiv/vendor-blog check | Our differentiators: unit of account, per-sandbox attribution, phase mapping, residual. Publish Blog 1 early (week 4) to timestamp the method |
| R2 | Home server not bare metal / lacks VMX or constant TSC | L–M | Fatal for host | G0 checks | Stop on that host; AWS metal if account allows; else re-scope to runc/gVisor-systrap only and say so |
| R3 | RAM too small for tmpfs + co-tenancy | M | M | Week-0 specs | Shrink O5 N range; fewer/smaller repos; never fall back to HDD |
| R4 | eBPF probe overhead distorts tails or neighbors | M | H | G2 overhead table | T1 counters only; `perf kvm stat` sampling; LTTng as second stack |
| R5 | Mis-attribution of kernel-thread work (vhost, kworker, softirq) | M | M | Σ attributed vs `/proc/stat` mismatch | "Unattributed host" bucket reported per class; kernel ≥ 6.4 for vhost_task |
| R6 | Trajectory replay non-deterministic | M | M | G1 | Fall back to phase micro-workloads |
| R7 | FS confound dominates install/clone (H-5 flip is FS, not mechanism) | H | M (it's a finding) | FS-control rows | Report as finding; adjust claim wording |
| R8 | Residual large → abstraction incomplete | M | M (it's a finding) | Week 5 | Report plainly; add guest-side probes for guest-kernel tax |
| R9 | Runtime version churn during the 8 weeks | M | L | Release feeds | Pin at week 1; no upgrades mid-study; note newer versions as future work |
| R10 | Kata/FC setup time eats weeks | M | M | Week 1 slips | Firecracker direct (no containerd) first; Kata-CLH second; drop Kata-QEMU |
| R11 | AWS account cannot launch metal | H | L | Week 0 check | Single-host result, stated plainly |
| R12 | Citations wrong/mis-titled | Confirmed for 1 | L–M | `01` §7 | Resolve before any draft; never cite unread numbers |
| R13 | Noise from home environment (thermal, other services, power events) | M | M | Pilot variance | Interleaved runs, reboot blocks, record temperature/frequency, overnight only if Q4 allows |
| R14 | Scope creep into security (escape testing) | L | M | — | Non-goal; refuse |
| R15 | Scooped by an active stateful-agent-sandbox group (HKUST: AgentSysBench+Crab; SJTU IPADS: DeltaBox; eunomia-bpf: AgentCgroup) | M–H | H | Weekly arXiv check on these author lists | Differentiate on cross-runtime boundary decomposition + durability-controlled FS comparison. Blog 1 early |
| R16 | tmpfs-backed storage cannot reproduce production lazy remote loading (Modal ImageFS, DSec 3FS, E2B NBD) | H | M | — | State as limitation, or emulate lazy loading with a local lazy block/FUSE server (Q-F3) |
| R17 | Scope creep into building a C/R system (H-S) | M | M | — | Measure native mechanisms only |
| R18 | Vendor numbers (Modal, Tensorlake) quoted as facts | M | M | Review | Grade V everywhere. Reproduce before citing in paper |
| R19 | Some sources unreachable from the cloud session (listman.redhat.com not allowed; lore.kernel.org 403) | Confirmed | L | — | Owner reads them, or allows the host |
