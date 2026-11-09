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
| R20 | Shared host: 33 sessions from 5 other accounts; noise unquantified (Maricq: CoV up to ~10% even on exclusive nodes) | H | H | Canary baseline (`12` §4) | Acceptance gates fixed before data; windows; re-run flagged batches; state the limitation |
| R21 | Turbo/`powersave` frequency varies with other tenants' load | H | M | APERF/MPERF per run | Record; consented `performance` windows; report cycles + seconds |
| R22 | Hybrid P/E cores: scheduler migration mixes microarchitectures; vPMU disabled in guests | H | M | — | Pin to P-core threads; no guest PMU claims; host-side `perf` on vCPU threads only |
| R23 | Firecracker host kernel off its validated matrix (5.10 / 6.1 / 6.18); host is 5.15 | H | M | Week-1 smoke tests | K1 + declare; AWS cross-check on a validated kernel; K3 only if artifact suspected |
| R24 | VMScape IBPB state alters `C` for userspace-handled exits; mitigation still evolving across kernels | M | M | sysfs state in manifest | Pin kernel; sub-columns by handler locus; H-M on AWS metal |
| R25 | Governance: reboot / global knobs / root-daemon installs on a shared box without consent | M | H | — | Written consent per action (`12` §7); fallback = measure, not control |
| R26 | External validity: client hybrid CPU, C1/C2 only, non-ECC, 2-channel memory | H | M | — | Claim structure/ordering, not absolute times; AWS server-CPU cross-check |
| R27 | Swap on (8 GiB) can swap tmpfs/guest RAM | M | M | PSI, `memory.swap.max` | cgroup swap max = 0; verify no swap-in during runs |
| R28 | Leaking other people's identifiers in a public repo | M | M | Review before commit | Sanitize evidence (done); rule D-27 |
| R29 | Non-ECC memory: silent corruption in long runs | L | M | Output-hash mismatch | Hash every output; repeat flagged runs |
