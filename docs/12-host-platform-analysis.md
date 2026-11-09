# 12 — Host platform analysis: what the real hardware does to the study

Written 2026-11-09 from the owner's evidence pack (`docs/evidence/host-pack-2026-11-09.md`, "HOST-A"). Method: take each assumption the earlier docs made about "the home server", test it against the pack and against sources, and record whether it **survives, weakens, or fails**.

**Evidence grades** (as in `11`): **P** peer-reviewed · **A** arXiv · **V** vendor/project docs · **K** kernel docs/source · **C** community/mailing list · **O** owner-reported host fact (unverified by me) · **I** my inference (stated as such, to be tested).

**TL;DR**
- **Bare metal: confirmed (O).** VMX with EPT, invariant TSC, `/dev/kvm` usable, 128 GiB RAM. Invariant 1 holds. No TDX/SEV/IOMMU, so the CVM row is dead (H5 closed).
- **"Quiet": not established, and probably not true.** 33 sessions from 5 other accounts, turbo on, `powersave` governor, hybrid cores, one isolated CPU. Even *exclusive* CloudLab servers show per-benchmark CoV up to ~10% (Maricq OSDI'18 [P]). So the question isn't "is it quiet" but "can we measure and gate on the noise". Plan: a canary + acceptance protocol (§4).
- **Three of my earlier assumptions fail:**
  1. *"vhost attribution needs kernel ≥ 6.4"* — wrong as stated. In v5.15 the vhost worker is a kthread that is moved into the owner's cgroups (`cgroup_attach_task_all`) [K]. And Firecracker does not use vhost at all (I, to verify), Cloud Hypervisor doesn't by default (I). The "blocker" only touches the optional Kata-QEMU row.
  2. *"Upgrade to the 6.8 HWE kernel to fix things"* — doesn't help the main consumer. Firecracker's **validated host kernels are 5.10, 6.1 and 6.18** [V]. Neither 5.15 nor 6.8 is on the list.
  3. *"We can use guest-side hardware counters for the guest-kernel tax"* (`03` §6, plan wk 7) — KVM **disables the vPMU on Intel hybrid hosts**, so guests get no PMU on this CPU [K/C].
- **A new variable I hadn't modeled: VMScape mitigation.** The pack shows `Vmscape (IBPB before exit to userspace)`. The kernel conditionally issues IBPB on the first return to userspace after a VM exit. That inflates exactly the exit classes that go to a userspace VMM (and, I predict, gVisor-KVM's host calls), so the unit cost `C_c` depends on mitigation state. It must be pinned, reported, and ideally varied (H-M, exploratory).
- **External validity is the biggest scientific weakness of this host.** A hybrid client Alder Lake with shallow C-states (C1/C2 only) and a BIOS nobody can change remotely isn't what vendors run. **Update 2026-11-09 (round 4): the owner has no AWS credits, so there is no paid second host.** The cross-check options are now: another machine in the owner's lab, a faculty-sponsored CloudLab/Chameleon project, a within-host P-core vs E-core comparison, or an explicit single-host claim (§6b).
- **Governance, not engineering, is the critical path:** a pending reboot on a shared box, runtime-global knobs (turbo, governor, SMT, CPU hotplug), root-daemon installs (containerd, Docker/Kata/gVisor) — all need the other tenants' consent (§7).

---

## 0. Decisions recorded from the owner's reply
"Agree with all the recs" resolves, as **decided** (logged in `10`, D-19…):
S1 state ops in scope (measure native mechanisms only) · S2 new phase set · S3 prebuilt-image SWE/RL + tree-search primary, fresh-repo as sensitivity · S4 O1 crossover curve · S5 cost per unit of workload demand · S6 CVM out · F1 block-backed VM rows primary · F2 durability held equal · F3 lazy-loading = stated limitation (emulation stretch) · F4 `runc-idmap` optional, init phase only · W1 thin trace-replay harness (scoped exception) · W2 public traces first · W3 Python + Node core + one compiled control · R1 gVisor systrap + KVM with default FS · R2 Cloud Hypervisor only · R3 Firecracker driven directly · A2 blog wk 4 / arXiv wk 8 · A3 vendors may be named, factual tone, advance notice · A4 repo public from day one.

**Still open:** C1–C3 (citations), W2's GPU sub-question (pack doesn't mention a GPU; `nomodeset` suggests none; ask). **Closed in round 4:** A1 (no AWS credits → AWS dropped, D-31); C4 (the virtio-fs thread was read through a qemu-devel mirror, §8).

## 1. Assumption audit

| # | Earlier assumption | Verdict | Evidence | Consequence |
|---|---|---|---|---|
| 1 | Host is bare metal | **Holds** | `systemd-detect-virt=none`, no hypervisor flag, MSI board (O) | Invariant 1 satisfied. Re-check at every session start and store the output |
| 2 | Host is quiet enough for sweeps | **Unproven** | 33 sessions; load avg 0.5 from a single snapshot; Maricq: CoV up to ~10% even on exclusive nodes [P] | Noise baseline + acceptance gates (§4) |
| 3 | Intel VMX exit-reason mapping | **Holds** | `vmx` present, `svm` absent (O) | Use VMX exit reasons only; no AMD column |
| 4 | Constant TSC for guest↔host correlation | **Holds, needs a check** | `constant_tsc nonstop_tsc tsc_known_freq tsc_adjust art` (O) | One-off per-core `IA32_TSC_ADJUST`/offset check (hybrid part) before any correlation claim |
| 5 | RAM large enough for tmpfs + co-tenants | **Holds, with a swap caveat** | 128 GiB, 116 GiB available, but 8 GiB swap is on (O) | tmpfs and guest RAM are swappable → set `memory.swap.max=0` on our cgroups (Firecracker's own prod guide says disable swap [V]) |
| 6 | Can pin a quiet core | **Weak** | Only CPU 2 isolated; sibling thread probably CPU 3 (I; pack lacks `thread_siblings_list`) | A VM needs ≥ vCPU + VMM + probe + harness threads. One core is not enough (§3) |
| 7 | Frequency is a constant | **Fails** | turbo on; ratios 4.9 → 4.5 GHz depending on active cores; governor `powersave`; observed P 4.5 / E 3.4 GHz (O) | Frequency depends on other tenants' load → time comparisons confounded. Record APERF/MPERF per run; ask to fix governor/turbo in windows (§3.3) |
| 8 | Deep C-state experiments possible | **Not possible** | cpuidle only POLL/C1/C2; CFG Lock = 1 (O) | Good for timer determinism; but HLT-exit→wake costs (class TMR) measured here reflect *shallow* idle only → limitation |
| 9 | Kernel ≥ 6.4 needed for vhost attribution | **Fails** | v5.15 `vhost.c`: worker `kthread_create` + `cgroup_attach_task_all(owner, current)` [K, fetched from the v5.15 tag] | See §2. Attribution works on 5.15 if the cgroup exists *before* the VMM opens the vhost device (I; test) |
| 10 | A newer kernel helps | **Weak** | Firecracker validates host 5.10, 6.1, 6.18 only [V] | Choose kernel against the runtimes' support matrices, not "newest" (§2) |
| 11 | Guest hardware counters available | **Fails** | KVM disables vPMU on hybrid; reaffirmed by Intel at KVM Forum 2024 [K/C] | Guest-side tax via timestamps/software counters only; host-side `perf` on vCPU threads still works |
| 12 | Mitigation state is a footnote | **Fails** | `Vmscape: IBPB before exit to userspace` active (O); kernel doc: Alder Lake and newer are affected; IBPB issued before the first exit to userspace after a VM-exit [K] | Pin, report, and vary (§5) |
| 13 | BIOS knobs can be set (SMT, turbo, C-states) | **Mostly fails** | Desktop board, no BMC; BIOS changes need physical access (O) | Only runtime knobs: sysfs governor/no_turbo/SMT/CPU online, cpuset partitions |
| 14 | HDD confound handled by tmpfs | **Unknown storage** | Root is LVM; device type not in pack | Ask `lsblk -d -o NAME,ROTA,MODEL` (§9) |
| 15 | ECC protects long runs | **Fails** | Non-ECC DDR5 (O) | Mandatory output-hash verification per run (already in `04`); repeat runs; no `edac` signal |
| 16 | Same-hardware-class reproducibility | **Needs rewording** | Hybrid client part; Firecracker/AWS fleets are server CPUs | Define classes (§6) |

## 2. Kernel: what we actually need and what is supported

**Facts (V = Firecracker `docs/kernel-policy.md`, fetched 2026-11-09):**

| Host kernel | First supporting Firecracker | "minimum end of support" |
|---|---|---|
| 5.10 | v1.0.0 | 2024-01-31 |
| 6.1 | v1.5.0 | 2025-10-12 |
| 6.18 | v1.16.0 | 2028-05-28 |

"Other kernel versions and configs may work, but Firecracker does not regularly test them, so behavior may be unexpected." So: **5.15 (running) and 6.8 (HWE candidate) are both off-matrix.** The 6.1 row's end-of-support date has already passed as of today; effectively the validated choices are 5.10 (long past) and 6.18.

**What is a "need" and what is not:**

| Need | Needs newer kernel? | Evidence |
|---|---|---|
| `kvm:kvm_exit/kvm_entry/kvm_userspace_exit` tracepoints, `perf kvm stat` | No | Present for years; verify on host (§9) |
| eBPF ring buffer, fentry/fexit, cgroup-id helper, CO-RE | No (≥ 5.8) | Needs `/sys/kernel/btf/vmlinux`; Ubuntu ships it on generic kernels (I; verify) |
| Per-sandbox vhost worker CPU attribution | **No** | v5.15 attaches the vhost kthread to the owner's cgroups [K]. Caveat (I): attachment is done once at `SET_OWNER`; a later move of the VMM process is not followed → create the cgroup first, then exec |
| Firecracker/CHV/Kata running at all | No (probably) | CHV and Kata have their own minimums; Kata doc search was inconclusive → **TBD** |
| Firecracker *officially supported* | **Yes: 5.10 / 6.1 / 6.18** | [V] |
| Representativeness of "what vendors run" | Unknown | No source found; do not claim |

**Options:**

| Option | Cost | Pro | Con |
|---|---|---|---|
| K1: stay on 5.15 (after the already-pending reboot to -202) | Low; reboot already owed | No new risk to other users; Ubuntu security backports include VMScape (O) | Firecracker unvalidated; must state it in the paper |
| K2: Ubuntu HWE 6.8 | Reboot + package install | Distro-supported | Still off Firecracker's matrix; changes mitigation behavior; no clear gain |
| K3: self-built or mainline 6.1 LTS / 6.18 | Reboot + build + boot-loader work on a shared box | On Firecracker's matrix | Highest governance cost; Secure Boot is disabled (O) so unsigned kernels would boot; rollback plan needed |
| ~~K4: primary or cross-check work on AWS x86 metal~~ | **Dropped 2026-11-09: no credits (D-31)** | — | — |

**Recommendation (to be confirmed by owner, Q-K1):** start on **K1** (no extra disruption) and declare Firecracker's off-matrix status. Run the week-1 smoke tests (boot, snapshot full/diff, balloon, block I/O) on 5.15 *before* committing and record any failures as findings. Choose **K3** (a Firecracker-validated 6.1 or 6.18 kernel, ideally as an extra boot entry so 5.15 stays the default) only if smoke tests fail or a result looks like a kernel artifact. *Critique of my own rec:* K1 risks building a month of data on a configuration Firecracker's authors don't test, and without AWS there is no independent kernel to cross-check on. K3 as a dual-boot entry is therefore the realistic hedge, but it needs admin/physical access and a rollback plan (§7).

## 3. Placement, isolation and frequency on a shared hybrid host

### 3.1 What the topology gives
```
 CPU ids    core        threads  max GHz   notes (O)
 0-15       P0..P7      2 each   4.8-4.9   CPU 2 = isolcpus/nohz_full/rcu_nocbs; sibling presumably CPU 3 (I)
 16-19      E8..E11     1 each   3.6       no SMT; shared L2 cluster
 L3 25 MiB shared by all; 1 NUMA node; 2-channel DDR5-4800
```
First-principles consequences:
- Sandbox threads must be **pinned to P-core threads only**; vCPU migration between P and E cores would silently mix two microarchitectures.
- **Noisy-neighbor (O5) placement becomes a designed variable**, not an accident:
  - *same-core SMT sibling* (worst case: shares L1/L2, ports),
  - *different P-core* (shares L3/ring/memory),
  - *E-core* (shares only L3/memory with P-cores, plus its own L2 cluster).
  This is a genuinely richer O5 than the brief's "N neighbors" and cheap to run. *Caveat (I):* unknown whether the other tenants' load already lands on those cores; we need our own cpuset to keep them out.
- E-cores for harness/probes (keeping them off the P-core under test) is attractive but changes probe cost; measure it (probe overhead `T0/T1/T2` per placement).

### 3.2 Runtime isolation without a reboot
- The pack shows only CPU 2 isolated at boot. The cgroup v2 cpuset controller offers `cpuset.cpus.partition` = `root` / `isolated`: CPUs in an isolated partition get "no load balancing from the scheduler" and are excluded from unbound workqueues [K, docs.kernel.org cgroup-v2]. That lets us claim a set of cores at runtime. **Not established by the doc text I read:** whether it also gives `nohz_full`/`rcu_nocbs`-like tick/RCU isolation (I think not; verify). So CPU 2 stays special: the only core with tick and RCU-callback isolation.
- Claiming N cores removes them from other tenants → needs consent (§7). Note that it only affects the scheduler; interrupts need affinity changes, and shared L3/memory/power budget stay shared.
- Taking CPU 2's SMT sibling offline (runtime, sysfs hotplug) would make CPU 2 a *whole-core* quiet resource. Affects other tenants minimally (one hyperthread). Needs root.

### 3.3 Frequency
- `intel_pstate` + `powersave` + turbo on: effective frequency is a function of active-core count (4.9 GHz with 1 active core → 4.5 GHz with 8, per `MSR_TURBO_RATIO_LIMIT`, O) and thermals. Other tenants' activity moves our frequency.
- Options, in increasing intrusiveness: (a) record APERF/MPERF (`MSR 0xE8/0xE7`) per run and per core, report frequency-normalized and raw times (cheap, no consent beyond MSR read); (b) governor `performance` for our cores during windows; (c) `no_turbo=1` (global; affects everyone); (d) reboot with `intel_pstate=passive`/fixed ratio (not possible without physical/BIOS—out).
- *Self-critique:* option (a) cannot fix the confound, only expose it; a time-based claim "runtime X is 8% slower than Y" is invalid if X ran at 4.9 GHz and Y at 4.5. Interleaved randomized ordering (already in `04` §6) averages this out only if frequency noise is independent of config — which fails if one runtime uses more cores and therefore lowers the turbo ratio for itself. Report **cycles (host PMU) alongside seconds** for host-side work; use seconds for tenant-visible latency.

## 4. Noise: measure it, gate on it

Maricq et al. (OSDI'18, 835 CloudLab servers, ~900k data points, exclusive access) [P, text read]:
- "Some degree of variation in hardware performance is unavoidable … Coefficients of variance of up to 10% may be attributed to hardware variability and considered expected."
- Performance data "does not follow the normal distribution" → use nonparametric statistics (median and rank-based CI) [already adopted].
- Repetitions: CONFIRM estimates the number of repetitions needed for a median CI within r = 1% at 95%; most configurations with CoV up to ~4% need only tens of repetitions, outliers need hundreds; "if we stop after 10 measurements in the 10-server case, our reported median values will be outside of the 95% CIs … after the recommended 58–68 measurements."
- Their data is from *exclusive* machines. Ours is shared and hybrid, so their numbers are a **floor**, not an estimate.

**Protocol (plan, no code yet):**
1. *Noise baseline, week 0–1:* run a fixed set of **canary micro-benchmarks** on the cores we will use, every N minutes for ≥ 7 days (covers weekday/weekend and long-job periods): null-syscall loop, fixed-work integer loop (cycles + seconds), pointer-chase (L3/DRAM latency), STREAM-like copy (memory bandwidth), `rdtsc` vs `CLOCK_MONOTONIC_RAW` consistency. Log APERF/MPERF, temperatures, load, and PSI (`/proc/pressure/*`).
2. *Quantify:* per-canary CoV and CONFIRM-style repetition estimates; correlate with time-of-day and with co-tenant job starts (aggregate counts only; no identities).
3. *Acceptance gate:* every experiment batch is bracketed by canaries; a batch is **flagged** (not silently dropped) if the bracketing canary deviates beyond a threshold fixed *before* looking at results (provisional: > 2× the baseline MAD, to be set in week 1 from step 2). Flagged batches are re-run, and both versions are kept in `raw/`.
4. *Choose windows:* use the baseline to select low-noise windows; overnight is the likely best, but only if step 1 shows it.
5. *Canary blind spots (self-critique):* a CPU canary is blind to cache/memory-bandwidth contention and to I/O; hence the pointer-chase and copy canaries. It is also blind to interference that only affects exits (e.g. hypervisor-specific paths) — add a minimal KVM canary (null-exit loop via a tiny test VM) once the stack runs.

## 5. VMScape: a mitigation that changes the unit cost

**What the kernel doc says [K, docs.kernel.org/admin-guide/hw-vuln/vmscape.html]:**
- Affected Intel parts: Skylake generation (no Enhanced-IBRS), Cascade Lake, and "**Alder Lake and newer**" (BHI-affected). Ice Lake-style BHB-clearing parts are listed as *not* vulnerable.
- Behavior: "issues an IBPB before the first exit to userspace after VM-exit", only if a potentially malicious guest ran on that CPU; "If userspace did not run between VM-exit and the next VM-entry, no IBPB is issued."
- Control: `vmscape=off | ibpb | force` (boot parameter); status in `/sys/devices/system/cpu/vulnerabilities/vmscape`. The doc states no performance numbers.
- Related: an Intel patch series (Sept 2025, reported by Phoronix) proposes replacing IBPB by a branch-history-clearing sequence on Alder Lake and newer because IBPB is "overkill" there [C, snippet-level; I did not see charts]. The mitigation is therefore **still moving between kernel builds**.

**Implications for the study (I = inference; each is testable):**
1. *Which exits pay:* in-kernel-handled exits (EPT violation, HLT, timer MSR, ioeventfd-signalled doorbells) don't return to userspace → no IBPB. Exits handled by the userspace VMM (Firecracker/CHV/QEMU device emulation such as PIO or MMIO not covered by ioeventfd) do (I).
2. *gVisor-KVM:* the Sentry runs in guest mode and reaches the host by exiting to the host-side `runsc` process → every host call in the KVM platform may incur the IBPB (I). That would raise exactly the cost of the "gVisor-KVM" row that the docs recommend for bare metal. Systrap is not affected in the same way because it is not a VM.
3. *Comparability:* per the kernel doc, Ice Lake-class parts are not vulnerable while Skylake/Cascade Lake and Alder Lake+ are → any cross-host comparison of `C_c`, and any comparison with published numbers, mixes mitigation regimes unless the sysfs state is reported (the manifest must include it).
4. *Unit-of-account:* split classes by **handler locus**: `KERNEL-handled` vs `USERSPACE-handled` exits. This was implicit; now explicit in `03` (to update).
5. *Experiment (exploratory H-M):* measure `C` for userspace-handled exits with `vmscape=ibpb` (current) vs `off`. Needs a reboot with a changed cmdline → only with admin consent on a shared host, or on another host we control (§6b); otherwise report as an unquantified caveat. *Self-critique:* turning a security mitigation off on a shared multi-user host is security-relevant; only in a bounded window with the admin's consent. The no-reboot alternative is §6b's P-core vs E-core comparison, which varies the mitigation set (RFDS) and microarchitecture without a boot change.

## 6. External validity: hardware class

| Property | HOST-A | Typical vendor fleets (I; unverified here) |
|---|---|---|
| CPU | Client hybrid Alder Lake, 8P+4E | Server Xeon/EPYC, or Graviton |
| Memory | 2-ch DDR5-4800, non-ECC | 8–12 ch ECC |
| Idle states | C1/C2 only (locked) | Often configurable |
| Mitigations | eIBRS + BHI_DIS_S + RFDS + VMScape IBPB | Varies by part |
| vPMU in guests | Disabled by KVM (hybrid) | Available |
| Tenancy | Shared, 33 sessions | Exclusive metal for the study; production = multi-tenant |

Consequences:
- Absolute numbers will not transfer; **ratios and class-structure (A, which exits occur) are more likely to transfer than C (unit costs)**. *Hypothesis, not fact.* Without a second host this can only be tested within the host (P vs E cores, §6b) or against published counts; the paper must say that cross-host transfer is untested.
- Redefine the DoD item "a stranger with the same hardware class": **(a)** same *class of host* = Intel VMX, bare metal, invariant TSC, stated mitigation state; **(b)** the headline table is regenerated from the committed raw data in CI, and the *runbook* regenerates raw data on any such host; **(c)** we claim reproducibility of *structure and ordering*, not of absolute times.
- ~~AWS candidates~~ **Removed 2026-11-09:** the owner has no AWS credits. The price research (third-party figures, c5/c6i/c7i/m5 metal at roughly $4–5.4/h) is kept in `references.md` only as a record of why a paid cross-check was out of reach.

## 6b. Zero-cash options for a second data point (round 4, 2026-11-09)

Owner: no AWS credits; AWS dropped (D-31). What can still address "one hybrid client CPU is not enough"?

| Option | Gives | Cost / blocker | Evidence |
|---|---|---|---|
| X1: another bare-metal machine in the owner's lab | A different CPU (ideally a server Xeon/EPYC) and maybe a kernel we may change | Needs the admin's consent; the pack suggests the lab runs more than one server (a tenant's job reports to a different lab host) — **ask what exists** | O (indirect) |
| X2: CloudLab project (exclusive bare metal, free for research/education) | Server CPUs, root, own kernel, exclusive use; the platform Maricq et al. measured | Students **cannot lead a project**; "ask their advisor or course instructor to create a new project"; staff review "may take a few days"; project leader approves each member. Whether non-US institutions are accepted is **not stated** on the page I read | V (docs.cloudlab.us/users.html) |
| X3: Chameleon bare metal | Similar | PI status: students "not typically considered eligible"; international PIs only "with ongoing, active collaborations with scientists in the US"; a faculty advisor can create a project and add the student; startup allocation 20,000 SU/6 months (1 SU = 1 hour of a base bare-metal server); whether use is free was **not stated** | V (chameleoncloud.readthedocs.io PI eligibility; FAQ via search) |
| X4: P-core vs E-core on HOST-A | Two microarchitectures (Golden Cove vs Gracemont) and, I believe, different mitigation sets (RFDS affects Atom-class cores only — **my knowledge, verify** against the kernel's RFDS doc and the per-core sysfs) on the same board, kernel and memory | Free, no reboot. Caveat: same memory system, same kernel; vPMU off; E-cores have no SMT; is a *weak* proxy for a different vendor/microarchitecture | I |
| X5: single-host claim, stated plainly | — | Honest, cheap; limits generality | — |
| X6: compare our A (HSE counts) against exit/syscall counts published for other hardware | A cross-check of *structure* only | Few papers report per-class exit counts (2609.04478 discusses exit types qualitatively) | A |

**Recommendation:** run **X4** (cheap, falsifiable: "A is core-type-invariant, C is not") and **X5** as the baseline claim; ask the owner about **X1**; pursue **X2** only if a faculty sponsor exists and can approve (the owner's paper on scheduling and attestation suggests there may be one → ask). Reserve H-M for X1/X2; do not do it on the shared HOST-A without admin consent.

**New exploratory hypothesis H-X:** on HOST-A, for a fixed sandbox workload pinned to P-core threads vs E-core threads, the per-class HSE counts `A_c` agree within bootstrap CIs, while unit costs `C_c` differ. *Disconfirmed if:* `A_c` itself differs (then structure is not hardware-independent, which would be a surprising and important finding), or `C_c` doesn't differ (then C may transfer better than assumed). *Caveats:* E-cores may have different KVM behavior and timers; kernel scheduling must be prevented from migrating vCPUs.

## 7. Governance: what needs other people's consent

| Action | Scope | Reversible? | Needs |
|---|---|---|---|
| Install containerd/runc/runsc/Kata/CHV/Firecracker (root daemons, iptables/nft, cgroups, `kvm` group use) | host-wide | Yes | Operator + other tenants informed |
| Create cgroup/cpuset partitions claiming cores | host-wide scheduling | Yes (instant) | Consent; announce windows |
| CPU hotplug (offline SMT sibling of CPU 2) | host-wide | Yes | Consent |
| Governor `performance`, `no_turbo=1` | host-wide, affects everyone's performance and thermals | Yes | Consent; windows only |
| `drop_caches` (NOPASSWD already allowed) | host-wide page cache | n/a | Avoid; hurts others |
| Memory use (tmpfs up to X GiB) | host-wide | Yes | Cap agreed up front |
| Reboot (pending anyway) | all 33 sessions | n/a | Scheduled window |
| Changing boot cmdline (e.g. `vmscape=off`) | host-wide, security-relevant | Yes | Explicit admin approval |
| Running microVM/sandbox runtimes whose purpose includes isolation testing | security posture | n/a | Non-goal: **no escape testing** (brief §6) — also protects the other tenants |

*Critique of the plan:* the earlier docs wrote "pin governor, turbo off, SMT fixed" as if free. On this host each is a negotiation. The honest fallback is to **measure the confounds (APERF/MPERF, canaries, PSI) rather than remove them**, and to say so.

## 8. Network access (owner said "full network access") — RESOLVED in round 4
Round-4 outcome: `listman.redhat.com` was shut down by Red Hat (owner), so its failure is expected and permanent, not a policy block. `lore.kernel.org` rejects automated clients as bots. I did **not** try to work around that. The same virtio-fs thread was cross-posted to qemu-devel and is readable at `lists.gnu.org` (see `11` §2.4). Nothing is needed from the owner for virtio-fs. Original round-3 text follows for the record.

### (round-3 text, superseded — and partly wrong)
**Correction:** I attributed the failure to the session's network policy. That was a misdiagnosis: the tool's error text suggested it, but the actual cause is that the service no longer exists (DNS `ENOTFOUND`, proxy 502). Lesson: an error message that offers one explanation isn't evidence for it.
Verified this session: `usenix.org` PDFs download; `lore.kernel.org` returns HTTP 403; **`listman.redhat.com` still fails (proxy CONNECT 502)**, and `WebFetch` reports the host as outside the session's allowed domains. So the owner's statement doesn't (yet) cover that host in *this* session. Not blocking: the virtio-fs thread is a secondary source. Action: owner either allows `listman.redhat.com` in the session network settings or pastes the text.

## 9. Evidence requests (run on HOST-A, paste output; read-only)
```bash
cat /sys/devices/system/cpu/cpu2/topology/thread_siblings_list
grep -m1 '^flags' /proc/cpuinfo ; grep -m1 'vmx flags' /proc/cpuinfo
grep -H . /sys/module/kvm_intel/parameters/*
ls -l /sys/kernel/btf/vmlinux; mount | grep -E 'cgroup2?'; cat /sys/fs/cgroup/cgroup.controllers
lsblk -d -o NAME,ROTA,TYPE,MODEL,SIZE; findmnt -no FSTYPE,SOURCE,AVAIL /
grep -H . /sys/devices/system/cpu/vulnerabilities/*
cat /sys/devices/system/cpu/smt/control; cat /sys/devices/system/cpu/intel_pstate/status
lspci | grep -i -E 'vga|3d|nvidia'
perf --version; bpftrace --version; clang --version | head -1; pahole --version
sudo rdmsr -a 0x3b      # IA32_TSC_ADJUST on every CPU
ulimit -a | head -20 ; cat /proc/pressure/{cpu,memory,io}
```
These are read-only probes. They add: SMT-sibling topology, APICv/PML/EPT module parameters (they determine which interrupt and dirty-logging exits exist), BTF availability, cgroup v2 state, storage type, per-vulnerability mitigation states, tool versions, TSC offsets, baseline pressure.

## 10. Consequences for the other documents
- `03`: correct the vhost/6.4 statement (§5); split exit classes by handler locus (kernel vs userspace) — done in this commit.
- `04`: manifest gets host-specific fields; noise protocol §4 added; "pin governor/turbo/SMT" reworded as "pin if consented, otherwise measure".
- `05`: add **H-M** (mitigation sensitivity, exploratory) and a **placement** sub-hypothesis for H-6 (SMT-sibling vs P-core vs E-core neighbor).
- `06`/`07`/`08`/`09`/`10`: updated for host facts, plan (week 0 noise baseline, week 1 smoke tests on the off-matrix kernel), risks R20–R28, new questions, decisions D-19…D-30.
