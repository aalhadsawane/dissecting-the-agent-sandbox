# 09 — Questions for the owner (consolidated)

> **Status 2026-11-09:** owner replied "agree with all the recs" and supplied the host evidence pack. Groups **S, F, W, R, A2–A4** are **decided** (`10` D-19). **H1–H5 answered** (see `docs/evidence/host-pack-2026-11-09.md`, analysis `12`). **Open:** A1, C1–C3, C4 (see below), W2-GPU, and the new round-3 questions at the end of this file.

**TL;DR**
- 22 questions in 6 groups. **Blocking before week 1:** H1–H4 (hardware), S1 (stateful scope), F1 (FS matrix), A1 (AWS account).
- Each question has a reason (with evidence pointer) and a recommendation. A short answer like "agree with rec" is enough.

---

## H — Hardware and host (blocking)

| ID | Question | Why | Recommendation |
|---|---|---|---|
| H1 | CPU model, vendor (Intel/AMD), cores/threads, sockets? Output of `lscpu` and the `flags` line of `/proc/cpuinfo` | Exit-reason mapping differs VMX vs SVM. Constant TSC needed for guest↔host correlation (`03` §6) | Paste raw output |
| H2 | RAM size? | tmpfs must hold images + repos + mirrors. Memory, not CPU, bounds agent density (AgentCgroup; DSec 3,200 containers vs 800 microVMs/node) | ≥ 64 GB is comfortable. If ≤ 32 GB, O5 shrinks |
| H3 | Is it truly bare metal, and can I get root, BIOS settings (turbo, C-states, SMT) and a custom/pinned kernel? | Invariants 1 and 3. vhost_task attribution needs kernel ≥ 6.4 | Paste `systemd-detect-virt` and `dmidecode -s system-product-name` |
| H4 | Remote access method, other services/users on the box, and can it run unattended overnight? | Interleaved sweeps, three-strikes reruns, O5 sample counts | Overnight runs strongly preferred |
| H5 | Does the CPU support SEV-SNP or TDX with host access (BIOS + firmware)? | The only way a CVM row could be decomposed host-side (`06` §4) | Only matters if yes |

## S — Scope after the stateful-sandbox findings (`11`)

| ID | Question | Why | Recommendation |
|---|---|---|---|
| S1 | Should **state operations** (checkpoint / fork / rollback / suspend-resume) be in core scope, measured with each runtime's *native* mechanism? | DeltaBox: state management 47–77% of tree-search trajectory time on FC/CHV baselines. E2B pause ~4 s/GiB. Highest interaction between isolation and FS design (`11` §5) | **Yes**, measurement only. No new C/R system (DeltaBox/Crab own that) |
| S2 | Accept the new phase set (env-build, init/restore, turn:test, turn:python, turn:edit-grep, idle-live, state-op) instead of clone/install/build/test? | AgentCgroup: install ~10% of Bash time, tests 44–73%, init 31–48% of lifecycle (`11` §3) | **Yes**. Clone/install kept under env-build |
| S3 | Which agent populations matter most to you: (a) prebuilt-image SWE/RL tasks, (b) "fresh repo" products with setup scripts (Codex/Claude Code web style), (c) tree-search/RL with forking? | Each weights phases differently: (a) test-heavy, (b) env-build-heavy but cached, (c) state-op-heavy | (a) + (c) primary, (b) as a sensitivity case |
| S4 | Keep O1 as the crossover curve `f(L)`? | 2605.26297 says LLM is 71–98% of time; AgentCgroup says OS-level is 56–74%. The gap is explained by model speed and whether init is counted | **Yes** |
| S5 | Keep "host time per unit of workload demand" (D×A×C) instead of "per crossing"? | `02` §2 | **Yes** |
| S6 | CVM in or out? | No host access on cloud CVMs; overlaps 2609.04478 | **Out**, unless H5 = yes |

## F — Filesystem / storage design

| ID | Question | Why | Recommendation |
|---|---|---|---|
| F1 | Make **block-backed** VM configs primary (FC virtio-blk, Kata-CLH block) and virtio-fs secondary? | Vendors converge on block (Modal, Tensorlake, E2B, DSec, this session's container). Modal: `bun install` 12.5 s virtio-fs vs 1.07 s blk [vendor] (`11` §4) | **Yes** |
| F2 | Hold **durability** equal (guest fsync reaches host fsync) in the main table, with vendor defaults as a second labelled series? | FC default `cache_type` = `Unsafe`. VEE'20 FC write speed came from skipping write-back (`11` §6) | **Yes** |
| F3 | Production uses **lazy remote image loading** (Modal ImageFS, DSec 3FS, E2B NBD). Our storage is tmpfs. Choose: (a) state as limitation, (b) emulate with a local lazy block/FUSE server with injected latency, (c) skip | Runtime touches only 4.2–13.3% of image data (DSec), 6.4% (Slacker). First-touch reads are host-serviced events | **(a)** for core, **(b)** as stretch |
| F4 | Is a rootless/ID-mapped config (`runc-idmap`) in scope? | AgentCgroup attributes 31–48% of lifecycle to Podman ID remapping | Optional row, init phase only |

## W — Workloads and data

| ID | Question | Why | Recommendation |
|---|---|---|---|
| W1 | OK to build a **thin trace-replay harness** (not a benchmark suite) over published traces? This is a scoped exception to "don't build a new suite" | No existing suite covers the agent lifecycle. SeBS/vHive stay as calibration | **Yes** |
| W2 | Trace source: (a) public trajectories (SWE-agent / Terminal-Bench / released AgentCgroup or Crab traces if available), (b) self-recorded with a local model, (c) API model | (b)/(c) cost GPU time or money. Licensing of (a) varies | (a) first, (b) if the server has a GPU — does it? |
| W3 | Which repos/languages: Python-only, or also Node and a compiled project? | Python import storms dominate tests. Node is the most metadata-heavy install (Modal's bun/pnpm data) | Python + Node core, one C/Rust build as compute control |

## R — Runtimes

| ID | Question | Why | Recommendation |
|---|---|---|---|
| R1 | gVisor: `systrap` + `KVM` platforms, defaults for FS (overlay `root:self`, directfs on)? | Docs: systrap default, KVM best on bare metal. Overlay/directfs are defaults since 2023 | **Yes**. `directfs=off` optional |
| R2 | Kata hypervisor: Cloud Hypervisor only, or also QEMU? | VMM is a separate variable. Time budget | CLH only. QEMU if time allows |
| R3 | Firecracker: drive it directly (own kernel + rootfs), not via firecracker-containerd? | Removes orchestration confound (`02` §6) | **Yes** |

## A — Accounts, publication, logistics

| ID | Question | Why | Recommendation |
|---|---|---|---|
| A1 | AWS account type (Academy Learner Lab vs normal account with credits) and credit amount? | Learner Labs likely block `.metal`. Decides portability check | Tell me which |
| A2 | Venue/timeline: blog + arXiv first, APSys later? Any hard date (graduation, applications)? | APSys deadline not yet verified | Blog wk 4, arXiv wk 8 |
| A3 | Blog domain/stack, and may posts name vendors whose numbers we audit (Modal, E2B, Tensorlake)? | H-F audits vendor claims. Tone and right-of-reply | Name them, factual tone, notify before publishing |
| A4 | Should the repo stay public from day one? | Scoop risk (R15) vs openness/timestamp | Public. The commit history timestamps the method |

## C — Citations to resolve (from the brief)

| ID | Item |
|---|---|
| C1 | Source of "Does Virtualization Really Incur a 5% Overhead?" (not found under that title) |
| C2 | Exact refs: Zhang et al. I/O virtualization (2010); "Hofstra multi-threaded overhead diagnosis" |
| C3 | Origin of "178k production sessions" for AgentSysBench (not seen in the part read) |
| C4 | Optional: allow `listman.redhat.com` in this session's network settings (virtio-fs performance thread), or read it yourself |

---

## Round 3 — new questions from the host analysis (`12`)

| ID | Question | Why (evidence) | Recommendation |
|---|---|---|---|
| K1 | Kernel: stay on 5.15 (after the pending reboot to -202), HWE 6.8, self-built 6.1/6.18, or do the primary kernel-sensitive work on AWS? | Firecracker validates host 5.10/6.1/6.18 only; 5.15 and 6.8 are off-matrix (`12` §2) | Stay on 5.15, declare it, smoke-test in week 1, AWS cross-check on a validated kernel |
| G1 | Who must consent to: installing runtimes, claiming cores via cpuset partitions, taking CPU 2's SMT sibling offline, governor/turbo windows, a memory cap, one scheduled reboot? Is there a lab admin and a mailing list/etiquette? | 33 sessions from 5 other accounts; all of these are host-wide (`12` §7) | You get written OK from the admin and an announcement for each window |
| G2 | Memory cap for our tmpfs + guests (suggest ≤ 48 GiB, i.e. well under the ~116 GiB available) and a CPU budget (suggest CPU 2 + its sibling + a cpuset of ~4 other P-core threads)? | Shared box; Maricq-style gating needs stable resources | Agree numbers up front |
| G3 | May we change the boot cmdline once (e.g. `vmscape=off`) for H-M? Or only do that on AWS? | It is a security-relevant change on a shared host | Only on AWS metal |
| G4 | Who set `isolcpus=2 nohz_full=2 rcu_nocbs=2`, and does anyone use CPU 2 today? | We may be colliding with an existing use | Find out before claiming it |
| E1 | Run the read-only evidence commands in `12` §9 and paste output (SMT sibling of CPU 2, `kvm_intel` parameters, BTF, cgroup v2/cpuset, storage type, mitigation states, tool versions, TSC adjust, GPU) | Needed to finalize placement, probes and the workload store | Run them |
| E2 | Is the pasted pack's usernames/endpoint redaction acceptable (we keep only aggregate counts in the public repo)? | Public repo; other people's identifiers | Yes; keep the unredacted pack private |
| A1 | ~~AWS account type and credits~~ **Closed 2026-11-09: AWS student/builder account with no credits → AWS dropped (D-31)** | — | — |
| C4 | ~~listman blocked~~ **Closed 2026-11-09:** listman.redhat.com was shut down by Red Hat; lore blocks bots. I read the thread via a qemu-devel mirror (`11` §2.4). Nothing needed from the owner | — | — |

## Round 4 — new questions (no AWS, `12` §6b)

| ID | Question | Why | Recommendation |
|---|---|---|---|
| X1 | Does the lab (or BITS Goa) have **another bare-metal machine**, ideally a server Xeon/EPYC, that you may use or even reboot? (The host pack suggests the lab runs more than one server.) | Only free route to a different CPU and to a mitigation-toggle experiment (H-M) without touching the shared HOST-A | Ask the admin |
| X2 | Do you have a **faculty advisor/sponsor** who could create a CloudLab (or Chameleon) project and add you? Is your institution accepted? (CloudLab page doesn't say; Chameleon limits international PIs to those with US collaborations.) | Exclusive server hardware, root, own kernel, free for research/education | Ask your paper's supervisor |
| X3 | Accept H-X (P-core vs E-core on HOST-A) as the baseline second data point, plus an explicit single-host claim? | Zero cost, no reboot; weak proxy | Yes |
| X4 | Accept a K3 dual-boot entry (Firecracker-validated 6.1/6.18) only if week-1 smoke tests on 5.15 fail? Requires admin/physical rollback plan | Without AWS there is no independent kernel to cross-check on | Yes, conditionally |
| E1 (repeat) | Run the read-only commands in `12` §9 and paste output | Still needed | Run them |
