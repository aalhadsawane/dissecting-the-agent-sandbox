# 07 — 8-week plan with go/no-go gates

**TL;DR**
- Order is chosen to produce a **publishable result early** (folk-number audit, week 4) and to **validate the probe before any phase sweep**.
- Five gates. Failing a gate changes scope (listed), not the invariants.
- Minimum viable paper = H-F + H-4/H-5 on four runtimes with validated probes. O5 and any second-host check are stretch.

```
week   0     1        2          3            4           5          6          7          8
       │     │        │          │            │           │          │          │          │
       Q&A   host     workloads  probe v0     calibration phase      co-tenancy 2nd-data-pt +      write-up
       specs bring-up + replay   + validation + folk audit sweeps    (O5)       3-strikes  artifact
       pre-  pin env  + O1 pilot + overhead   BLOG 1      O2–O4                 reruns     BLOG 2
       reg.                                                                               arXiv
             G0       G1         G2           G3                     G4
```

## Week 0 — questions + pre-registration (no hardware work)
- Owner answers `09-open-questions.md`; provides server specs (`06-infrastructure.md` §1).
- Read the top-3 items of the reading queue (`01-literature.md` §10).
- Freeze hypotheses/thresholds (`05`), taxonomy (`03` §2), runtime matrix (`04` §2). Commit = pre-registration timestamp.

## Week 0b — host noise baseline and consent (added 2026-11-09; `12` §3–§4, §7)
- Owner runs the read-only evidence requests (`12` §9) and obtains consent/windows from the other tenants for: core claims (cpuset partitions), governor/turbo windows, memory cap, runtime installs, one scheduled reboot.
- Start the **canary baseline** (≥ 7 days: null-syscall, fixed-work loop with cycles+seconds, pointer-chase, copy bandwidth, APERF/MPERF, temperature, PSI). Decide acceptance thresholds from it before any experiment (pre-registered).
- Decide kernel option K1/K2/K3/K4 (`12` §2). Default K1 (+ K3 dual-boot entry only if smoke tests fail). AWS dropped (D-31); ask about a second lab host / faculty sponsor (`12` §6b).

## Week 1 — host bring-up
- Bare-metal proof (`04` §1) committed. **G0: host is bare metal and has VMX + constant TSC (owner-reported pack says yes; re-verify and store output). Fail → stop (invariant 1); fall back to AWS metal if available.**
- Pinned kernel + runtimes installed; each runtime row boots and runs `true` and a 1-file Python script. **Smoke tests on the off-matrix kernel (5.15) for Firecracker: boot, block I/O, snapshot full/diff, balloon; every failure is recorded as a finding (`12` §2).**
- tmpfs layout; local git/PyPI/npm mirrors; no-internet run mode.
- Environment manifest generator design (manual checklist acceptable at first).

## Week 2 — workloads + O1 pilot
- Select trajectories and phase repos; verify deterministic replay (output hashes).
- Pilot n=10 on 2 rows (`native`, `runc-seccomp`) → variance estimate → sample sizes.
- H-1 crossover curve for `runc-seccomp` (cheap, no probes needed).
- **G1: replay is deterministic for ≥ N trajectories (N set in week 0) and pilot CIs reach target width. Fail → reduce to phase micro-workloads only; drop replay-based H-1 claim.**

## Week 3 — probe v0 + validation + overhead
- T1 counters per (cgroup, class); T2 per-event for short runs.
- Validation table (`04` §5) filled for every row.
- Probe-overhead table (T0 vs T1 vs T2) per workload, including effect on an untraced neighbor.
- **G2: all validators within acceptance; T1 overhead measured (any value — but if > 5% on a workload, that workload's per-event results are flagged). Fail → switch to LTTng / `perf kvm stat`-only counting for the failing class.**

## Week 4 — calibration + folk-number audit → Blog 1
- Calibration micro-benchmarks for all rows; D/A/C for each class.
- H-F: reproduce each folk number, attribute gaps to config.
- **G3: three-strikes re-run of any surprising calibration result passes. Blog 1 published (method + audit).**

## Week 5 — phase sweeps (O2, O3, O4, residual)
- All rows × all phases at T1; H-2, H-R, H-4, H-5 analyses.
- FS-control rows (`*-blk`) for H-5.

## Week 6 — co-tenancy (O5)
- Neighbor matrix; H-6 with predictors fixed in week 0.
- **G4: tail measurements reach ≥ 1000 samples/config and probe-on/off tails agree within tolerance. Fail → report O5 as exploratory only.**

## Week 7 — portability + robustness
- Second data point (`12` §6b): P-core vs E-core run on HOST-A (H-X) always; another lab host or CloudLab only if available.
- Three-strikes reruns for every headline finding, in separate boot sessions.
- Stress the abstraction: guest-side probes in one VM row to measure "guest kernel tax" component of residual.

## Week 8 — write-up and release
- Artifact: runbook, manifests, raw data + checksums, CI-regenerated tables.
- Blog 2; arXiv preprint; workshop submission per Q3 (venue deadline **TBD** — not looked up; verify before week 0 ends).

## Scope ladder (cut from the top when behind)
```
cut first ─► second-host run (H-M, other hardware)
             gvisor-kvm, kata-qemu, fc-snapshot rows
             O5 (co-tenancy)
             trajectory replay (keep phase micro-workloads)
never cut ─► probe validation + overhead, four core rows, H-F, H-4/H-5, residual reporting
```
