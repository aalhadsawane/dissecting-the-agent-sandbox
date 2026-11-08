# 02 — Critical review of the brief

**TL;DR**
- The thesis conflates three different quantities: **dollar cost**, **latency**, and **host CPU time**. AgentSysBench's ">99% of cost" is a billing number; the brief's objectives are about time. Pick one per claim.
- O1 as written is either **contradicted** (real model: LLM is 71–98% of time per 2605.26297) or **tautological** (stubbed model: model time ≈ 0). Reframe as a crossover curve.
- A "boundary crossing" is not the same event across runtimes: runc has no separate host handler, VMs handle most syscalls *inside* the guest, gVisor traps every syscall. **Per-crossing** normalization rewards mechanisms that batch. Normalize by **workload demand** instead.
- Some isolation cost produces **no crossing at all** (2-D page walks, duplicated guest kernel work, cache/TLB pollution). The abstraction must report a residual, and the residual may be large. That is a likely disconfirmation — plan for it.
- For clone/install, **filesystem design** (gofer/directfs/overlay, virtio-fs, virtio-blk) probably matters more than the isolation mechanism. It must be a named axis, or O4 will attribute an FS effect to "the mechanism".

---

## 1. Misconception: "the sandbox is the cost center" ⇒ "sandbox time dominates wall-clock"

```
            what AgentSysBench measured            what O1 claims
            ---------------------------            --------------
 dollars  = sandbox_rate × session_wallclock      sandbox_time / e2e_time > 0.5
            (billed while idle, waiting on LLM)    (active execution)
```

- A sandbox billed for 10 min while it idles waiting on the model is expensive in **dollars** and cheap in **CPU**.
- AgentSysBench itself: non-LLM dominates latency in only **5/10** apps; Claude Code on MCP-Atlas is task-dependent (LLM up to 90%, tools up to 84%).
- 2605.26297 (Claude Code + Harbor + vLLM, SWE-bench Pro, Terminal-Bench 2.0): LLM inference ≈ **71–98%** of execution time (snippet; confirm).
- With a **stubbed model**, O1 is true by construction — it measures the stub, not the world.

**Reframe (adopted, see `10-decision-log.md` D-03):**
- Record or reuse real agent trajectories; replay tool calls in the sandbox; inject model latency `L` from a distribution.
- Report `f(L) = T_sandbox / (T_sandbox + T_model(L))` as a **curve** and the crossover `L*` where `f = 0.5`, per workload and per runtime.
- Separately report **idle-but-live** cost as resident memory × time (GB·s), the thing that actually drives the billing result.

This is more useful to a vendor than a single ">50%" yes/no: it tells them at which model speed their sandbox becomes the bottleneck.

## 2. Misconception: a syscall, a seccomp trap and a VM exit are "the same phenomenon"

They are the same *kind* of thing — control leaving tenant-controlled execution for a host-trusted handler — but they occur at different layers with different frequencies:

```
                 app syscall                 who handles it            leaves tenant domain?
runc             open()  ──────────────────► host kernel (seccomp filter + VFS)   yes, every syscall,
                                                                                   but handler = "native" path
gVisor systrap   open()  ─SIGSYS trap──────► Sentry (host process)     yes, every syscall
                          └─► Sentry may issue host syscalls / gofer IPC   (second-order crossings)
gVisor KVM       open()  ─guest ring3→ring0► Sentry in guest mode      no VM exit for the trap;
                          └─► Sentry host syscall ⇒ exit to host mode    exits only on host calls
Kata / FC        open()  ─────────────────► guest kernel               no — unless it causes I/O,
                          └─► virtio kick / HLT / timer / EPT fault ⇒ VM exit
```

Consequences:
- Crossing **counts** differ by orders of magnitude between gVisor (≈ every syscall) and a VM (≈ every I/O batch / idle / timer).
- "Host time **per crossing**" (O3 as stated) is therefore dominated by *granularity*: a VM with few, fat exits can look worse per crossing and better in total.
- runc has no distinct "host handler" — its boundary cost is the **delta** vs. an unconfined process (seccomp BPF filter, namespace/cgroup accounting, overlayfs). Note: runc applies a seccomp profile only if the engine passes one (Docker: default profile on; Kubernetes: only with `RuntimeDefault`). This must be a named config.

**Fix (adopted, D-04):** decompose boundary cost as

```
T_boundary(runtime, workload) = Σ_class  D_c(workload) × A_c(runtime) × C_c(runtime)

D_c  demand      events of class c the workload asks for, measured once under a reference
                 (native process, same kernel) — e.g. file opens, bytes written, packets, page touches
A_c  amplification   host-serviced events per unit of demand under this runtime
C_c  unit cost   host time per host-serviced event of class c
```

- O3's single number becomes `T_boundary / D` (host time per unit of workload demand) — comparable because the denominator is runtime-independent.
- `A_c` and `C_c` are reported separately: that is where the explanation lives ("gVisor pays per call, VMs pay per batch").
- Full definition in `03-unit-of-account.md`.

## 3. Misconception: all isolation cost shows up as crossings

Costs with **no** boundary event:
- 2-D page walks (EPT/NPT) on TLB misses — inflates guest *user* time.
- Guest kernel doing work the host kernel would have done (duplicated page cache, duplicated network stack, guest scheduler).
- vCPU preemption / steal time; lock-holder preemption inside the guest.
- Cache and TLB pollution from handler code on the same core.
- gVisor Sentry's own user-space work after the trap (it *is* attributable, but it is "guest" work in a host process — classify carefully).

**Requirement:** every decomposition reports
```
residual = T_total(runtime) − T_total(native) − T_boundary(runtime)
```
If the residual is a large fraction for VMs on memory-heavy phases (likely for `build`), the abstraction is **incomplete**, and the paper says so. This is the most probable candidate for the "at least one hypothesis disconfirmed" requirement.

## 4. Confound: filesystem design masquerades as isolation mechanism

> **Update 2026-11-08 (round 2):** the original version of this section said FS cost "probably" dominates clone/install. That was a conjecture. `11-stateful-fs-path.md` replaces it with graded evidence:
> - **Supported:** per-metadata-op cost differs ~10–50× across FS paths. Example: Modal reports `bun install` at 12.5 s over virtio-fs vs 1.07 s over virtio-blk [vendor].
> - **Contradicted for prebuilt-image agent tasks:** install is ~10% of Bash time and tests 44–73% (AgentCgroup, arXiv:2602.09345).
> - **New:** initialization/restore and state operations (checkpoint/fork) are FS-design-bound phases the brief did not list.
>
> The table below stays as the *mechanism* description. The *claim* is now the one in `11` §3.

| Runtime (typical config) | Rootfs / workspace path | Host-side handler |
|---|---|---|
| runc | overlayfs on host | host kernel |
| gVisor | gofer (LISAFS) / directfs / rootfs overlay backed by memory or host file | Sentry + gofer |
| Kata-QEMU / Kata-CLH | virtio-fs (virtiofsd) or devmapper block | virtiofsd process / VMM |
| Firecracker | virtio-blk only (no virtio-fs) | Firecracker VMM thread |

- `git clone` and `pip install` are metadata-heavy (many small files). Their cost is dominated by the FS path above, not by "VM vs trap".
- **Rule (D-05):** FS backend is part of the runtime's name, e.g. `gvisor-systrap-overlay=memory-directfs=on`, `kata-clh-virtiofs-cache=auto`, `fc-1.x-virtioblk-ext4`. One FS change = one variable.
- O4's "ranking flips between phases" must be tested **with FS held as close as possible** (e.g. all-block-device variant) *and* with each runtime's default FS. Report both; the gap between them is itself a finding.

## 5. O2: cold start is the wrong comparator in production

- Production platforms hide cold start: GKE Agent Sandbox serves from a **warm pool**; Firecracker vendors restore **snapshots**.
- Snapshot restore moves cost into **post-restore page faults** (REAP: host page faults ≈ 95% of function processing in baseline snapshots). Those faults are themselves host-serviced events.
- **Reframe (D-06):** compare boundary cost against *three* start paths — cold boot, snapshot restore, warm-pool claim — and count post-restore faults inside the unit of account, not inside "cold start".
- O2 as stated (crossings > cold start after ~2 s) is likely trivially true versus warm pools. Keep it as a pre-registered check, but expect the interesting result to be "how much of cold start is hiding in the first N seconds of steady state".

## 6. Firecracker is a VMM, not a container runtime

- runc, runsc, kata are OCI runtimes; Firecracker alone is not (needs firecracker-containerd, Kata-with-FC, or a custom harness).
- Comparing "direct Firecracker" with "containerd → shim → runtime" mixes orchestration cost into one side.
- **Decision (D-07):** measure **steady-state** inside a started sandbox for all four; measure orchestration/start separately and never sum them silently. Candidate: Firecracker driven directly (own kernel + rootfs) as the microVM data point, plus Kata-CLH as the "VM-backed OCI container". Kata-FC optional.

## 7. "The folk numbers are unreproducible" — partly; they are also stale

- Young'19 predates systrap (2023), directfs, and rootfs overlay (2023). Not reproducible because the system changed.
- Firecracker "<125 ms" is boot-to-init with a minimal guest kernel; "agent-ready" (Python + tools loaded) is a different event.
- Stronger contribution than "they are wrong": **re-run the folk measurements with pinned modern versions and explain each discrepancy** (version, platform, FS, warm pool, what event is timed). This is cheap, early, and blog-worthy (H-F in `05-objectives-hypotheses.md`).

## 8. Things the brief assumes but does not state

| Assumption | Risk | Action |
|---|---|---|
| Home server CPU supports VMX/SVM, constant/invariant TSC | Exit reasons and TSC correlation differ Intel vs AMD | Get `lscpu`, `/proc/cpuinfo` flags in week 0 |
| Enough RAM for tmpfs images + repos + caches + N co-tenants | O5 density limited by RAM, not CPU | Get RAM size; budget in `06-infrastructure.md` |
| Home server is truly bare metal | Invariant 1 | `systemd-detect-virt`, `hypervisor` flag, DMI strings |
| AWS student credits can launch `.metal` | AWS Academy Learner Labs restrict instance sizes; metal likely blocked | Check account type before planning around it |
| CVM comparison is feasible | On AWS you are the **guest** of a CVM — no host-side tracing is possible | See `09-open-questions.md` Q2; recommend drop |
| GitHub Actions / Claude container useful for measurement | Both are VMs. This session's container is a **Firecracker guest** (`dmesg`: `--firecracker-init`, "Hypervisor detected: KVM", no `/dev/kvm`) | Use only for analysis CI, never for numbers |
| Citations in §2 of the brief all exist as named | "Does Virtualization Really Incur a 5% Overhead?" not found | See `01-literature.md` §7 |

## 8b. Round-2 findings (stateful sandboxes; details in `11-stateful-fs-path.md`)

| # | Finding | Evidence | Effect on plan |
|---|---|---|---|
| 1 | Agent sandboxes are long-lived stateful sessions. FaaS suites model none of the lifecycle | DSec median life 15–17 min, p99 > 3 h. E2B/Daytona/Vercel/Modal/Fly persistence features | Lifecycle phases replace clone/install/build/test (D-14) |
| 2 | Brief §2 "measured within one mechanism at a time" is **wrong** | VEE'20 (LXC/gVisor/FC), Middleware'21, MDPI 2023, Modal 2026 | Novelty restated (D-18) |
| 3 | Vendors converge on block-device FS for VM sandboxes | Modal, Tensorlake, E2B, DSec, this session's container | Block-backed configs primary (D-16) |
| 4 | Firecracker's default block cache is `Unsafe`. VEE'20 FC write speed comes from skipping write-back | FC docs mirror, VEE'20 | Durability declared and held equal (D-15) |
| 5 | State ops (checkpoint/fork/restore) can dominate tree-search trajectories | DeltaBox: 47–77% on FC/CHV baselines | State-op phase added (D-17) |
| 6 | Init (overlay ID-remap) is 31–48% of task lifecycle under Podman | AgentCgroup | Init is a measured phase. Rootless/ID-mapped config is named |
| 7 | Memory, not CPU, bounds density. Co-tenant CPU interference already measured | AgentCgroup, DSec (+45.2% step latency at 50% BE load) | O5 must cite DSec and add memory pressure |
| 8 | Three groups active on stateful agent sandboxes | HKUST, SJTU IPADS, eunomia-bpf | Scoop risk raised (R1, R15) |

## 9. What is genuinely strong in the brief

- The gap (§3) survives the literature check: no one decomposes isolation cost per sandbox, across mechanisms, under agent phases.
- The invariants are the right ones; invariant 1 is load-bearing and correctly placed first.
- Treating the HDD as a held-constant confound rather than a metric is the correct call.
- "Name the platform/config" (§6) is the single rule that, applied strictly, makes the folk numbers explainable.
