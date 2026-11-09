# References (with links and verification status)

V = primary page read this session · S = secondary/snippet only · U = not yet located/read.

## Agent workloads
- [V] Chang et al., *From LLM Inference to Agentic Workloads: Characterization and Implications for Serving Systems* (AgentSysBench), arXiv:2608.15127, 2026. https://arxiv.org/abs/2608.15127 · HTML: https://arxiv.org/html/2608.15127
- [S] Yuan, Nayak, Kundu, Talati, *Agentic AI Workload Characteristics*, arXiv:2605.26297, 2026. https://arxiv.org/abs/2605.26297 · review: https://www.themoonlight.io/en/review/agentic-ai-workload-characteristics
- [S] NVIDIA TensorRT-LLM blog, *Evaluating Agentic Serving with Trace Replay and Job-Level Metrics*. https://nvidia.github.io/TensorRT-LLM/blogs/tech_blog/blog27_Evaluating_Agentic_Serving_with_Trace_Replay_and_Job_Level_Metrics.html
- [U] MLSys 2026 agent-workload papers (titles TBD).

## Isolation cost
- [known] Manco et al., *My VM is Lighter (and Safer) than your Container*, SOSP 2017.
- [known] Agache et al., *Firecracker: Lightweight Virtualization for Serverless Applications*, NSDI 2020.
- [V abstract / S numbers] Young et al., *The True Cost of Containing: A gVisor Case Study*, HotCloud 2019. https://www.usenix.org/conference/hotcloud19/presentation/young
- [V abstract] Niroula, Shan, Ding, *Performance Study of Serverless Workloads in Confidential Virtual Machines*, arXiv:2609.04478, 2026. https://arxiv.org/abs/2609.04478
- [V abstract] Ustiugov et al., *Benchmarking, Analysis, and Optimization of Serverless Function Snapshots* (vHive, REAP), ASPLOS 2021. https://arxiv.org/abs/2101.09355
- [U] "Does Virtualization Really Incur a 5% Overhead?" — not found under this title. Candidate: ICPP 2015 virtualization overhead study, https://www.cs.virginia.edu/~hs6ms/publishedPaper/Conference/2015/ICPP-VirtualOverhead.pdf (unconfirmed match).
- [U] Zhang et al., I/O virtualization overhead, 2010.
- [U] Hofstra, multi-threaded virtualization overhead diagnosis.

## Security-side comparisons (scoop watch)
- [S] Andronchik, Lokhmakov, *AI Code Sandboxes: A Comparative Security Study. Part 1 of 2*, arXiv:2606.08433, 2026. https://arxiv.org/abs/2606.08433
- [S] SandboxEscapeBench, arXiv:2603.02277.
- [S] *Sandlock: Confining AI Agent Code with Unprivileged Linux Primitives*, arXiv:2605.26298. https://arxiv.org/pdf/2605.26298

## Tracing
- [S] DORSAL lab progress reports on hypertracing (Benbachir, 2016–2017): https://hsdm.dorsal.polymtl.ca/system/files/abder%20-%20progress%20report%20meeting%20dec2017_0.pdf
- [S] Bastien, DORSAL progress meeting May 2018 (host/guest trace correlation): https://amdls.dorsal.polymtl.ca/files/ProgressMeetingMay2018_gbastien.pdf
- [S] Linux Plumbers 2017 proposal 4769 (ftrace across virtualization layers): https://www.linuxplumbersconf.org/2017/ocw/sessions/4769
- [S] lttng-dev RFC 2018 on host/guest trace linking: https://lists.lttng.org/pipermail/lttng-dev/2018-January/027704.html
- [S] LKML 2023, KVM selftest: eBPF program to collect VMX exit stats: https://lkml2.uits.iu.edu/hypermail/linux/kernel/2301.3/02667.html
- [S] Craun et al., *Eliminating eBPF Tracing Overhead on Untraced Processes*, eBPF '24, DOI 10.1145/3672197.3673431. https://people.cs.vt.edu/djwillia/papers/ebpf24-mookernel.pdf
- Tools: `perf kvm stat`, LTTng, bpftrace, libbpf.

## gVisor documentation
- [V] Platform guide: https://gvisor.dev/docs/architecture_guide/platforms
- [V] Changing platforms: https://gvisor.dev/docs/user_guide/platforms/
- [S] Systrap release blog (2023): https://gvisor.dev/blog/2023/04/28/systrap-release/
- [S] Rootfs overlay (2023): https://opensource.googleblog.com/2023/04/gvisor-improves-performance-with-root-filesystem-overlay.html

## Vendor / industry (folk numbers; audit targets, not evidence)
- [S] Google Cloud blog, Agent Sandbox on GKE: https://cloud.google.com/blog/products/containers-kubernetes/bringing-you-agent-sandbox-on-gke-and-agent-substrate
- [S] InfoQ, GKE Agent Sandbox / Hypercluster (2026-05): https://infoq.com/news/2026/05/gke-agent-sandbox-hypercluster
- [S] Northflank, Cloud Hypervisor vs gVisor: https://northflank.com/blog/cloud-hypervisor-vs-gvisor
- [S] gVisor vs Kata vs Firecracker (bex.co, 2026-09): https://bex.co/blog/2026/09/22/gvisor-vs-kata-vs-firecracker-agent-sandbox-isolation
- [S] "Benchmark: gVisor 1.0 vs Kata 3.0 vs Firecracker 1.5": https://johal.in/benchmark-gvisor-10-vs-kata-containers-30-vs/
- [S] AI agent sandbox comparison 2026 (grigio.org): https://grigio.org/ai-agent-sandbox-technologies-a-complete-2026-comparison/
- [S] Secure Agent Sandboxes on EKS: https://builder.aws.com/content/3ADDWTtyI2gevtzY9d2vzULAxzS/secure-agent-sandboxes-on-eks

## Benchmark suites
- SeBS (Copik et al., Middleware 2021); vHive (Ustiugov et al., ASPLOS 2021); ServerlessBench (Yu et al., SoCC 2020); FaaSdom (Maissen et al., DEBS 2020).

## AWS facts (verify at time of use)
- [S] m5zn.metal pricing (third-party): https://calculator.holori.com/aws/ec2/m5zn.metal/us-east-1
- [S] c6g.metal pricing (third-party): https://calculator.holori.com/aws/ec2/c6g.metal/us-east-1
- [S] AWS Academy Learner Lab restrictions (course copy): https://cyberlab.pacific.edu/courses/comp175/resources/aws-academy/AWS_Academy_Learner_Lab-Foundational_Services.pdf

## Round 2 (2026-11-08): stateful sandboxes and FS path
Grades: P peer-reviewed · A arXiv · V vendor · C community. "read" = primary text read this session.
- [P, read] Anjali, Caraza-Harter, Swift, *Blending Containers and Virtual Machines: A Study of Firecracker and gVisor*, VEE 2020, DOI 10.1145/3381052.3381315. https://par.nsf.gov/servlets/purl/10176031
- [P, read] van Rijn, Rellermeyer, *A Fresh Look at the Architecture and Performance of Contemporary Isolation Platforms*, Middleware 2021. https://arxiv.org/abs/2110.11462
- [P, read] Lee, Kim, Kwon, Park, Hong, *Impact of Secure Container Runtimes on File I/O Performance in Edge Computing*, Applied Sciences 13(24):13329, 2023. https://doi.org/10.3390/app132413329
- [P, abstract] Jang, Suh, Tak, *Understanding File System Operations of a Secure Container Runtime Using System Call Tracing Technique*, IEICE Trans. Inf. & Syst. E107-D(2):229–233, 2024. DOI 10.1587/transinf.2023EDL8039
- [P, S] Harter et al., *Slacker: Fast Distribution with Lazy Docker Containers*, FAST 2016. https://research.cs.wisc.edu/wind/Publications/slacker-fast16.pdf
- [P, S] Oakes et al., *SOCK: Rapid Task Provisioning with Serverless-Optimized Containers*, ATC 2018. https://www.usenix.org/conference/atc18/presentation/oakes
- [P, S] Li et al., *RunD*, ATC 2022. https://www.usenix.org/conference/atc22/presentation/li-zijun-rund
- [P, S] Ao, Porter, Voelker, *FaaSnap*, EuroSys 2022. https://www.sysnet.ucsd.edu/~voelker/pubs/faasnap-eurosys22.pdf
- [A, read] Zheng et al., *AgentCgroup: Understanding and Controlling OS Resources of AI Agents*, arXiv:2602.09345. https://arxiv.org/html/2602.09345v2
- [A, read] Wu, Chang, Cao, Gao, Wang, *Crab: A Semantics-Aware Checkpoint/Restore Runtime for Agent Sandboxes*, arXiv:2604.28138. https://arxiv.org/html/2604.28138v1
- [A, read] Dong et al., *DeltaBox: Scaling Stateful AI Agents with Millisecond-Level Sandbox Checkpoint/Rollback*, arXiv:2605.22781. https://arxiv.org/html/2605.22781v1
- [A, read] Xu, Zhou, Wu, Kaffes, *Toward Systems Foundations for Agentic Exploration*, arXiv:2510.05556. https://arxiv.org/abs/2510.05556
- [A, read partial] Huang et al., *DeepSeek Elastic Compute (DSec)*, arXiv:2609.22978. https://arxiv.org/html/2609.22978
- [A, read] *SWE-MiniSandbox: Container-Free Reinforcement Learning*, arXiv:2602.11210. https://arxiv.org/html/2602.11210v3
- [A, S] Shepherd arXiv:2605.10913; TVCACHE arXiv:2602.10986; RepoForge arXiv:2508.01550; CodeGrep arXiv:2608.05886.
- [V, read] Modal, *VM sandbox filesystem* (2026-10-16). https://modal.com/blog/vm-sandbox-filesystem
- [V, read] Tensorlake, *Firecracker disk snapshots, O(changed bytes)* (2026-07-13). https://tensorlake.ai/blog/firecracker-disk-snapshots-o-changed-bytes
- [V, read] *Scaling Firecracker using OverlayFS to save disk space* (2025-02-12), on E2B blog. https://www.e2b.dev/blog/scaling-firecracker-using-overlayfs-to-save-disk-space
- [V, read] gVisor rootfs overlay (2023). https://opensource.googleblog.com/2023/04/gvisor-improves-performance-with-root-filesystem-overlay.html
- [V, read] gVisor directfs (2023-06-27). https://gvisor.dev/blog/2023/06/27/directfs/
- [V] gVisor filesystem guide (overlay2 default root:self). https://gvisor.dev/docs/user_guide/filesystem/
- [V, read] Epoch AI, *How to run SWE-bench Verified in one hour on one machine*. https://epoch.ai/latest/swebench-docker
- [V] E2B persistence docs. https://e2b.dev/docs/sandbox/persistence
- [V] Daytona persistence docs. https://www.daytona.io/docs/en/persistence
- [V] Vercel Sandbox snapshots/persistence. https://vercel.com/docs/sandbox/concepts/snapshots
- [V] Modal sandbox snapshots. https://modal.com/docs/guide/sandbox-snapshots
- [V] Fly Sprites checkpoint/restore. https://fly.io/sprites/checkpoint-restore.md
- [V] Cloudflare Sandbox backup/restore. https://developers.cloudflare.com/sandbox/guides/backup-restore
- [V] GKE Agent Sandbox snapshots. https://agent-sandbox.sigs.k8s.io/docs/sandbox/snapshots/ · InfoQ on Pod Snapshots: https://infoq.com/news/2026/09/gke-pod-snapshots-benchmarks
- [V] OpenAI Codex cloud environments. https://developers.openai.com/codex/cloud/environments
- [V] CodeSandbox, *Cloning microVMs using userfaultfd*. https://codesandbox.io/blog/cloning-microvms-using-userfaultfd
- [V] Modal memory snapshots (gVisor). https://modal.com/blog/mem-snapshots
- [C] Firecracker block caching doc (mirror): https://cocalc.com/github/aos/firecracker/blob/main/docs/api_requests/block-caching.md — verify upstream
- [C] LWN, virtio-fs RFC with smallfile numbers. https://lwn.net/Articles/774495/
- [C] Kata runtime issue #2138 (virtio-fs fio). https://github.com/kata-containers/runtime/issues/2138
- [V] Man Group, pip performance analysis. https://www.man.com/technology/improving-python
- [C, blocked] virtio-fs perf numbers thread (listman.redhat.com not allowed in session; lore mirror 403). https://listman.redhat.com/archives/virtio-fs/2020-December/002384.html

## Round 3 (2026-11-09): host platform
- [V, read] Firecracker kernel support policy (host 5.10 / 6.1 / 6.18). https://github.com/firecracker-microvm/firecracker/blob/main/docs/kernel-policy.md
- [V, read] Firecracker production host setup (SMT off, swap off, KSM off recommendations; no CPU-model list). https://github.com/firecracker-microvm/firecracker/blob/main/docs/prod-host-setup.md
- [K, read] Linux kernel docs, VMSCAPE (affected CPUs, IBPB before first exit to userspace, `vmscape=`). https://docs.kernel.org/admin-guide/hw-vuln/vmscape.html
- [C, snippet] Phoronix, Intel patches to reduce VMSCAPE mitigation cost on Alder Lake+. https://www.phoronix.com/news/Intel-Less-Cost-VMSCAPE
- [K, read] Linux cgroup v2 docs, cpuset `cpuset.cpus.partition` (root/isolated/member). https://docs.kernel.org/admin-guide/cgroup-v2.html
- [K, read] Linux v5.15 `drivers/vhost/vhost.c` (kthread worker, `cgroup_attach_task_all`). https://raw.githubusercontent.com/torvalds/linux/v5.15/drivers/vhost/vhost.c
- [C, snippet] KVM: disable vPMU on Intel hybrid hosts, lkml 2023. https://lkml.iu.edu/hypermail/linux/kernel/2302.0/05143.html ; KVM Forum 2024 talk (hybrid still unsupported). https://pretalx.com/kvm-forum-2024/talk/NYWTMH
- [P, read] Maricq, Duplyakin, Jimenez, Maltzahn, Stutsman, Ricci, *On the Variability of Performance... / Taming Performance Variability*, OSDI 2018. https://www.usenix.org/system/files/osdi18-maricq.pdf
- [C] AWS bare-metal instance prices (third-party, unverified): https://calculator.holori.com/aws/ec2/c5.metal/us-east-1 , https://cloudprice.net/aws/ec2/instances/c6i.metal , https://www.doit.com/compute/spot/us-east-1/c7i.xlarge
- [owner] Host evidence pack (sanitized): docs/evidence/host-pack-2026-11-09.md
