# 09 — Open questions for the owner (with recommendations)

**TL;DR**
- Q1 gVisor: run **systrap** (default) and **KVM** (bare-metal best case). Both, because they cross the boundary differently.
- Q2 CVM: **drop from core**. On AWS you are the CVM guest; the host side cannot be traced, so it cannot populate the unit of account.
- Q3 venue: **blog + arXiv first**, then workshop. Deadline TBD.
- Q4 overnight runs: needed for co-tenancy; answer decides O5 scope.
- Q5–Q9 are new questions this review raised.

---

### Q1. Which gVisor platform, and is one alternative in scope?
- **Recommendation:** `systrap` (default since 2023; what users run) **and** `kvm` (docs recommend it on bare metal). ptrace is deprecated — exclude.
- Why both: systrap = trap per syscall to a host process; KVM = Sentry in guest ring0, exits only on host calls. Same product, different HSE structure — a clean within-vendor test of the abstraction.
- Cost: one extra row.

### Q2. Confidential-VM comparison: wanted or distraction?
- **Recommendation:** distraction for the core paper.
- Reasons: (1) no host access on cloud CVMs → no HSE decomposition; (2) the home server would need SEV-SNP/TDX-capable CPU + firmware to do it properly — unknown; (3) arXiv:2609.04478 already covers CVM vs VM for serverless.
- Exception: if the home CPU supports SEV-SNP or TDX with host access, it becomes a strong extension (encrypted memory changes MEM-class costs). Ask after specs arrive.

### Q3. Workshop first or arXiv + blog first?
- **Recommendation:** blog (week 4) + arXiv (week 8), then workshop.
- APSys deadline: **TBD** — not verified in this session. Check before freezing the plan.

### Q4. Can the server run overnight sweeps?
- Affects O5 (needs many samples per neighbor configuration) and three-strikes reruns.
- If daytime only: O5 becomes exploratory; reduce neighbor types.

### Q5 (new). Server specs — see `06-infrastructure.md` §1. **Blocking.**

### Q6 (new). Is the reframing of O1 acceptable?
- Original: "sandbox > 50% of wall-clock with stubbed model" — cannot fail.
- Proposed: crossover curve `f(L)` and `L*` (`05` H-1). Prior work (2605.26297) suggests LLM time dominates for real models; the curve says when that stops being true.

### Q7 (new). Is replacing "host time per crossing" with "host time per unit workload demand" acceptable?
- See `02-critique.md` §2. Per-crossing will still be reported, to show why it misleads.

### Q8 (new). AWS account type?
- Academy Learner Lab vs regular account with credits decides whether metal is possible at all.

### Q9 (new). Citations to resolve
- Source of "Does Virtualization Really Incur a 5% Overhead?" — not found under that title.
- Exact entries for "Zhang et al. I/O virtualization (2010)" and "Hofstra multi-threaded overhead diagnosis".
- Where the "178k production sessions" figure for AgentSysBench comes from.
