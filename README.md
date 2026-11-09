# Dissecting the agent sandbox

Where does an agent sandbox's time actually go — at the isolation boundary, per sandbox, per workload phase — across runc, gVisor, Kata and Firecracker, measured comparably on bare metal?

**Status:** planning / research phase (started 2026-11-08). No code yet, on purpose.

## TL;DR of the plan
- **Gap is real:** the leading agent-workload study (AgentSysBench, arXiv:2608.15127) uses Docker-only sandboxes measured with cAdvisor/Prometheus. No one has decomposed sandbox cost across isolation mechanisms, per sandbox, under agent workloads.
- **The thesis needs care:** a second 2026 study (arXiv:2605.26297) finds LLM inference is ~71–98% of execution time for Claude-Code-driven benchmarks. "Sandbox is the cost center" holds for *dollars* (idle billing) far more than for *time*. O1 is reframed as a crossover curve.
- **Unit of account:** the *host-serviced event* (HSE), cost = demand × amplification × unit cost, plus a reported residual for overhead that never crosses the boundary (2-D page walks, guest-kernel work).
- **First result (week 4):** a folk-number audit — re-measure published gVisor/Firecracker/vendor numbers with pinned modern versions and explain the gaps.
- **Round 2 (stateful sandboxes):** agent sandboxes are long-lived stateful sessions, not FaaS. Vendors converge on block-device FS for VM sandboxes. State ops (checkpoint/fork) and durability semantics are new first-class variables. "FS dominates clone/install" is withdrawn: graded evidence in [`docs/11-stateful-fs-path.md`](docs/11-stateful-fs-path.md).
- **Round 3 (host pack):** bare metal confirmed, but shared (33 sessions), hybrid, turbo-on, kernel 5.15 off Firecracker's validated host kernels, VMScape IBPB mitigation active, KVM vPMU off. Earlier "vhost needs ≥ 6.4" claim withdrawn. Noise is measured and gated, not assumed ([`docs/12-host-platform-analysis.md`](docs/12-host-platform-analysis.md)).
- **Round 4:** no AWS credits → AWS dropped; second data point must come from a lab machine, a faculty-sponsored CloudLab project, or P-core vs E-core on the same host (H-X). virtio-fs thread read via mirror: results vary ~5–8× with cache mode, so every virtio-fs row records its knobs.
- **Blocking:** home server specs and the answers in [`docs/09-open-questions.md`](docs/09-open-questions.md).

## Documents (read in order)
| # | File | What it holds |
|---|------|---------------|
| 00 | [brief](docs/00-brief.md) | Original brief, preserved |
| 01 | [literature](docs/01-literature.md) | Prior work, what each establishes, verification status, reading queue |
| 02 | [critique](docs/02-critique.md) | Misconceptions and confounds in the brief, and fixes |
| 03 | [unit of account](docs/03-unit-of-account.md) | HSE definition, taxonomy, D×A×C model, per-runtime control flow, attribution |
| 04 | [methodology](docs/04-methodology.md) | Platform discipline, runtime matrix, workloads, probes, validation, statistics |
| 05 | [hypotheses](docs/05-objectives-hypotheses.md) | O1–O6 as pre-registered hypotheses with thresholds and priors |
| 06 | [infrastructure](docs/06-infrastructure.md) | What each asset is for (and not for) |
| 07 | [plan](docs/07-plan.md) | 8-week schedule, gates, scope ladder |
| 08 | [risks](docs/08-risks.md) | Risk register |
| 09 | [open questions](docs/09-open-questions.md) | For the owner, with recommendations |
| 10 | [decision log](docs/10-decision-log.md) | Decisions and their evidence |
| 11 | [stateful FS path](docs/11-stateful-fs-path.md) | Round 2: stateful sandboxes, FS data path evidence (graded), state ops, durability |
| 12 | [host platform analysis](docs/12-host-platform-analysis.md) | Round 3: the real hardware vs earlier assumptions (noise, kernel, hybrid CPU, VMScape, governance) |
| — | evidence | [host pack, sanitized](docs/evidence/host-pack-2026-11-09.md) |
| — | [references](docs/references.md) | Links + verification status |

## Invariants (from the brief, non-negotiable)
Bare metal only · never fabricate or interpolate (`TBD` + issue) · pin and publish the environment · quantify the observer · one variable at a time, median + CI · validate probes against an independent tool · three strikes for surprising results.
