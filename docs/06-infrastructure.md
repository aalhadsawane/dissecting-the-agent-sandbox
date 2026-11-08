# 06 — Infrastructure: what each asset is for

**TL;DR**
- **Home bare-metal server** = the only measurement platform. Unknown specs are the first blocker.
- **GitHub Actions** = CI for analysis and docs. Runners are VMs → never produce numbers.
- **Claude cloud container** = planning, writing, analysis. Verified to be a Firecracker guest without `/dev/kvm` → never produce numbers.
- **AWS credits** = one short bare-metal portability check *only if* the account can launch `.metal`; otherwise artifact hosting. CVM on AWS cannot be host-traced (you are the guest).
- **Domain + blog** = two posts before the paper.

---

## 1. Home server (primary)

Needed from owner before week 1 (blocking):

| Item | Why it matters |
|---|---|
| CPU model, vendor (Intel/AMD), cores/threads, sockets | exit-reason mapping; isolcpus plan; NUMA |
| `vmx`/`svm`, `ept`/`npt`, `constant_tsc`, `nonstop_tsc`, posted interrupts / APICv / AVIC | exit costs and clock correlation |
| RAM size | tmpfs budget + co-tenant density for O5 |
| BIOS access (C-states, turbo, SMT) | pinning (invariant 3) |
| Distro + ability to install a custom/pinned kernel | eBPF features, vhost_task attribution (≥ 6.4) |
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

## 4. AWS student credits

Facts to establish first:
- Account type: AWS Academy Learner Lab accounts restrict instance types and sizes (course docs list caps such as nano–large and ≤ 32 vCPU in some versions); `.metal` is very likely blocked. Credit sizes reported by universities: $50–$100. **Verify the owner's actual account.**
- If a normal account with credits: third-party price trackers list `m5zn.metal` ≈ $3.96/h on-demand (us-east-1) and `c6g.metal` (ARM Graviton2) cheaper. **Verify on the AWS pricing page at time of use.**

Recommended use, in priority order:
1. **x86 bare-metal portability check** (same CPU vendor as home server if possible): run the calibration set + one phase sweep for all rows. Budget ≈ a few hours; at ~$4/h that is tens of dollars. Spot only if the run is restartable.
2. Do **not** use ARM metal as a "validation" of x86 numbers — it is a different ISA and a different experiment.
3. If metal is unavailable: artifact/dataset hosting (S3) and state plainly that portability used one host.

Confidential VM: on AWS, SEV-SNP is offered on regular (non-metal) instances **[verify families]**. The experimenter is the CVM **guest**; host-side HSE probes are impossible. A CVM row could only be black-box (end-to-end + guest-visible counters), which does not populate the unit of account and largely repeats arXiv:2609.04478. Recommendation in `09-open-questions.md` Q2: drop from core.

## 5. Domain + blog

- Post 1 (≈ week 4): method + folk-number audit (H-F). Includes the unit-of-account diagram and probe-overhead table.
- Post 2 (≈ week 8): phase decomposition + co-tenancy results, with links to raw data.
- Every figure generated from `derived/`; every post links to the commit that produced it.
