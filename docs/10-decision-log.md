# 10 — Decision log

Format: ID · date · decision · evidence · reversible?  Changes to pre-registered items after data collection starts must be logged here with a reason.

| ID | Date | Decision | Evidence / reason | Reversible |
|---|---|---|---|---|
| D-01 | 2026-11-08 | Planning phase only; no code until week 0 questions are answered | Owner instruction | — |
| D-02 | 2026-11-08 | Claude container and GitHub Actions are never used for measurement | Container verified Firecracker guest (`dmesg`, `hypervisor` flag, no `/dev/kvm`); Actions runners are VMs; invariant 1 | No |
| D-03 | 2026-11-08 | O1 reframed as crossover curve `f(L)`, `L*` | 2605.26297 reports LLM ≈ 71–98% of time; stubbed-model O1 is tautological | Yes, if owner rejects (Q6) |
| D-04 | 2026-11-08 | Unit of account = host-serviced event; headline metric = host time per unit workload demand, decomposed D×A×C, plus residual | Crossing granularity differs by orders of magnitude between gVisor and VMs (`02` §2) | Yes (Q7) |
| D-05 | 2026-11-08 | Filesystem backend is part of every runtime's name; FS-control rows added | FS path likely dominates clone/install (`02` §4) | No |
| D-06 | 2026-11-08 | O2 compares against cold boot, snapshot restore, and warm-pool claim; post-restore faults counted as MEM HSEs | GKE warm pool; REAP shows restore cost hides in page faults | Yes |
| D-07 | 2026-11-08 | Steady-state and start/orchestration measured separately; Firecracker driven directly, Kata-CLH as VM-backed OCI row | Firecracker is a VMM, not an OCI runtime | Yes |
| D-08 | 2026-11-08 | gVisor rows: systrap + KVM; ptrace excluded | gVisor docs: systrap default, ptrace deprecated, KVM best on bare metal | Yes (Q1) |
| D-09 | 2026-11-08 | CVM recommended out of core scope | No host-side tracing on cloud CVMs; overlaps 2609.04478 | Yes (Q2) |
| D-10 | 2026-11-08 | Folk-number audit (H-F) added as first publishable result | Folk numbers stale (pre-systrap) and mutually inconsistent | Yes |
| D-11 | 2026-11-08 | No number from a source marked S/U in `01` is cited until the primary source is read | Invariant 2; one brief citation not found | No |
| D-12 | 2026-11-08 | `02` §4 "FS dominates clone/install" withdrawn as a claim and replaced by graded evidence (`11`) | Owner review; AgentCgroup shows install ~10% of Bash time | No |
| D-13 | 2026-11-08 | Evidence grades P/A/V/C used for every number | Vendor numbers lack methods | No |
| D-14 | 2026-11-08 | Phases = env-build, init/restore, turn:test, turn:python, turn:edit-grep, idle-live, state-op. Weights from published traces | Agent sandboxes are stateful sessions (DSec lifetimes, AgentCgroup, Crab) | Yes (Q-W2) |
| D-15 | 2026-11-08 | Durability and cache mode declared per row and held equal in main comparison | FC default `Unsafe`; VEE'20 write-back finding | No |
| D-16 | 2026-11-08 | Block-backed VM configs primary; virtio-fs secondary; gVisor defaults (overlay root:self, directfs) primary | Vendor convergence on block (Modal, Tensorlake, E2B, DSec) | Yes (Q-R2) |
| D-17 | 2026-11-08 | State ops measured with native mechanisms; no new C/R design | DeltaBox/Crab occupy design space | Yes (Q-S1) |
| D-18 | 2026-11-08 | Novelty restated: current versions + metadata/agent workloads + per-sandbox boundary attribution + lifecycle/state ops + controlled FS/durability | VEE'20, Middleware'21, MDPI 2023, Modal 2026 exist | No |
