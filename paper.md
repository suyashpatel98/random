# Research proposal: Does the Sandbox Change the Reward?

**Working title:** *Does the Sandbox Change the Reward? Differential Testing of Agent Environments Across Isolation Backends*

**Target quality:** an EuroMLSys / MLSys-workshop paper (6 pages plus an appendix), which can be extended to a full
10-page version. arXiv preprint plus a public artifact. The bar is "a rigorous second-year PhD student".

**Status:** proposal. Novelty last checked 2026-11-21. Re-check before writing (see §12).

---

## 1. One-paragraph pitch

Agentic RL and agent evaluation both run untrusted tasks in sandboxes. Operators pick an isolation backend
(runc containers, gVisor, Kata/Firecracker microVMs) for security, density, and cost, and usually assume the
choice only affects *performance*. We test whether it also changes *outcomes*: does the reference solution
still pass, does doing nothing still fail, and do agent pass rates move? We run real Terminal-Bench 2.0 and SWE-bench
tasks across backends, resource-enforcement modes, and timeouts. We measure reward flips against a measured
noise floor and attribute each flip to a concrete kernel- or runtime-level cause. We then quantify what these
flips do to RL training signal and evaluate a cheap mitigation. Most experiments need **zero model tokens and no GPU**.

## 2. Draft abstract (to be revised as results come in)

> Sandboxed execution is the substrate of agentic RL and agent evaluation, and operators routinely swap
> isolation backends for security or density. We ask whether doing so silently changes rewards. Using
> N tasks from Terminal-Bench 2.0 and SWE-bench, we run reference ("oracle") and null solutions under runc, gVisor, and Kata
> across resource-enforcement and timeout configurations, with repeated runs and an A/A noise floor.
> We find [X]% of tasks change verdict under at least one backend, [Y]% of them *silently* (the verifier
> returns a wrong reward rather than an infrastructure error). We attribute flips to [K] root-cause classes,
> including resource-limit semantics, kernel-interface differences, and slowdown-induced timeouts. Re-verifying
> [M] public agent patches shows grading changes of [Z] points. A simple model shows such flips
> corrupt [W]% of GRPO groups. A conformance pre-flight check catches [V]% of flips at [cost]. We release
> the harness and a labeled flip dataset.

---

## 3. Motivation

- **RL depends on correct rewards.** In GRPO-style training, a silent reward flip changes the sign or size of
  advantages. A task whose oracle fails on the training backend teaches the model the wrong thing, and nothing
  in the training loop reports it.
- **Backends get swapped for non-ML reasons**: security reviews (runc → gVisor), density and cost (→ gVisor), or
  stronger isolation for untrusted code (→ microVMs). Eval leaderboards also run on different providers than
  the labs do.
- **Scattered evidence that this matters already exists, but nothing systematic** (§4):
  - Tencent saw Terminal-Bench 2 scores differ between runc and gVisor.
  - Amazon's Sandman saw 15/500 SWE-bench reference patches change verdict across execution conditions.
  - Anthropic showed resource enforcement moves scores by up to about 6 points.
  - A hobby benchmark observed that gVisor kills the *whole sandbox* on OOM or PID-limit violations, where
    runc kills one process.

## 4. Related work and the gap

| Work | What it covers | What it does not cover |
|---|---|---|
| Anthropic, *Quantifying infrastructure noise in agentic coding evals* (Feb 2026) | Resource enforcement levels (1× to uncapped) on TB 2.0 and SWE-bench (GKE). Infra error rate 5.8%→0.5%. Up to +6 pts | Runtime/isolation backend (not even named), timeouts, root causes beyond OOM, other models |
| Tencent / gVisor blog, *Scaling Agentic-RL Sandboxes to the Millions with gVisor* (Apr 2026) | runc vs gVisor score gap on TB2 (100% vs 94.38% → 97.75% after a fix). One root cause (tmux CPU spin) | Systematic study, microVMs, resource×backend interaction, taxonomy, RL impact |
| Sandman (Amazon, arXiv 2610.39711) | Production sandbox service for agentic RL. 15/500 SWE-bench reference patches flip across *execution conditions* (11 due to network denial) | Uses runc only. Explicitly did not evaluate gVisor or microVMs |
| *The Rollout Infrastructure Tax* (arXiv 2607.01415) | Cold-start/latency across 4 substrates for coding-agent RL | Rewards/correctness. Runtimes not named |
| BunshinRL (arXiv 2611.10453) | Workspace forking for RL rollouts (vs CRIU, Firecracker-based AgentENV) | Correctness across backends |
| *Stop Comparing LLM Agents Without Disclosing the Harness* (arXiv 2605.23950) | Harness variance > model variance on SWE-bench Verified. Proposes a disclosure card with an "Execution" layer | Execution layer not measured. Timeouts held fixed |
| DeltaSelect (arXiv 2609.19607) | Run-to-run noise, cheap A/B for coding agents | Infrastructure factors (explicitly out of scope) |
| OpenAI, *Separating signal from noise in coding evaluations* (Jul 2026) | About 30% of SWE-Bench Pro tasks broken (task-quality audit) | Infrastructure-induced flips |
| sandboxbench (GitHub, hobby) | runc vs gVisor perf on synthetic agent-shaped workloads (+38% to +155%). OOM/PID semantics differ | Real tasks, rewards, microVMs |

**Gap:** no study systematically varies the **isolation backend** (together with resource enforcement and
timeouts) on **real agent benchmarks**, measures **reward flips against a noise floor**, separates **loud**
(infra error) from **silent** (wrong verdict) flips, **attributes** them to kernel/runtime mechanisms, and
connects them to **RL training signal**.

**Positioning:** we systematize and extend scattered industry observations into a controlled, reproducible,
mechanism-level study with an open artifact. It is a measurement paper. Novelty comes from the experimental design,
the taxonomy, and the RL framing, not from a new system.

---

## 5. Research questions and hypotheses

**RQ1 (Prevalence).** How often does a task's *ground-truth* verdict change across isolation backends,
resource-enforcement modes, and timeouts?
- Ground truth means the oracle solution should pass and the null solution should fail.
- *H1a:* Oracle flip rate under gVisor and Kata exceeds the runc A/A noise floor.
- *H1b:* Backend and resource enforcement interact. Strict 1× limits produce more flips under gVisor,
  plausibly because the Sentry's own memory and threads count against the limits.
- *H1c:* Some flips are **silent**: the sandbox completes normally and the verifier returns a wrong reward.

**RQ2 (Mechanism).** What causes each flip? Build a root-cause taxonomy with kernel/runtime-level evidence.
- *H2:* Most flips fall into a small number of classes (candidate classes in §7.3), led by
  resource-limit semantics and slowdown-induced timeouts.

**RQ3 (Effect on agent scores).** Does the backend change measured agent performance beyond run-to-run noise?
- **RQ3a, grading only (zero tokens):** re-verify public agent patches (SWE-bench) across backends. This
  separates "the backend changes grading" from "the backend changes agent behavior".
- **RQ3b, end to end (small token budget):** live agent runs on a TB2 subset across backends.
- *H3:* Grading-only shifts are non-zero but small. End-to-end shifts are larger on syscall- and
  build-heavy tasks.

**RQ4 (RL impact and mitigation).** What fraction of GRPO groups would receive corrupted advantages? How
much of this does a cheap conformance pre-flight catch, and at what cost?
- *H4:* The pre-flight (oracle + null on the target backend, n runs) catches most deterministic flips at a
  small fraction of rollout cost. Stochastic flips need a timeout-margin rule.

**Falsification:** if flip rates under all backends are statistically indistinguishable from the A/A floor
once resources are enforced identically, the paper becomes a well-powered **negative result** ("the backend
does not matter, resource enforcement does"), backed by the taxonomy of the few residual cases. That is still
publishable at workshop level.

---

## 6. Experimental design

### 6.1 Factors

| Factor | Levels (primary) | Stretch levels |
|---|---|---|
| **Backend** B | `runc` (baseline), `runsc` (gVisor, systrap platform), `kata` (Kata Containers, Cloud Hypervisor or QEMU) | `runsc` KVM platform, `kata` + Firecracker |
| **Resource enforcement** R | (a) task spec as hard limit (1×), (b) spec guaranteed with a 3× ceiling, (c) uncapped. These mirror Anthropic's levels for comparability | 2× |
| **Timeout** T | task default, 2× default | 0.5× (stress) |
| **Network** N | allowed (default) | denied outbound (replicates Sandman's dominant cause) |
| **Host load** L | fixed concurrency c (e.g., 8) | c=1 vs c=16 |

Run the full B×R grid with T=default first. Run T and N as targeted follow-ups on tasks near the margins
(see the timeout-margin model, §7.4).

### 6.2 Tasks

- **Terminal-Bench 2.0** via Harbor: all tasks (about 89, verify). Tasks with a docker-compose environment are
  reported separately.
- **SWE-bench Verified subset:** about 100 instances, stratified by repository. Pick instances whose images
  are available and whose reference patch passes under runc (filtering pass, reported).
- Optional: SWE-bench Pro subset, cross-referenced with OpenAI's broken-task audit, to separate task brokenness from
  infrastructure effects.

### 6.3 Runs per task-configuration ("cell")

- Agents: `oracle` (reference solution) and `nop` (does nothing), both built into Harbor.
- **Two-phase adaptive design:**
  1. *Screening:* n=3 runs per cell.
  2. *Confirmation:* any cell discordant with its runc counterpart (or with itself) gets n=10 total.
- **A/A noise floor:** the full runc grid is repeated on a different day, after a reboot, with caches cleared.
  The discordance rate between the two runc runs is the floor that every claim must exceed.
- Rough volume: (89 + 100) tasks × 3 backends × 3 R × 2 agents × 3 runs ≈ 10k sandbox runs, plus confirmations.
  At a few minutes per run and concurrency 8–12, that is on the order of 2–4 days of machine time. Measure in the pilot.

### 6.4 Outcomes recorded per run

- Verdict (reward), verifier output, exit codes, and **outcome class**:
  - `pass`
  - `fail`: verifier ran and returned 0
  - `infra_error`: sandbox died, OOM kill, timeout before the verifier ran, or setup failed
- Wall-clock per phase (setup, solution, verify), peak memory, CPU time, and cgroup events
  (`memory.events`, `pids.events`).
- Backend logs: runsc debug logs (and `--strace` in root-cause runs), Kata agent and hypervisor logs, and dmesg.
- All software versions and host configuration (kernel, runsc, Kata, Docker/containerd, Harbor, CPU governor).

### 6.5 RQ3 specifics

- **RQ3a (zero tokens):** take published per-instance agent patches for SWE-bench, which the SWE-bench
  experiments repository publishes for leaderboard submissions. Verify the license and availability. Re-run
  the verifier for each patch under each backend. Metric: per-submission score change and rank changes.
- **RQ3b (tokens):**
  - One or two inexpensive models through an API, with `terminus-2` or `mini-swe-agent`.
  - About 40 TB2 tasks × 3 backends × k=5 attempts.
  - **Pilot first** to measure cost per trial, then size k and the task count to fit the budget.

---

## 7. Methodology details

### 7.1 Statistics (analysis plan written before the main runs)

- **Per task:** Fisher's exact test comparing a backend's pass count with runc's. Benjamini–Hochberg FDR
  control across tasks.
- **Aggregate flip rate:** proportion of tasks with a significant discordance, with Wilson 95% CI, compared
  against the A/A floor.
- **Paired agent comparison (RQ3b):** mixed-effects logistic regression (backend as fixed effect, task as random
  effect) or a paired bootstrap over tasks. Report effect sizes in percentage points with 95% CIs.
- **Rankings (RQ3a):** Kendall's τ between per-backend leaderboards, and the count of pairwise rank reversals.
- **Flip classes:**
  - *Deterministic flip:* always passes under one config, never under the other.
  - *Stochastic shift:* the pass probability changes.
  - Each flip is also labeled *loud* or *silent*.
- **Pre-registration:** commit `analysis_plan.md` to the repo before the main experiments. Any later deviation is
  labeled exploratory.

### 7.2 Root-cause protocol (RQ2)

For every confirmed flip:
1. Diff the verifier output and logs between backends.
2. Check cgroup events and the OOM, PID, and timeout signals.
3. Compare durations against the timeout (margin analysis).
4. Re-run with syscall tracing (`strace` under runc, `runsc --strace` / debug logs under gVisor) and diff the
   syscall sequences around the failure.
5. **Minimize** to a standalone reproducer: a few lines of shell or Python that flip on their own.
6. Classify into the taxonomy. File upstream issues where the cause is a runtime bug, and link them in the paper.

### 7.3 Candidate root-cause taxonomy (hypotheses to confirm or replace)

| Class | Example mechanism (hypothesised) |
|---|---|
| C1 Resource-limit semantics | OOM or PID limit kills the whole sandbox (gVisor) vs one process (runc). Runtime overhead counted against the limit |
| C2 Slowdown → timeout | Syscall-heavy builds and tests run slower, so the test or task times out. Verdict depends on the margin |
| C3 Kernel-interface differences | `/proc`, `/sys`, reported kernel version, visible CPUs and memory. Tools size thread pools or behave differently |
| C4 Unsupported or partial features | Syscalls, ioctls, or kernel features missing or partially implemented in the runtime |
| C5 Filesystem semantics | Overlay vs gofer vs virtio-fs: locking, inotify, mmap sharing, permissions, xattrs, `/dev/shm` |
| C6 Terminal/device behavior | TTY and pseudo-terminal behavior, devices (cf. Tencent's tmux CPU spin) |
| C7 Networking | Different network stack, DNS, loopback, or egress behavior |
| C8 Timing/clock | Timer resolution, clock sources, scheduling jitter that exposes test flakiness |

### 7.4 Timeout-margin model (analytical component)

For each task, measure its baseline duration distribution D under runc and its slowdown factor s_b under backend b.
Predict P(timeout flip) = P(s_b · D > T), then check the prediction against observed C2 flips.
- This produces a simple, falsifiable model plus a practical rule: "a task needs margin ≥ X× on the
  slowest backend".
- Also report the per-task slowdown distribution for runsc and Kata. This links to the performance
  literature without being the main focus.

### 7.5 RL impact model (RQ4)

- For task t, the agent success probability is p_t (from RQ3b or public pass rates), and the backend-induced
  silent-failure probability is f_t (from RQ1/RQ3a). The observed success probability is p′_t = p_t(1−f_t), plus
  false passes where the null solution passes.
- For GRPO group size G, compute:
  - P(zero-variance group) under the true and observed rewards
  - expected advantage bias
  - fraction of groups whose best and worst trajectories swap
- Validate by simulation using the measured p_t and f_t. Report sensitivity over G ∈ {4, 8, 16}.

### 7.6 Mitigation: conformance pre-flight

Before admitting a task to a training or eval pool on backend b:
- Run the oracle and null n times on b.
- Reject the task if their verdicts are not stable, or if the timeout margin is below the threshold.

Measure:
- recall on known flips
- false-rejection rate
- cost in sandbox-minutes, compared with a typical rollout budget

Compare against "trust runc results" (no check) and "check on runc only".

---

## 8. Testbed

- **One bare-metal Linux server with KVM.** Kata and the gVisor KVM platform need `/dev/kvm`, and most cheap VMs
  don't expose nested virtualization.
  - ≥8 cores, ≥64 GB RAM, ≥1 TB NVMe. SWE-bench images are large.
  - Ubuntu LTS. Pinned kernel. CPU governor set to `performance`. SMT and turbo settings recorded.
- **Software:** Docker/containerd with runtimes registered (`runc`, `runsc`, Kata shim), Harbor (pinned).
  - A thin custom Harbor environment, or a docker runtime override, so the backend is a per-run parameter.
    How Harbor passes `--runtime` needs verifying in the pilot.
  - Kata may be easier via containerd/nerdctl or a single-node Kubernetes cluster with `RuntimeClass`. Decide in the pilot.
- **Development** on Mac or a local VM. All measurements on the bare-metal box only.
- **Everything scripted:** `make grid`, `make aa`, `make rootcause TASK=…`. Results are written as append-only
  JSONL plus raw logs.

## 9. Budget (cap $100)

| Item | Estimate | Notes |
|---|---|---|
| Bare-metal server, about 1–1.5 months | $45–65 | Main cost. Check current auction or dedicated prices |
| API tokens for RQ3b | $25–35 | Sized after the pilot. Drop to RQ3a-only if needed |
| Storage/egress | ~$0–5 | Keep images local |
| **Total** | **≤ $100** | RQ1, RQ2, RQ3a, and RQ4 need no tokens and no GPU |

---

## 10. Milestones

| Week | Milestone | Exit criterion |
|---|---|---|
| 1–2 | Read the related work in §4 in full. Pilot on Mac or VM: 5 TB2 tasks, oracle and null, runc vs runsc | Harness runs the backend as a parameter. Pilot shows feasibility and run times |
| 3 | Bare-metal box: runc, runsc, Kata all working under Harbor. Write `analysis_plan.md` | Plan committed. 5-task smoke test passes on all three |
| 4–5 | RQ1 screening grid + A/A runc repeat | All cells n=3. Noise floor measured |
| 5–6 | RQ1 confirmation runs, T and N follow-ups | Flip list with significance and loud/silent labels |
| 6–8 | RQ2 root-causing, minimized reproducers, upstream issues | Every confirmed flip classified, or marked "unexplained" with evidence |
| 8–9 | RQ3a patch re-verification. RQ3b pilot, then runs | Score deltas with CIs, ranking stability |
| 9–10 | RQ4 model, simulation, pre-flight evaluation | Figures for GRPO corruption and pre-flight recall/cost |
| 10–12 | Write. Re-check novelty. External feedback from 1–2 people. Release artifact. arXiv | 6-page version plus appendix. Artifact reproduces the main figures |

## 11. Paper outline (6-page workshop version)

1. **Introduction:** a motivating flip example. Contributions:
   - first systematic cross-backend reward study
   - loud vs silent flips
   - root-cause taxonomy
   - RL impact and pre-flight
   - artifact
2. **Background:** isolation backends (runc / gVisor / Kata), Harbor task model, GRPO advantages.
3. **Methodology:** factors, noise floor, statistics, root-cause protocol.
4. **RQ1 prevalence:** flip-rate figure by backend × resource mode, plus the loud/silent split.
5. **RQ2 mechanisms:** taxonomy table with counts and one worked example traced down to the syscall level.
6. **RQ3 effect on scores:** patch re-verification deltas and ranking stability. End-to-end agent deltas.
7. **RQ4 RL implications and mitigation:** GRPO corruption model and pre-flight results.
8. **Related work, limitations, conclusion.**
9. **Appendix:** full per-task tables, versions, the timeout-margin model, and reproducers.

**Key figures:**
- Heatmap of flip rate (backend × resource mode)
- Stacked bars of loud vs silent flips by root-cause class
- Timeout-margin predicted vs observed
- Score deltas with CIs
- GRPO corrupted-group fraction vs G

## 12. Threats to validity

- **Internal:**
  - *Host noise and contention:* fixed concurrency, A/A floor, repeated runs on different days.
  - *Ordering effects:* randomized run order.
  - *Image caching:* cold and warm states controlled and recorded.
- **Construct:** an "oracle pass" is a proxy for task validity. Tasks broken for non-infrastructure reasons are
  filtered out using runc behavior and cross-checked against the OpenAI audit where possible.
- **External:**
  - One host and one hardware generation.
  - Specific runtime versions. Runtimes evolve (gVisor fixes land quickly), so results are tied to versions.
    That is why we report versions and minimized reproducers rather than only rates.
  - Two benchmarks. Not all agent workloads.
  - Kubernetes-managed production settings may differ. Discuss where cluster factors (scheduling, node
    pressure) would add effects.
- **Statistical:** multiple comparisons (BH-FDR), small n per cell (adaptive confirmation), pre-registered plan.
- **Novelty decay:** this is a fast-moving area. Re-run the novelty search (Anthropic and gVisor blogs, arXiv cs.DC/cs.SE,
  EuroMLSys/MLSys programs) at week 10.

## 13. Artifact

- Harness (backend-parameterized Harbor runner, grid driver, log collectors).
- Raw JSONL results and logs (as size permits). The labeled flip dataset (task, config, class, reproducer).
- Analysis notebooks that regenerate every figure and table.
- Minimized reproducers, plus links to the upstream issues they led to.

## 14. Working practices (the "second-year PhD" bar)

- **Lab notebook:** a dated `notebook.md`. Every run batch has a hypothesis, a command, and an outcome. Negative
  results stay in.
- **Figures from scripts only.** No hand-edited numbers. Every number in the paper traces back to a file.
- **Weekly self-review:** what did I learn, what claim am I making now, what evidence would change it?
- **External feedback:** send the 1-page RQ summary (week 3) and the full draft (week 11) to 1–2 people
  who work on agent infra or sandboxing. Related-work authors are good candidates.

## 15. Open decisions

- Kata via Docker shim, containerd/nerdctl, or single-node Kubernetes with `RuntimeClass` (decide in the pilot).
- Whether to include the gVisor KVM platform and Kata+Firecracker (stretch, if time allows).
- Model(s) for RQ3b, chosen by cost per trial after the pilot.
- SWE-bench Verified vs Pro for the SWE subset.

## 16. References (starting set)

- Anthropic. *Quantifying infrastructure noise in agentic coding evals.* 2026. https://www.anthropic.com/engineering/infrastructure-noise
- gVisor blog. *Scaling Agentic-RL Sandboxes to the Millions with gVisor at Tencent.* 2026. https://gvisor.dev/blog/2026/04/23/scaling-agentic-rl-sandboxes-to-the-millions-with-gvisor-at-tencent/
- *Sandman: A Sandbox Service for Production Agentic Reinforcement Learning.* arXiv 2610.39711. https://arxiv.org/abs/2610.39711
- *The Rollout Infrastructure Tax in Coding-Agent Reinforcement Learning.* arXiv 2607.01415. https://arxiv.org/abs/2607.01415
- *BunshinRL: Whole-Workspace Forking for Agentic RL Rollouts.* arXiv 2611.10453. https://arxiv.org/abs/2611.10453
- *Stop Comparing LLM Agents Without Disclosing the Harness.* arXiv 2605.23950. https://arxiv.org/abs/2605.23950
- *DeltaSelect: Affordable A/B Testing for Coding Agents.* arXiv 2609.19607. https://arxiv.org/abs/2609.19607
- OpenAI. *Separating signal from noise in coding evaluations.* 2026. https://openai.com/index/separating-signal-from-noise-coding-evaluations
- AI21. *Lessons from 200,000 SWE-bench runs.* 2026. https://www.ai21.com/blog/scaling-agentic-evaluation-swe-bench/
- sandboxbench (runc vs gVisor on agent-shaped workloads). https://github.com/athish-shubhan/sandboxbench
- Harbor framework. https://github.com/laude-institute/harbor · https://docs.harborframework.com
- Background to add: the gVisor design docs, Kata Containers architecture docs, the Firecracker paper (NSDI '20),
  the original SWE-bench and Terminal-Bench papers, and GRPO (DeepSeekMath).
