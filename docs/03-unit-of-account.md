# 03 — The unit of account: host-serviced events (HSE)

**TL;DR**
- Define one event type for all runtimes: a **host-serviced event (HSE)** = control leaves code the tenant controls and enters code the host trusts, on behalf of one sandbox.
- Classify HSEs into a fixed taxonomy (syscall-emulation, storage I/O, network I/O, memory, timer/idle, interrupt/IPI, other). Every runtime-specific event maps to exactly one class.
- Cost per sandbox = Σ over classes of **demand × amplification × unit cost** (`D × A × C`), plus a reported **residual** for overhead that never crosses.
- Attribution to a sandbox = **cgroup** of the thread that runs the handler (vCPU thread, VMM thread, Sentry, gofer, virtiofsd), plus explicit handling for kernel threads.
- This document is a proposal to be defended in the paper. Every "verify" marker is a fact to confirm against source code or a probe before relying on it.

---

## 1. Definition

```
HSE := (sandbox_id, class, t_enter, t_return, handler_cpu_time, handler_identity)

  tenant-controlled code       : app code, guest kernel, gVisor-sandboxed app
  host-trusted code            : host kernel, KVM, VMM (FC / QEMU / CLH), Sentry*, gofer, virtiofsd
  t_enter / t_return           : when control left / came back to tenant code (synchronous part)
  handler_cpu_time             : CPU time on host-trusted code caused by the event,
                                 including asynchronous work on other threads (e.g. VMM I/O thread)
```

\* The gVisor Sentry is host-trusted from the host's view (it is the isolation layer) but runs as an unprivileged host process. It is counted as a handler. Its time is split into "emulation" (Sentry's own work) and "pass-through" (host syscalls the Sentry issues).

Two time measures, always both:
- **Synchronous latency** — `t_return − t_enter`; what the tenant waits for. Drives latency (O2, O4, O5).
- **Handler CPU time** — including async threads; what the host pays. Drives density/cost (O3).

They diverge for VMs: a virtio kick returns fast (ioeventfd), the VMM thread does the work later.

## 2. Taxonomy (fixed before measurement; pre-registered)

| Class | runc | gVisor systrap | gVisor KVM | Kata (QEMU/CLH) | Firecracker |
|---|---|---|---|---|---|
| **SYS** syscall-emulation | n/a (native) — seccomp filter delta only | every app syscall → SIGSYS → Sentry | app syscall → Sentry in guest ring0 (no exit) | n/a (guest kernel) | n/a (guest kernel) |
| **STO** storage I/O | host VFS via overlayfs | Sentry → gofer IPC / directfs host syscalls | same, via bluepill exits | virtio-fs (virtiofsd) or virtio-blk kicks + VMM/vhost work | virtio-blk MMIO kick + FC block thread |
| **NET** network I/O | host netstack (veth) | netstack in Sentry → host fd / AF_PACKET | same | virtio-net, vhost-net kernel thread (QEMU) | virtio-net + tap, FC net thread |
| **MEM** memory | host page faults (native) | Sentry-managed faults + host mmap | EPT faults + Sentry | EPT violations, host page faults backing guest RAM, balloon | EPT violations, host faults; UFFD faults after snapshot restore |
| **TMR** timer / idle | host scheduler (native) | Sentry timers, futex waits | exits on HLT / timer | HLT exits, TSC-deadline MSR writes, halt-polling | same |
| **IRQ** interrupts / IPI | n/a | n/a | exits | external-interrupt exits, IPI exits, posted-interrupt support varies | same |
| **OTH** other | — | — | — | CPUID, I/O port, MSR misc | same |

**Handler locus (added 2026-11-09, `12` §5).** Every VM-exit class is further tagged `KERNEL-handled` (EPT violation, HLT, in-kernel APIC/timer, ioeventfd doorbell) or `USERSPACE-handled` (exit returned to the VMM or, for gVisor-KVM, to the `runsc` process). Reason: on HOST-A the kernel issues IBPB before the first return to userspace after a VM exit (VMScape mitigation), so `C_c` for USERSPACE-handled exits depends on the mitigation state, and KERNEL-handled exits do not. Reported as separate sub-columns, never merged.

Notes:
- "n/a (native)" means the event exists but is the **reference path**; its cost is counted in demand `D`, not in boundary cost.
- KVM exit-reason → class mapping must be written per vendor (Intel VMX exit reasons ≠ AMD SVM exit codes). **Verify** on the actual host CPU.
- Firecracker device notifications likely use KVM **ioeventfd** (exit handled in-kernel, VMM thread woken asynchronously). **Verify** in Firecracker source; it decides whether STO/NET handler time is "synchronous" or "async".

## 3. Cost model

```
T_boundary(r, w) = Σ_c  D_c(w) · A_c(r) · C_c(r)          [host-handler CPU time]

D_c(w)   demand of workload w in class c, measured on the reference
         (same kernel, native process, same FS on tmpfs). Runtime-independent.
         Units: syscalls (SYS), files+bytes (STO), packets+bytes (NET),
                page touches (MEM), wakeups (TMR).
A_c(r)   HSEs per unit of demand under runtime r      ("how often do I cross")
C_c(r)   handler CPU time per HSE                      ("how much does a crossing cost")

Headline metric (replaces O3's "host time per crossing"):
   η(r, w) = T_boundary(r, w) / Σ_c D_c(w)·w_c        host time per unit of workload demand
   (w_c = fixed class weights, or report η per class — prefer per class; see §6)

Residual (reported, never hidden):
   R(r, w) = T_total(r, w) − T_total(ref, w) − T_boundary(r, w)
```

Why this shape:
- **A** separates *design* (gVisor traps per call; VMs batch) from **C** *implementation quality* (how fast a given trap/exit is).
- A vendor can act on it: "your A_STO is 40× runc's because of X" is actionable; "you are 23% slower" is not.
- It is falsifiable: if `T_boundary` predicts end-to-end slowdown poorly (large or variable `R`), the model fails — see H-3 in `05-objectives-hypotheses.md`.

## 4. Per-runtime control flow (where probes go)

### 4.1 runc (reference + seccomp delta)
```
 app ──syscall──► [seccomp BPF] ──► host kernel handler ──► return
                    ▲ probe: raw_syscalls:sys_enter/sys_exit, filtered by cgroup id
 boundary cost = (runc − native) per syscall class  ≈ seccomp + overlayfs + cgroup accounting
```

### 4.2 gVisor systrap
```
 app thread ──syscall──► seccomp RET_TRAP ──► SIGSYS in stub ──► shared-mem/futex ──► Sentry
                                                                                     │
                     ┌───────────────────────────────────────────────────────────────┘
                     ├─ emulate in Sentry (SYS)                       probe: Sentry CPU time
                     ├─ host syscall (filtered by Sentry seccomp)     probe: sys_enter on Sentry tids
                     └─ gofer IPC (if not directfs)                   probe: gofer CPU time, IPC count
                     ▼
              resume app thread
 independent count: runsc's own trace points / --strace (validate SYS counts)   [verify flags]
```

### 4.3 gVisor KVM platform
```
 app (guest ring3) ──syscall──► Sentry (guest ring0) — no VM exit
                                 └─ Sentry needs host ──► "bluepill" switch ──► host syscall
                                                          probe: kvm:kvm_exit on Sentry vCPU threads
```

### 4.4 Kata (QEMU or Cloud Hypervisor) and Firecracker
```
 guest app ──syscall──► guest kernel (no exit)
                          │
                          ├─ virtio queue notify ──MMIO/PIO write──► KVM exit
                          │      ├─ ioeventfd: signal VMM/vhost thread, re-enter guest fast
                          │      └─ else: exit to VMM userspace (kvm_userspace_exit)
                          ├─ HLT / TSC-deadline MSR ──► KVM exit (TMR)
                          ├─ EPT violation ──► KVM exit + possible host page fault (MEM)
                          └─ IPI / external interrupt ──► exit (IRQ)
 probes (host): kvm:kvm_exit(reason), kvm:kvm_entry, kvm:kvm_userspace_exit,
                sched:sched_switch on VMM/vhost/virtiofsd tids, page-fault tracepoints
 independent:   perf kvm stat record/report
```

## 5. Attribution to a sandbox

```
 sandbox S ─► cgroup v2 path P_S
   threads in P_S:  vCPU threads, VMM main/IO threads (FC, QEMU, CLH)
                    Sentry + gofer (gVisor)
                    virtiofsd (Kata, if placed in sandbox cgroup)        [verify config]
                    app processes (runc)
   kernel threads NOT in P_S by default:
                    vhost workers — CORRECTED 2026-11-09 (docs/12 §2): on v5.15 the worker is a
                    kthread that is moved into the owner's cgroups via cgroup_attach_task_all()
                    at SET_OWNER [kernel source], so it IS attributable if the cgroup exists
                    BEFORE the VMM opens the vhost device [inference; test]. Since 6.4 it is a
                    vhost_task owned by the VMM process. Firecracker does not use vhost
                    [to verify in pinned source]; matters only for Kata-QEMU (optional row).
                    kworkers / softirq (network RX, block completion) → attribute by device ownership
```

Rules:
- Primary key = cgroup id from `bpf_get_current_cgroup_id()` at the handler.
- Kernel-thread work without a cgroup is reported as **"unattributed host"**, per class, never silently dropped or spread.
- Validation: with one sandbox, Σ attributed + unattributed must equal host CPU delta measured by `/proc/stat` within a stated tolerance.

## 6. Open design questions (to resolve in week 3)

1. **Aggregate η or per-class η?** A single number invites misuse; per-class is honest but harder to rank. Proposal: per-class is primary; a single η with published weights from the workload's own demand mix (so it is workload-specific, never universal).
2. **Where does Sentry emulation time go — boundary or "guest"?** Proposal: boundary (it is the isolation layer's work), but report it split so others can re-classify.
3. **Guest kernel time in VMs** — tenant-controlled, so not boundary; but it is isolation cost vs runc. It lands in residual `R`. Reported as "guest kernel tax" separately when guest-side probes are available.
4. **Synchronous vs async handler time** for ioeventfd paths — report both; O5 uses synchronous, O3 uses CPU.
5. **Clock correlation** guest↔host: KVM guest TSC = host TSC × scale + offset; on constant-TSC hosts without scaling, offset alone suffices (per-vCPU offset readable via KVM debugfs **[verify path for pinned kernel]**). gVisor has no guest clock problem.

## 7. What would falsify the abstraction

- `R` is large (> ~30% of slowdown, threshold to pre-register) and unstable across phases → crossings are not the main cost; the paper says so.
- Class mapping is ambiguous for a large share of events (e.g. > 10% of exits in OTH) → taxonomy is wrong.
- η ordering is no more stable across workloads than raw throughput ordering → O3 fails as stated.
