# 04 — Measurement methodology

**TL;DR**
- Measure on the bare-metal home server only; everything else (GitHub Actions, the Claude container) is a VM and is used for analysis CI, never for numbers.
- Workloads: (a) **agent trajectory replay** for realism, (b) **phase micro-workloads** (clone / install / build / test / edit-grep), (c) **calibration micro-benchmarks** for comparison with Firecracker'20 / Young'19 / SeBS.
- Probes in three tiers (off / counters / per-event). Probe overhead = A/B against "off". Every probe validated against an independent tool before use.
- Statistics: interleaved randomized run order, median + bootstrap 95% CI, rank statistics for ordering claims, pre-registered thresholds.

---

## 1. Platform discipline (invariants 1, 3)

Week-0 bare-metal proof (all must pass; result committed as raw output):
```
systemd-detect-virt              → "none"
grep -c hypervisor /proc/cpuinfo → 0
dmidecode -s system-product-name → real vendor model, not "KVM"/"VMware"/"HVM domU"
ls /dev/kvm                       → present
grep -o -E 'vmx|svm' /proc/cpuinfo | sort -u → vmx or svm
grep -o -E 'constant_tsc|nonstop_tsc' /proc/cpuinfo | sort -u → both
```
Counter-example on record: this planning session's container prints `docker`, has the `hypervisor` flag, and its kernel cmdline contains `--firecracker-init` — it is a Firecracker guest without `/dev/kvm`. It fails every check, as expected.

**Host-specific additions (2026-11-09, `12`):** the home server is shared (33 sessions), hybrid (8P+4E), turbo-on/`powersave`, C1/C2-only, and carries the VMScape IBPB mitigation. Therefore: (a) "pin governor/turbo/SMT" means *pin if consent is granted for a window, otherwise measure* APERF/MPERF per run per core; (b) all sandbox threads pinned to P-core threads, core IDs recorded; (c) every batch bracketed by **canary benchmarks** with a pre-registered acceptance gate (`12` §4) — flagged batches are re-run and both versions kept; (d) `memory.swap.max=0` on all sandbox cgroups; (e) manifest records `/sys/devices/system/cpu/vulnerabilities/*`, `cpuidle` states, `kvm_intel` module parameters, thermal readings, and the Firecracker-supported-kernel status (off-matrix on 5.15/6.8).

Pinned environment manifest (published per run):
- Kernel version + full `.config` hash + cmdline; microcode version; BIOS version.
- CPU governor (`performance`), turbo/boost state (off), SMT state (fixed), C-states policy (fixed; affects HLT exit cost), THP mode, KSM off, swap off, NUMA policy, `irqbalance` off.
- `isolcpus`/`nohz_full` sets; which cores run sandboxes vs. probes vs. harness.
- Exact versions/commits: runc, containerd, runsc (+ platform, FS flags), kata-runtime (+ hypervisor, FS), Firecracker, guest kernel(s) + configs, guest rootfs image digests.
- Toolchain: bpftrace/libbpf/perf versions.

## 2. Runtime matrix (one name = one config; invariant: name every platform)

| Name | Mechanism | Must | Optional |
|---|---|---|---|
| `native` | plain process, same kernel, same tmpfs | reference for `D` | — |
| `runc-seccomp` | runc + default seccomp profile, overlayfs | ✓ | `runc-noseccomp` (isolates seccomp delta) |
| `gvisor-systrap` | runsc, systrap, default FS settings of pinned version | ✓ | — |
| `gvisor-kvm` | runsc, KVM platform, same FS | ✓ (bare-metal best case) | — |
| `kata-clh-virtiofs` | Kata + Cloud Hypervisor + virtio-fs | ✓ | `kata-qemu-virtiofs` (VMM variable only) |
| `fc-virtioblk` | Firecracker direct, ext4 rootfs + workspace on virtio-blk backed by tmpfs file | ✓ | `fc-snapshot` (restore path, O2) |
| `kata-clh-blk` | Kata + CLH + block workspace | for FS-held-constant comparison (D-05) | — |

**Round-2 revision (D-16, D-15; evidence in `11` §4, §6):**
- Primary VM rows are **block-backed**: `fc-virtioblk` and `kata-clh-blk`. That is where stateful vendors converge (Modal, Tensorlake, E2B, DSec). `kata-clh-virtiofs` becomes a labelled secondary ("shared-directory" case).
- gVisor primary = default `--overlay2=root:self` + directfs (both defaults per gVisor docs). `gvisor-*-directfs=off` is optional (tests the 2023 change).
- Every row declares **durability** (guest fsync → host fsync or not) and **cache mode** (FC `cache_type`, virtio-fs `cache=`, guest mount options). The main comparison holds them equal. Vendor defaults (e.g. FC `Unsafe`) are a separate labelled series.
- `runc-idmap` (rootless / ID-mapped overlay) is optional: AgentCgroup attributes 31–48% of task lifecycle to Podman ID remapping.

Guest kernels: one pinned guest kernel version for all VM rows; config differences (FC minimal vs Kata) documented. Same vCPU count and RAM for every sandbox.

## 3. Storage and network held constant (brief §8)

- Everything the workload touches on tmpfs: images, rootfs files, workspaces, git bare repos, wheel/npm caches.
- HDD used only for the OS and for writing results **after** a run (results buffered in RAM during the run).
- Network: no internet during runs. Host-local mirrors on tmpfs:
  - git: bare repos served by `git daemon` / HTTP on a host bridge (exercises NET path), *and* a variant via local file copy (removes NET) to separate NET from STO.
  - Python: a local package index containing exactly the wheels/sdists the workloads need (pinned lockfiles).
  - Node: a local registry (same idea).
- State the choice in the paper: storage is not the variable under test; HDD numbers are never reported.

## 4. Workloads

### 4.1 Agent trajectory replay (realism)
- Source: public agent trajectories (e.g. SWE-agent / mini-SWE-agent / Terminal-Bench runs) on pinned repos, or self-recorded Claude Code / local-model runs.
- Extract the ordered tool-call sequence (bash commands, file reads/edits, test runs).
- Replay deterministically in each runtime; model calls replaced by `sleep(L)` with `L` from (i) 0, (ii) recorded latencies, (iii) a sweep for the O1 crossover curve.
- Caveat: replay removes feedback (a real agent would react to different outputs). Only trajectories whose tool outputs match the original (exit codes, test results) are kept; mismatches are logged.

### 4.2a Lifecycle phases (round 2, D-14 — supersedes the phase list in 4.2 for weighting)
| Phase | Example | Why (evidence, `11`) |
|---|---|---|
| env-build | clone + install from local mirror | Usually cached in images/templates. Measured, **not** weighted as session time |
| init / restore | cold boot, snapshot restore, warm-pool claim, ID-mapped overlay setup | Init 31–48% of lifecycle (AgentCgroup) |
| turn:test | pytest collection + run on pinned repos | Tests 44–73% of Bash time (AgentCgroup). Python import storm |
| turn:python | short Python snippets | 26.9% of GLM Bash time (AgentCgroup) |
| turn:edit-grep | read/edit/rg/git status | Frequent, individually cheap |
| idle-live | waiting on model | ~90% of sandboxes ≤ 5% CPU (DSec) |
| state-op | checkpoint / fork / rollback / suspend-resume, native mechanism per runtime | 47–77% of trajectory time in tree search on FC/CHV baselines (DeltaBox) |

Phase weights for headline numbers come from published trace distributions (AgentCgroup, Crab), cited, not from our intuition.

### 4.2 Phase micro-workloads (O4)
| Phase | Example | Dominant classes (expected) |
|---|---|---|
| clone | `git clone` of 3 pinned repos (small/medium/large file count) | NET, STO (metadata) |
| install | `pip install -r` / `npm ci` from local mirror | STO (many small files), NET, SYS |
| build | C/C++ or Rust build of a pinned project, `-j = vCPUs` | MEM, SYS (fork/exec), STO |
| test | pinned project test suite (pytest) | SYS, STO, TMR |
| edit-grep | `rg`/`sed`/`cat` loops modelled on Read/Bash/Edit tool mixes reported by 2605.26297 | SYS, STO |
| idle-live | sandbox alive, waiting on "model" | TMR, MEM (resident) |

Each workload: pinned commit, pinned lockfile, verified output hash.

### 4.3 Calibration micro-benchmarks
- Null syscall, `open/close` on tmpfs, 4 KiB read/write, small-file create/unlink, `fork+exec`, TCP RTT over loopback/bridge, page-touch (first-touch fault), idle wake-up latency.
- Subset of SeBS / vHive functions for comparability.
- Folk-number audit: reproduce, with pinned modern versions, the specific numbers quoted in Firecracker'20, Young'19, and the vendor blogs (`01-literature.md` §3) — and identify which config difference explains each mismatch.

## 5. Probes (invariants 4, 6)

Three tiers, selectable per run:
```
T0  off          only wall-clock + /proc/stat + cgroup cpu.stat     → overhead baseline
T1  counters     in-kernel per-(cgroup, class) counters + histograms (eBPF maps), no per-event export
T2  per-event    per-HSE records (ring buffer) for timelines and tail analysis
```
- Primary data for O2–O4 from T1; T2 only for short targeted runs and O5 tail attribution.
- Probe overhead reported per workload: `(T_wall(Tk) − T_wall(T0)) / T_wall(T0)` with CI, plus effect on co-tenants (Craun et al. show untraced processes pay too).
- Validation before trust:

| Probe output | Independent validator | Acceptance (pre-register) |
|---|---|---|
| exit counts per reason | `perf kvm stat` | counts within ±1% |
| exit handling time distribution | `perf kvm stat` report | medians within stated tolerance |
| syscall counts (runc/native) | `perf trace -s` / `strace -c` (on separate runs: strace is heavy) | ±1% |
| gVisor app-syscall counts | gVisor's own tracing/metrics **[verify mechanism]**, cross-checked vs native `D_SYS` | ±1% |
| per-cgroup CPU | cgroup `cpu.stat` | within ±2% |
| total host CPU | `/proc/stat` delta | Σ attributed + unattributed within ±2% |

LTTng is the fallback/second stack if eBPF probes disagree with validators.

## 6. Experimental design and statistics (invariants 5, 7)

- **One variable at a time**: each comparison changes exactly one config axis (runtime, platform, FS, VMM, neighbor count).
- **Run order**: randomized and interleaved across configs within a session (avoid "all gVisor in the morning" drift); reboot-separated sessions as blocks.
- **Warm-up**: discard first K iterations (K from pilot); report it.
- **Sample size**: pilot (n=10) to estimate variance → choose n for CI half-width ≤ 5% of median (or report achieved width).
- **Reporting**: median + 95% bootstrap CI (BCa); IQR; full distributions committed. p99 only when ≥ 1000 samples per configuration per run, reported with its own CI.
- **Ordering claims (O3/O4)**: rank stability via bootstrap rank distributions; concordance across workloads via Kendall's W; "flip" = rank change with non-overlapping bootstrap rank intervals.
- **Prediction claim (O5)**: models fixed in advance; evaluated by held-out neighbor types (leave-one-type-out), Spearman ρ and MAE on p99.
- **Three strikes**: a surprising result is re-run in three independent sessions (separate boots) before it is reported as a finding.
- **Pre-registration**: hypotheses and thresholds in `05-objectives-hypotheses.md` are committed before data collection; git history is the timestamp. Post-hoc analyses labelled as exploratory.

## 7. Data provenance (DoD: every number traces to a raw file)

```
results/<run-id>/
  manifest.json        environment manifest (§1), git commit of harness, config name
  raw/                 probe outputs, validator outputs, wall-clock logs (immutable)
  derived/             analysis outputs, regenerated by CI from raw/
paper/ and blog/ numbers are generated from derived/ — no hand-typed numbers.
missing value → "TBD" + GitHub issue number (invariant 2)
```
Large raw data: GitHub Releases or an external store with checksums in the repo.
