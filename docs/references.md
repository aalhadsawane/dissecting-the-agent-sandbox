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
