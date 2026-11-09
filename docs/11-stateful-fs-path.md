# 11 — Stateful agent sandboxes and the filesystem data path (evidence review)

Written 2026-11-08, after the owner pointed out that (a) serverless sandboxes are stateless while agent sandboxes are stateful, and (b) "the FS path dominates" in `02-critique.md` §4 was a conjecture without numbers. This document replaces that conjecture with evidence, graded by source quality.

**Evidence grades:** **P** = peer-reviewed · **A** = arXiv preprint · **V** = vendor engineering blog/docs (no methods, single host, own workload) · **C** = community / issue tracker / mailing list. A number from **V** or **C** is a *lead to check*, never a result we cite as fact.

**TL;DR**
- **Agent sandboxes are long-lived, stateful sessions, not FaaS invocations.** DeepSeek's production platform reports median sandbox lifetimes of 15–17 min, p99 above 3 h, and about 90% of sandboxes using ≤5% of their requested CPU [A]. The lifecycle (build env → start/restore → many short turns with idle gaps → checkpoint/fork/suspend → resume) has no counterpart in SeBS/vHive-style suites.
- **FS-path cost differences are real and large, but only proven for metadata-heavy operations, and mostly by vendors.** Modal (Oct 2026) reports `bun install` at 12.5 s over virtio-fs vs 1.07 s over a virtio-blk design, and 100k `chown` calls at 9.87 s vs 0.21 s [V]. gVisor's rootfs overlay cut fsstress from 262.79 s to 3.18 s [V]. A peer-reviewed letter measured gVisor issuing about 48× more host syscalls than native for `open` [P].
- **Counter-evidence on *which phases matter*:** in Claude Code SWE-style tasks under Podman, package installation is ~10% of Bash time and tests are 44–73%. Git and file exploration are "lightweight". Container/agent initialization is 31–48% of task lifecycle [A]. So "clone/install dominate" is **not** supported for prebuilt-image benchmarks. Tests (Python import storms) and initialization are the better-supported targets.
- **Industry is converging on block devices, not shared filesystems, for stateful VM sandboxes:** Modal VM Sandboxes (virtio-blk + lazy block daemon), Tensorlake (CoW overlay inside Firecracker's virtio-blk), E2B (CoW block devices, NBD), DeepSeek microVMs (OverlayBD over ublk). This very session's container uses virtio-blk ext4 plus read-only block devices, and no virtio-fs. **Kata + virtio-fs, the brief's implicit default for Kata, is probably the *least* representative config for stateful agent vendors.**
- **State operations are a new cost term the brief missed.** Checkpoint/restore/fork can be 47–77% of trajectory time in tree-search agents on Firecracker/CHV baselines [A]. Their cost is set by the FS design (file-level vs block-level copy-on-write, CoW), not mainly by the isolation mechanism.
- **Durability is a hidden confound.** Firecracker's default block `cache_type` is `Unsafe` [C/docs mirror]. VEE'20 found Firecracker writes fast "because it makes no effort to write data back to persistent storage" [P]. Comparing runtimes without fixing durability semantics compares different contracts.
- **Prior art is denser than the brief says.** Cross-mechanism comparisons exist: VEE'20, Middleware'21, MDPI 2023, and Modal 2026. Three active groups publish on stateful agent sandboxes: HKUST (AgentSysBench, Crab), SJTU IPADS (DeltaBox), and eunomia-bpf (AgentCgroup, which also uses eBPF + cgroups). The gap narrows, and gets sharper (§8).

---

## 1. Stateless FaaS vs stateful agent sandbox — the lifecycle

```
FaaS invocation (what SeBS / vHive / ServerlessBench model)
  [cold/warm start] → [handler runs ms–s] → [return, instance may be reused but state is not a contract]

Agent session (what vendors actually run)
  [env build: clone+install, once, cached]                       ← often done at image/template build time
     │
  [start | snapshot-restore | warm-pool claim]                   ← start cost + post-restore page faults
     │
  ┌─►[turn: LLM wait (idle-live)] → [tool call: bash/test/edit, s] ─┐   × ~100+ turns
  │                                                               │
  └──────────────[checkpoint? fork? rollback?]◄───────────────────┘   ← state ops (new)
     │
  [suspend / pause / archive]  …minutes–hours…  [resume]          ← memory+FS persistence
     │
  [destroy]
```

Evidence for each segment:

| Segment | Evidence | Grade |
|---|---|---|
| Long life, mostly idle | DSec: median lifetime 17.4 min (containers) / 15.5 min (microVMs), p99 > 3 h. ~90% of sandboxes use ≤ 5% of requested CPU on average | A (arXiv:2609.22978) |
| Idle-but-live state | AgentSysBench: sessions hold state idle for minutes to hours. Sandbox working set peaks at 28 GB/session | A (2608.15127) |
| Many short turns | Crab (Terminal-Bench traces): median turn 3.34 s, about 117 turns per task. `run_shell_command` is 60.4% of tool calls | A (2604.28138) |
| Tool-call durations | AgentCgroup (Claude Code, Podman): Bash averages 4–6 s. Read/Edit under 0.5 s. Bash spans 3 orders of magnitude | A (2602.09345) |
| Env build is pre-done | Epoch AI moved `git clone` into a shared image layer (git history = 291 MB of a 330 MB layer). Each instance then only runs `git reset --hard` | V (epoch.ai) |
| Persist across pause | E2B pause keeps memory + FS. ~4 s per GiB RAM to pause. Paused sandboxes kept indefinitely | V (E2B docs) |
| | Daytona: stop keeps FS. Archive moves FS to object storage. Memory is not kept | V |
| | Vercel Sandbox (Firecracker): stopping a persistent sandbox auto-snapshots the FS. Snapshots expire 30 days after last use by default | V |
| | Modal: filesystem, directory and memory snapshots (memory: preview, gVisor-based, 7-day expiry) | V |
| | Fly Sprites: checkpoints restore in place and do not expire. "~300 ms" restore claim is third-party only | V / C |
| | Codex cloud: container state cached up to 12 h after setup. Cache invalidated on script/env change | V (secondary) |
| | GKE Pod Snapshots (gVisor): freeze and rehydrate. App starts before memory finishes loading | V (InfoQ) |
| | Cloudflare Sandbox: per-directory squashfs backups to R2 | V |
| Fork / branch | Morph Infinibranch: "<250 ms" snapshot/branch [V]. One third-party median of 176 ms (n=38) [C] | V / C |

**Consequence:** SeBS / vHive / ServerlessBench are kept as *calibration* only (`04` §4.3). The headline workload must be agent-lifecycle traces. No existing suite covers the whole lifecycle, so this is a scoped exception to the brief's "don't build a new suite" (see Q-W1 in `09`).

## 2. What the FS path costs — numbers, by mechanism

### 2.1 Metadata-heavy operations (the agent-relevant case)

| Source | Setup | Result | Grade |
|---|---|---|---|
| Modal, *VM sandbox filesystem*, 2026-10-16 | 100,000 calls each. Columns: virtio-fs (VM), gVisor (ImageFS behind gVisor), ImageFS (host-direct), ImageFS-blk (VM, virtio-blk) | `stat` 3.23 / 2.12 / 1.45 / 0.90 s. `chown` 9.87 / 2.31 / 1.58 / 0.21 s. `link` 9.75 / 3.91 / 2.57 / 0.80 s. `cp -R` 32.90 / 17.75 / 6.48 / 5.74 s | V — no hardware, no run count, no mean/median stated |
| same | Workloads | `bun install` 12.5 s (virtio-fs) / 3.1 s (ImageFS) / 1.07 s (ImageFS-blk). ripgrep over the Linux repo, warm: 6.03 / 3.77 / 0.18 s. Cold: 24.3 / 4.47 / 8.53 s (blk slower when cold, unexplained). SQLite speedtest1: 543.4 / 289.8 / 43.2 s | V |
| same | Mechanism | One `chown` = 3 guest→host round trips (~99 µs). `virtiofsd` makes 6 host syscalls. ~2.1 µs in-guest over virtio-blk. Recursive `chown` went from ~4 s to > 1 min when virtio-fs was added | V |
| gVisor rootfs overlay blog, 2023 | fsstress (LTP), gofer rootfs vs overlay | 262.79 s → 3.18 s. Abseil Bazel build: sandboxing overhead "halved" (no absolute numbers). Overlay is now the default (`--overlay2=root:self`, file-backed) | V |
| gVisor directfs blog, 2023-06-27 | gofer RPC vs direct host FDs | `stat` > 2× faster. "12% reduction" in benchmark workload time. 17% lower Ruby load time. Security trade: sandbox gets `openat` etc. and the gofer's capabilities | V |
| Jang, Suh, Tak, IEICE Trans. Inf. & Syst. E107-D(2), 2024 | syscall-mapping trace of gVisor | ≈ 48× more host syscalls than native for `open`, ≈ 6× for `read`/`write` (abstract) | P (letter) |
| Young et al., HotCloud'19 | gVisor 2019 (ptrace/KVM, gofer) | small-file read 11× slower; large download 2.8× slower (secondary quote) | P, numbers **S** |
| LWN 774495 (virtio-fs RFC, 2018) | smallfile, 8 threads | create: virtio-9p 194 files/s vs virtio-fs 714 (cache=none). `ls -l`: 2038 vs 13571 (cache=always) | C (author benchmark) |
| Man Group pip analysis | pip extraction | pip re-created the parent dir per extracted file. Hundreds of packages / tens of thousands of files take minutes. Fix gave 5% | V |

### 2.2 Bulk-data operations (less agent-relevant; where most prior papers looked)

| Source | Setup | Result | Grade |
|---|---|---|---|
| Anjali, Caraza-Harter, Swift, VEE'20 | LXC / gVisor (release-20200127.0, **KVM platform**) / Firecracker v0.19.1. 1 GB file. Kernel 5.4.13 | Firecracker writes "much faster than the other platforms because it makes no effort to write data back to persistent storage". gVisor ≈ LXC except small files. Warm guest cache gives about 2× the throughput of a warm host cache | P |
| van Rijn & Rellermeyer, Middleware'21 (arXiv:2110.11462) | fio 128 KiB, libaio, extra block device. **Firecracker excluded** (no extra drive). Kata on 9p | gVisor and Kata "in the best case reaching only half" of others. Cloud Hypervisor significantly worse at the time. Kata virtio-fs "significantly outperforms" 9p | P |
| Lee et al., *Applied Sciences* 13:13329, 2023 | Kata 1.12.0-alpha0 (virtio-fs), gVisor release-20201208.0, Firecracker v0.21.0. NVMe | Seq. Kata 581–752 MB/s. Random read avg: runc 2458 vs Kata 422 MB/s. 1 MiB random read: Firecracker 4749 MB/s, 4.3× Kata. Firecracker wins via guest page cache. Kata uses the most CPU | P (MDPI) |
| Kata issue #2138 | fio 4 KiB | IOPS: runc 45.5k, Kata 9p 13.1k, virtio-fs 3.6k (unpatched) → 19.4k (patched) | C |

### 2.3 What this evidence does and does not show
- **Shows:** per-metadata-op cost differs by roughly **10–50×** across FS paths (virtio-fs vs in-guest ext4 on virtio-blk). gVisor's FS cost has moved a lot with config/version (overlay, directfs). So the "folk numbers" are config artifacts (supports H-F).
- **Does not show:** what *fraction of agent session time* those operations take. Every agent-level number (§3) points away from clone/install.
- **Quality gap:** the only head-to-head that includes metadata ops and current versions is a vendor blog without methodology. Peer-reviewed cross-runtime FS studies use 2020-era versions and bulk I/O only. **That is a real gap we can fill.**

### 2.4 virtio-fs results depend on at least four knobs (added 2026-11-09)

Source: Goyal, *Some performance numbers for virtiofs, DAX and virtio-9p*, qemu-devel, 2020-12-10 [C, mirror read at lists.gnu.org]. **Setup:** Fedora 29 host, 376 GiB RAM, 2×20 cores×2 threads, host SSD; VM 32 vCPU / 64 GB / 16 GB DAX window; unmerged DAX/killpriv/thread-pool patches; virtiofsd `-o cache=none --thread-pool-size=0`; fio 4K, iodepth 16, 3-run averages. **fio only — the thread contains no metadata (create/stat/rename) results**, so it does not test the metadata claim in §2.1; that evidence is still the 2018 LWN smallfile numbers and Modal's blog.

| Workload (psync, 4 KiB) | 9p-none | vtfs-none | vtfs-none-dax | vtfs-auto | vtfs-always |
|---|---|---|---|---|---|
| seqread (MB/s) | 98.6 | 98.4 | 660.3 | 650.0 | 671.3 |
| randread | 101.5 | 104.9 | 662.5 | 77.9 | 79.3 |
| seqwrite | 93.2 | 95.3 | 486.5 | 78.6 | 75.9 |
| randrw read/write | 70.2/23.5 | 75.8/25.4 | 403.2/134.7 | 38.1/12.7 | 40.7/13.6 |

Reading: the same filesystem moves by ~8× on seqread and ~5× on randrw depending on cache mode and DAX; `cache=auto/always` *halves* randrw versus `cache=none` (the thread attributes write slowness to a per-write GETXATTR via `file_remove_privs()` under cache=auto). Follow-up threads (search snippets only, 2020-09 to 2020-11, Kata/Intel and Red Hat engineers) report: `--thread-pool-size=0` beat 1 or 64 by ~35–40%; 9p still ~30% better than virtiofs on a mixed r/w job; the maintainers called the first Kata comparison "not apples-to-apples", and proposed mapping 9p:none↔virtiofs:none and 9p:loose↔virtiofs:always.

**Rule added (D-33):** every virtio-fs row records `cache=`, `xattr`, DAX on/off + window size, `--thread-pool-size`, `virtiofsd` version, and guest/host kernel. A "Kata virtio-fs is X× slower" number without these is not comparable. Strengthens the case for block-backed primary rows (F1).

## 3. Where agent sessions spend tool time (the counter-evidence)

| Source | Finding | Implication |
|---|---|---|
| AgentCgroup (arXiv:2602.09345). Podman, Claude Code with Haiku 4.5 / GLM-4.7-Flash, 144 tasks | OS-level work = 56–74% of task time. **Init 31–48%** (dominated by Podman user-namespace ID remapping of overlay layers, which scales with image size). Tools ~26%. LLM 26–44%. Within Bash: **tests 72.9% (Haiku) / 43.7% (GLM)**. Python snippets 26.9% (GLM). **Package install ~10%.** Git and file exploration are lightweight. Images 2.9–17.3 GB (avg 3.5 GB) | Phase weights for prebuilt-image tasks: test ≫ install. Init is a first-class phase, and its cost is an FS operation (ID-mapped overlay) |
| 2605.26297 (Claude Code + Harbor + vLLM) | LLM 71–98% of execution time [S] | Slow models hide everything. Crossover curve needed (H-1) |
| Crab | ~5–25% of turns need an FS checkpoint. Up to 87% of turns can skip checkpointing | Most turns don't mutate recovery-relevant state |
| DeltaBox (arXiv:2605.22781) | Edits cluster ≤ 100 KB. Django workdir ≈ 358 MB with "hundreds of thousands of files". 62.0% of MCTS checkpoints were pure-read actions | Small writes into a large, metadata-heavy tree |
| Slacker (FAST'16, P); DSec (A) | Pull = 76% of container start, but only 6.4% of pulled data is read [P]. DSec: runtime touches only **4.2–13.3%** of image data [A] | Lazy loading is the production norm. First-touch reads are host-serviced events during *tests/imports*, not during "install" |
| SOCK (ATC'18, P) | Many popular Python packages take ~100 ms to import. Installs can take seconds | Test phase = Python import storm = metadata-read-heavy |

**Revised belief:** for agent sessions, the FS-sensitive phases are **(1) initialization/restore**, **(2) test/import (metadata reads, lazy first-touch)** and **(3) state operations** (checkpoint/fork/restore). Clone/install matter for *environment build* and for "fresh repo" products (Codex-style setup scripts), which usually cache the result. This changes the phase taxonomy (`10` D-14).

## 4. Industry FS architectures for stateful sandboxes (convergence on block devices)

```
                    file-level sharing                         block-level
                    (host FS visible to guest)                 (guest owns its FS)
gVisor        rootfs overlay (root:self, file-backed)  │
              + gofer/directfs to host files           │
Kata default  virtio-fs → virtiofsd                    │  devmapper / block (option)
Firecracker   (no virtio-fs)                           │  virtio-blk only
Modal VM SB   rejected virtio-fs ("chown >1 min")      │  ImageFS-blk: lazy sparse ext4 + dirty bitmap
Tensorlake                                             │  CoW overlay inside FC virtio-blk, 4 KiB bitmaps
E2B                                                    │  CoW block devices / NBD rootfs (repo README; NBD per secondary)
DeepSeek DSec containers: EROFS lowers from 3FS        │  microVM: OverlayBD over 3FS via ublk, 256 KiB chunks
This session  —                                        │  /dev/vda ext4 rw + RO ext4/squashfs devices (observed)
```

Notes:
- **Observed in this session (2026-11-08):** `/` is `/dev/vda` ext4 (256 G). Tool trees are separate read-only virtio-blk devices (`/opt/claude-code` ext4, `/opt/rclone` squashfs, `/mnt/skills/*` squashfs). No virtio-fs, no overlayfs mounts. Third-party posts from Oct 2025 described Claude Code on the web as gVisor + 9p [C]. Whether it migrated cannot be confirmed and is **not** claimed.
- **Why block wins for stateful VMs (vendor reasoning):** metadata ops stay in the guest (no exit per op). Dirty-block bitmaps make snapshots O(changed bytes). Tensorlake: a 100 MB change gives a 104.9 MB snapshot in 167 ms; E2B pause took 2.90–10.3 s for 1–8 GB written; Modal `snapshot_filesystem` took 17.6–64.6 s [V, competitor-authored — treat as biased].
- **Cost of block:** a duplicated page cache (guest + host) and no host-side visibility into files. DSec: virtio-pmem + DAX cut peak host memory by 40.2% but raised transient peak CPU from 26.5% to 41.4% [A]. Lee et al.: Firecracker's speed comes from the guest page cache [P].

## 5. State operations as a cost term

| Mechanism | Checkpoint | Restore | Source | Grade |
|---|---|---|---|---|
| Firecracker full snapshot (Django) | ~9.46 s | 251 ms | DeltaBox Table 2 | A |
| Firecracker diff snapshot | 475–531 ms | 1.33–1.49 s (chain merge) | DeltaBox | A |
| docker commit | ~49–52 ms | ~1.35 s | DeltaBox | A |
| copytree | — | 153–516 ms | DeltaBox | A |
| CRIU (2 GiB) | 1.445 s snapshot+restore | | Xu et al. 2510.05556 | A |
| Podman+CRIU (2 GiB) | 18.154 s snapshot+restore | | same | A |
| AWS VM snapshot (0 GB mem) | 353 s | | same | A |
| CRIU, 64 × 1 GB concurrent | 47 s | | Crab | A |
| ZFS snapshot under concurrency | ≤ 22 ms | | Crab | A |
| DeltaBox (overlay layer switch + CRIU template fork) | ~14.6 ms | 5.1 ms fast / 8.0 ms slow | DeltaBox | A |
| E2B pause | ~4 s / GiB RAM | ~1 s (secondary) | E2B docs | V |

- DeltaBox MCTS replay: state management = **47–77%** of trajectory time on FC-Diff+dm and CHV+dm baselines, 3–6% with DeltaBox. Crab: per-turn full checkpointing slowed Claude Code **3.06× at 64** and **3.78× at 96** sandboxes per host.
- **Implication for this study:** state ops are where *isolation mechanism × FS design* interact most strongly: microVMs must snapshot guest memory, gVisor snapshots Sentry state, runc needs CRIU. The paper should measure them as a **lifecycle phase** with the same D×A×C accounting (copied bytes, faults after restore). It must **not** propose a new C/R system — DeltaBox and Crab occupy that space.

## 6. Durability and caching semantics (confound)

- Firecracker block `cache_type`: **`Unsafe` is the default**. Uncommitted data stays in the host page cache. `Writeback` fsyncs the backing file on guest flush. Source is a mirror of `docs/api_requests/block-caching.md` [C]; **verify against the pinned Firecracker version**.
- VEE'20: Firecracker's write advantage comes from not writing back [P].
- Modal ImageFS-blk: relies on the guest ext4 journal flushing ~every 5 s [V].
- virtio-fs cache modes (`none`/`auto`/`always`) change metadata cost by ~3× in the 2018 numbers [C].
- **Rule (D-15):** every FS comparison declares its durability level (does `fsync` in the guest reach stable host storage, or only the host page cache?) and its cache mode. The main comparison holds both equal across runtimes. "Vendor default" configs are a second, labelled series.
- Our storage is tmpfs-backed (HDD confound, `04` §3), so "stable storage" is moot for timing. But flush *propagation* (exits, host syscalls) is still a cost and must still be configured the same way.

## 7. Memory and density — the FS path's second cost

- AgentCgroup: memory, not CPU, limits concurrency. ~185 MB framework baseline. Tool bursts up to 15.4× peak-to-average, lasting 1–2 s [A].
- DSec: stable at ≥ 3,200 containers **or** ≥ 800 microVMs per node (~4× gap). CPU QoS: at 50% best-effort load, latency-sensitive step latency +45.2% without controls, +17.3% with SCHED_IDLE + core scheduling [A]. **This is prior art for O5**, which must cite it and differ from it (it measures CPU interference; we would attribute via host-serviced events).
- Double page cache (guest + host) is the memory cost of block designs. Shared-FS designs (virtio-fs DAX, gVisor host files) share the host cache but pay per-op exits/RPCs. **Latency vs density trade-off of the FS path is itself a candidate headline result.**

## 8. How this changes the research direction

What survives:
- The **unit of account** (D×A×C + residual) applies directly. FS metadata ops are the cleanest example: virtio-fs pays A ≈ 1 exit + ~3 round trips per op; virtio-blk pays ~0 per metadata op and an exit per block batch; gVisor pays a trap per op plus host syscalls (IEICE: ≈ 48× host syscalls per `open`).
- **Per-sandbox attribution** and the **folk-number audit**. The Modal table is exactly the kind of number to reproduce with methods.

What changes (decisions in `10-decision-log.md`):
1. **Phases:** `env-build` (clone/install; measured, but not weighted as session time), `init/restore`, `turn:test`, `turn:edit-grep`, `turn:python`, `idle-live`, `state-op` (checkpoint/fork/restore/suspend). Phase weights come from published traces (AgentCgroup, Crab), not from intuition. (D-14)
2. **Runtime matrix:** the FS backend is a first-class axis with at least one *block-backed* config per VM runtime (Kata-CLH-blk, FC-virtio-blk) as primary, because vendors converge there. virtio-fs is a labelled secondary. gVisor default overlay (`root:self`) + directfs is primary. (D-16)
3. **Durability/cache mode declared and held equal.** (D-15)
4. **State ops measured** for each runtime with its native mechanism (runc+CRIU, runsc checkpoint/fs snapshot, FC snapshot full/diff, CHV snapshot). Report bytes copied and restore-fault HSEs. No new C/R design. (D-17)
5. **Density** (resident memory incl. duplicated page cache) reported alongside latency for each FS config.
6. **Brief §2 corrected:** cross-mechanism comparisons exist (VEE'20, Middleware'21, MDPI 2023, Modal 2026). The novelty claim becomes: current versions + metadata/agent workloads + per-sandbox boundary attribution + lifecycle/state ops + controlled FS/durability. (D-18)

What is newly risky:
- **Scoop:** HKUST (AgentSysBench + Crab, shared authors), SJTU IPADS (DeltaBox), eunomia-bpf (AgentCgroup, eBPF + cgroups at tool-call granularity) are all active in 2026. AgentCgroup's tooling is close to our probe design. We must cite it and differentiate: resource *control* vs boundary-cost *decomposition*.
- **Representativeness:** with storage on tmpfs we cannot reproduce lazy remote loading (Modal ImageFS, DSec 3FS, E2B NBD), which production relies on. Options in Q-F3 (`09`).

## 9. Open evidence gaps (actions)

| Gap | Action |
|---|---|
| Modal table: hardware, runs, statistics unknown | Reproduce the syscall microbenchmark (stat/link/chown/hardlink tree/cp -R) and `bun install` on our matrix as part of H-F |
| No agent-trace FS op counts (files opened/stat'd per turn) | Count them ourselves under `native` (demand D) for replayed traces. Publish as a dataset |
| Python import cost per phase | Measure `pytest` collection syscalls per runtime. Prior literature gives only "≥10 stat per module" estimates (C) |
| Firecracker cache_type default for pinned version | Read source/docs at pinned tag |
| E2B storage internals | Read `e2b-dev/infra` source (orchestrator, NBD, CoW) |
| gVisor numbers for overlay/directfs on agent workloads | Measure; vendor blogs give only fsstress / Abseil / "12%" |
| DSec / DeltaBox / Crab full text beyond first 100k chars | Read remaining sections |
| virtio-fs perf thread | **Resolved 2026-11-09:** `listman.redhat.com` no longer exists (Red Hat shut it down); `lore.kernel.org` blocks automated clients. Read the qemu-devel cross-post via `lists.gnu.org` instead (§2.4). It has fio only, no metadata data |
