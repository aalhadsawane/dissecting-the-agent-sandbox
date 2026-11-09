# 05 — Objectives → pre-registered hypotheses

**TL;DR**
- Each brief objective is restated as a hypothesis with a measurable statistic, a threshold, and what counts as disconfirmation.
- Two hypotheses are added: **H-F** (folk-number audit — the cheapest early, blog-worthy result) and **H-R** (residual — the most likely disconfirmation).
- Thresholds are provisional until the week-2 pilot; changes after data collection starts are logged in `10-decision-log.md` with reason.
- Prior expectation is stated for each, so the write-up can say honestly whether we were surprised.

Notation: `r` runtime, `w` workload, `D/A/C/η/R` from `03-unit-of-account.md`.

---

## H-1 (from O1) — sandbox share of wall-clock as a function of model latency

- **Statistic:** `f(L) = T_sandbox / (T_sandbox + Σ model latency)` over a replayed trajectory, per runtime; crossover `L*` where `f = 0.5`.
- **Claim:** for coding-agent trajectories, `L*` lies within the range of measured model-call latencies of a local model on the owner's hardware *and* of published API latencies — i.e. the sandbox is the bottleneck for realistic fast models.
- **Disconfirmed if:** `L*` is below the fast end of realistic latencies for all runtimes (sandbox never dominates), consistent with 2605.26297's 71–98% LLM share.
- **Prior:** ~60% that it is disconfirmed for `runc`, less so for `kata`/`gvisor` on install-heavy trajectories.
- **Why it beats the original:** the original O1 (>50% with stubbed model) cannot fail.

## H-2 (from O2) — boundary cost vs start cost

- **Statistic:** `T_boundary(r, w)` vs start cost under three start paths (cold boot, snapshot restore incl. first-N-seconds post-restore faults, warm-pool claim).
- **Claim:** for `w` with native duration ≥ 2 s, `T_boundary > cold-start` for all VM-based rows; versus warm-pool claim, true for all rows.
- **Disconfirmed if:** cold start exceeds `T_boundary` for the median replayed trajectory on any VM row.
- **Prior:** high confidence vs warm pool (near-trivial — reported as such); moderate vs cold boot for Kata.
- **Secondary (exploratory):** fraction of snapshot-restore cost that appears as MEM-class HSEs after "start" is reported (REAP-style hidden cold start).

## H-3 / H-R — the abstraction explains the slowdown

- **Statistic:** residual share `R(r,w) / (T_total(r,w) − T_total(native,w))`.
- **Claim:** `|R|` share ≤ 30% for every runtime × phase.
- **Disconfirmed if:** > 30% for any VM row on `build` (memory-heavy) — expected source: 2-D page walks, guest-kernel tax.
- **Prior:** ~50% that it is disconfirmed for VM rows on `build`. This is the planned "plainly disconfirmed" result if it happens.

## H-4 (from O3) — normalized metric orders runtimes consistently

- **Statistic:** Kendall's W of runtime rankings across workloads, computed for (a) per-class η, (b) raw throughput / wall-clock.
- **Claim:** W(η per class) > W(raw) with non-overlapping bootstrap CIs.
- **Changed from brief:** "host time per **crossing**" replaced with "host time per unit **demand**" (see `02-critique.md` §2). Per-crossing η is still computed and reported, to show why it misleads.
- **Disconfirmed if:** CIs overlap or W(raw) ≥ W(η).
- **Prior:** moderate. Per-class η should be stable; aggregate η may not.

## H-5 (from O4) — ranking flips between phases

- **Statistic:** bootstrap rank distribution per (runtime, phase); a flip = two runtimes swap order between two phases with non-overlapping rank intervals.
- **Claim:** ≥ 1 flip exists under default FS configs.
- **Control:** repeat with FS held as constant as possible (`*-blk` rows). If the flip disappears, the cause is **FS design, not isolation mechanism** — reported as such.
- **Prior:** high that a flip exists with default FS (e.g. gVisor competitive on compute-bound build, poor on metadata-heavy install); uncertain whether it survives FS control.

**Round-2 note on H-5:** the expected flip is now framed around **metadata-heavy phases** (init, turn:test, env-build) vs compute phases. Evidence for large FS-path effects comes from vendor data (Modal). Evidence that install is a small share of agent time comes from AgentCgroup. Both are in `11` §2–3. Clone/install remain measured, but under `env-build`.

## H-S (new, round 2) — state operations rank runtimes differently from turns

- **Statistic:** checkpoint and restore latency, bytes copied, and post-restore fault HSEs per runtime's native mechanism (CRIU for runc, runsc checkpoint/fs snapshot, FC full/diff snapshot, CHV snapshot), at fixed dirty-set sizes and on replayed traces.
- **Claim:** the runtime ranking on `state-op` differs from the ranking on `turn:*` phases (non-overlapping bootstrap rank intervals).
- **Disconfirmed if:** the same order holds.
- **Prior:** high that it differs. microVM memory snapshots scale with RAM (E2B ~4 s/GiB), while overlay/ZFS-style FS checkpoints are ~10–20 ms (Crab, DeltaBox).
- **Scope guard:** measurement of existing mechanisms only. No new C/R design.

## H-D (new, round 2) — durability semantics explain a large part of "FS speed" gaps

- **Statistic:** turn:test and env-build time per VM row under `cache_type`/flush settings that do vs do not propagate guest fsync to the host.
- **Claim:** at least one published "microVM is faster at file I/O" result (VEE'20, Lee et al. 2023) shrinks by ≥ 50% once durability is equalized.
- **Disconfirmed if:** the gap persists.
- **Prior:** moderate. VEE'20 itself attributes Firecracker's write speed to skipping write-back.

## H-M (new, 2026-11-09, exploratory) — mitigation state changes unit costs for userspace-handled exits only

- **Background:** on HOST-A the kernel issues IBPB before the first exit to userspace after a VM exit (VMScape; kernel doc). It is not issued if userspace did not run between exit and re-entry.
- **Statistic:** `C_c` (host time per HSE) for `USERSPACE-handled` vs `KERNEL-handled` exit classes, per runtime, under `vmscape=ibpb` vs `vmscape=off`.
- **Claim:** `C` of USERSPACE-handled classes rises materially (threshold to be fixed after the week-1 pilot) while KERNEL-handled `C` does not; gVisor-KVM host calls behave like USERSPACE-handled exits.
- **Disconfirmed if:** no class shows a difference outside bootstrap CIs, or KERNEL-handled classes change too (would indicate a different cause).
- **Feasibility:** needs a boot-parameter change → only with admin consent, preferably on AWS metal where we are the only tenant (Ice Lake-class parts are listed as not vulnerable, so choose a part that is). Otherwise reported as an unquantified caveat.
- **Prior:** moderate; the mechanism is documented, the magnitude is unknown (kernel doc gives no numbers).

## H-6 (from O5) — tail latency under co-tenancy

- **Setup:** one measured sandbox + N ∈ {0, 1, 2, 4, 8, …up to RAM limit} neighbors, neighbor types from a fixed set (CPU-bound, syscall-heavy, I/O-heavy, memory-bandwidth, idle-live).
- **Predictors (fixed in advance):** (a) aggregate host CPU utilization; (b) neighbors' HSE rate × unit cost, per class.
- **Statistic:** Spearman ρ and MAE for p99 of the measured sandbox's tool-call latency, leave-one-neighbor-type-out.
- **Claim:** (b) beats (a) on both metrics with non-overlapping CIs.
- **Disconfirmed if:** not.
- **Risk:** probe overhead on neighbors distorts tails (Craun et al.); T1 counters only, overhead measured with probes on/off.
- **Placement sub-hypothesis (2026-11-09):** neighbor placement is a designed factor on the hybrid host: same-core SMT sibling, different P-core, E-core. Claim: p99 inflation orders SMT-sibling > P-core > E-core for CPU- and syscall-bound neighbors, and the order changes for memory-bandwidth neighbors. Disconfirmed if the order is the same for all neighbor types.
- **Prior art (round 2):** DSec (arXiv:2609.22978) already reports +45.2% latency-sensitive step latency at 50% best-effort load, and +17.3% with SCHED_IDLE + core scheduling. H-6 differs by attributing the interference to host-serviced-event classes and by adding memory-pressure neighbors. Memory, not CPU, bounds density (AgentCgroup).

## H-F (new) — folk-number audit

- **Statistic:** for each published number in `01-literature.md` §2–3 (Firecracker boot, gVisor syscall/open overheads, vendor cold-start figures), our pinned-version measurement and the config difference that explains any gap.
- **Claim:** the majority of folk numbers are explained by one of: version (pre/post systrap), platform, FS backend, warm pool vs cold, or what event is timed.
- **Disconfirmed if:** a gap remains unexplained after controlling these — which is itself a vendor-relevant finding.
- **Why first:** cheap, needs only calibration micro-benchmarks, and is the blog post 1 result.

## O6 — release (not a hypothesis; acceptance criteria)

- `make reproduce-table` (or equivalent) regenerates the headline table from committed raw data in CI.
- Runbook regenerates raw data on a fresh bare-metal host of the same class.
- Environment manifest per run; dataset with checksums; probe overhead table; list of disconfirmed hypotheses.

## Summary table

| ID | Brief | Primary statistic | Threshold | Expected | Cheapest test week |
|---|---|---|---|---|---|
| H-F | — | gap explained | majority | likely confirm | 4 |
| H-1 | O1 | crossover `L*` | within realistic range | likely disconfirm (runc) | 2 |
| H-2 | O2 | T_boundary vs start | > start for ≥ 2 s | confirm vs warm pool | 5 |
| H-R | (O2/O3) | residual share | ≤ 30% | coin-flip for VMs/build | 5 |
| H-4 | O3 | Kendall W | η > raw | moderate | 5 |
| H-5 | O4 | rank flip | ≥ 1, survives FS control? | flip yes, survival unknown | 5–6 |
| H-6 | O5 | ρ, MAE on p99 | HSE > CPU util | uncertain | 6 |
