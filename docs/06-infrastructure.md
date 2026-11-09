# 06 — Infrastructure: what each asset is for

**TL;DR**
- **Home bare-metal server** = the only measurement platform. Unknown specs are the first blocker.
- **GitHub Actions** = CI for analysis and docs. Runners are VMs → never produce numbers.
- **Claude cloud container** = planning, writing, analysis. Verified to be a Firecracker guest without `/dev/kvm` → never produce numbers.
- **AWS** = dropped (no credits). External validity now rests on `12` §6b options. CVM on AWS cannot be host-traced (you are the guest).
- **Domain + blog** = two posts before the paper.

---

## 1. Home server (primary) — ANSWERED 2026-11-09; analysis in `12-host-platform-analysis.md`

Facts (owner-reported, `docs/evidence/host-pack-2026-11-09.md`): bare metal, Intel i7-12700 (hybrid 8P+4E, SMT on, VMX/EPT, invariant TSC), 128 GiB non-ECC DDR5-4800, Ubuntu 22.04.5, kernel 5.15.0-190 (reboot already pending), turbo on / `powersave`, C-states limited to C1/C2 (CFG-locked), CPU 2 isolated at boot, **shared with 33 sessions from 5 other accounts**, no BMC/IPMI, no TDX/SEV/IOMMU.

Consequences in short: not exclusive → noise baseline + acceptance gates; hybrid → P-core pinning and designed neighbor placement; off-matrix kernel for Firecracker; VMScape IBPB mitigation active; KVM vPMU disabled on hybrid; CVM row closed; BIOS knobs unreachable (runtime knobs only, need consent).

Original request list (kept for the record; all answered except the follow-ups in `12` §9):

| Item | Why it matters |
|---|---|
| CPU model, vendor (Intel/AMD), cores/threads, sockets | exit-reason mapping; isolcpus plan; NUMA |
| `vmx`/`svm`, `ept`/`npt`, `constant_tsc`, `nonstop_tsc`, posted interrupts / APICv / AVIC | exit costs and clock correlation |
| RAM size | tmpfs budget + co-tenant density for O5 |
| BIOS access (C-states, turbo, SMT) | pinning (invariant 3) |
| Distro + ability to install a custom/pinned kernel | eBPF features; (vhost_task ≥ 6.4 claim withdrawn, see `12` §2) |
| Noise: other services, shared use, thermal behavior | overnight sweeps (Q4) |
| Remote access method | runs while owner is away |

RAM budget sketch (to be replaced by measured numbers):
```
 OS + harness + probes                          a few GB
 tmpfs: guest kernels + rootfs images           ~ per-runtime image × 5 rows
 tmpfs: workload repos + local mirrors          depends on chosen repos (measure)
 sandboxes: vCPU/RAM per sandbox × N co-tenants  N bounded by RAM, not CPU
```
If RAM is small (e.g. ≤ 32 GB), O5's N range shrinks; say so rather than swap to HDD.

## 2. GitHub Actions (public repo, unlimited minutes)

Use for:
- Regenerating `derived/` and every paper/blog table from committed `raw/` on each push (DoD: traceability).
- Lint/format checks on analysis code and docs; link checking of citations.
- Building guest kernels / rootfs images reproducibly (build artifacts are not measurements; checksums published).

Never for: any timing number. Hosted runners are VMs; nested virtualization breaks invariant 1.

## 3. Claude cloud container (this session)

Evidence it is virtualized (captured 2026-11-08):
```
systemd-detect-virt   → docker
/proc/cpuinfo         → "hypervisor" flag, Intel Xeon @ 2.10GHz, 4 vCPU, 15 GiB
/dev/kvm              → absent
dmesg                 → "Hypervisor detected: KVM", kvm-clock, cmdline "--firecracker-init"
```
Use for: literature work, design docs, analysis code, data plots. Not for measurement.
Side note for the blog: the tool used to plan this study itself runs in a Firecracker microVM — a concrete example of the deployment pattern under study.

## 4. AWS (dropped 2026-11-09)

Owner has an AWS student/builder account **with no credits** (D-31). No paid bare-metal. Consequences: no independent second host from AWS, no AWS-based H-M, no kernel cross-check on AWS. Replacement options for a second data point (CloudLab/Chameleon via a faculty sponsor, another lab machine, P-core vs E-core on HOST-A, or an explicit single-host claim) are in `12` §6b.

Confidential VMs: closed independently — HOST-A has no TDX/SEV (D-21), and on cloud CVMs the experimenter is the guest and cannot trace the host side.

## 5. Domain + blog

- Post 1 (≈ week 4): method + folk-number audit (H-F). Includes the unit-of-account diagram and probe-overhead table.
- Post 2 (≈ week 8): phase decomposition + co-tenancy results, with links to raw data.
- Every figure generated from `derived/`; every post links to the commit that produced it.
