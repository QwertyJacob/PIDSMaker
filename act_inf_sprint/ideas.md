# Idea & Things to Test — Stream-Interruption Policies for PIDS

PIDSMaker's provenance-based intrusion detectors (Velox, Orthrus, and others) score system entities by processing their incoming edges as a stream, but every existing system decides when to trust that score using a fixed, hand-set rule — a fixed wait time (ThreaTrace) or a fixed time window (Kairos). This project asks whether that decision should instead be a learned, adaptive policy: for each entity, keep watching its incoming edges or stop and commit to a verdict now, under an explicit budget on how long detection is allowed to take. This reframes PIDS evaluation around Availability — how much detection latency a deployment can afford — rather than raw accuracy, and turns "when to stop" into a sequential decision-making problem we can hand to different policy classes: the fixed heuristics already in PIDSMaker, classical sequential statistics (Wald's SPRT, change-point detection), reinforcement learning, and active inference, and compare them on the same footing. The goal is less "beat the SOTA detector" and more "build the benchmark that shows which policy makes the best latency/accuracy trade-off, and why" — with active inference's explicit handling of uncertainty being the main scientific bet worth testing.

## What is the problem?

We're given a stream of edges, and we want to decide when to
 *stop now and commit to a verdict*, or
*wait for more edges* ?
 The budget is detection latency, how
long a node stays "open" waiting for a decision, or analyst-facing delay.

## Is this already explored?

Short answer: **partially, and not the way we want.**

- Two baselines already shipped in PIDSMaker have crude, *fixed*, non-adaptive versions
  of exactly this idea:
  - **ThreaTrace** queues flagged nodes and waits a fixed time `T` before committing —
    if a node hasn't reverted to "benign" within `T`, it's popped and declared malicious.
    `T` is a hand-set hyperparameter, not learned, not per-node.
  - **Kairos** decides in fixed 15-minute windows. The window size is fixed globally,
    not adapted to how much evidence a given node actually needs.
  - Neither treats "how long to wait" as something to *optimize*, per entity, under a
    cost model. That's the actual gap — not the idea of waiting itself.
- The general problem — decide when accumulated streaming evidence is enough, trading
  confidence against delay — is a well-solved classical statistics problem: **Wald's
  Sequential Probability Ratio Test** (provably minimizes expected samples needed for
  a given error rate) and **quickest change-point detection** (CUSUM / Shiryaev–Roberts
  / Bayesian Online Changepoint Detection, BOCPD). A 2025 streaming-IDS paper uses BOCPD
  for a very similar alert-timing decision, but at the network-flow level, not on
  provenance graphs, with no learned/adaptive policy.
- Closest 2026 adjacent work, **CALIBURN**, calibrates *alert thresholds* to a fixed
  alert budget (SRE-style burn-rate policies). That's a budget problem, but it picks a
  threshold, not a stopping time — different lever.
- RL-for-optimal-stopping is methodologically well trodden (this is how American-option
  pricing and similar problems are framed), and there's a fresh non-security precedent:
  **MistExit (2026)**, an RL policy that decides how much of a streaming video to watch
  before committing to a classification, explicitly to save an observation budget.
- On the Active Inference side: the **drift-diffusion model** (evidence accumulates to a
  boundary, then you commit) is mathematically the same shape as this problem, and there
  is published work directly connecting DDMs to active inference. This gives real
  theoretical grounding for AIF here — the two are already known to be equivalent in
  this exact "accumulate, then stop" setting, unlike, say, forcing AIF onto a problem
  it wasn't built for.

**Conclusion:** nobody appears to have framed provenance-graph PIDS as a learned,
per-node, budget-constrained *optimal stopping* problem, benchmarked against RL and
against classical sequential-testing baselines. That's defensible — but SPRT/CUSUM are
strong, cheap, and need to be in the comparison from day one, or a reviewer will ask
why you didn't just use Wald's test.

## Best way to model it (sketch — to be refined by the experiments below)

Per node `n`, at each new incident edge `e_t`:

- **Belief**: posterior over "n is malicious" given the edge-loss sequence seen so far
  (native to AIF; for RL it's a hand-built running feature — running max/mean loss,
  edge count, time elapsed).
- **Actions**: `{declare benign, declare malicious, wait for next edge}`.
- **Cost**: −(false negative, high) −(false positive, lower but nonzero — matches the
  benchmark's own precision/ADP framing) −(per-step waiting cost — the budget itself:
  edges, wall-clock time, or "node held open" memory).
- **Policies to compare**: (a) fixed-T / fixed-window — free, already in the repo via
  ThreaTrace/Kairos; (b) SPRT/CUSUM on the loss sequence — no training needed, provably
  optimal under stated assumptions; (c) DQN/PPO; (d) active inference (expected free
  energy = pragmatic + epistemic value).

##  Experiments to run first

For each: what to do in PIDSMaker terms, what it answers, why it matters *before*
writing any policy code.

### E1 — Shape of the evidence trajectory
Run Velox (and Orthrus) once per dataset as usual, but log the **running max loss per
node after each incident edge** instead of only the final score (`node_evaluation.py`
already computes this per-edge — just keep the trajectory instead of discarding it).
Plot malicious vs. benign trajectories.
→ Do malicious nodes' scores spike early and stay up (stopping early is easy and safe),
or drift up slowly/noisily (stopping early is risky, cost model needs rethinking)? This
is the single most important sanity check — if trajectories are indistinguishable until
very late, the paper's premise weakens a lot.

### E2 — Natural edge-count / duration per node
Histogram: edges per node, and node lifetime (first-to-last edge), malicious vs. benign,
per dataset.
→ Is there even a meaningful "wait longer" lever? Should the budget be edge-count-based
or wall-clock-based? Also flags a confound: if malicious nodes are just structurally the
ones with unusually many/few edges, that's a leak the policy could exploit trivially,
undermining the "smart stopping" story.

### E3 — Headroom above the existing fixed-T baselines
From E1's trajectories: for each malicious node, find the earliest edge-index where the
running score crosses the eventual detection threshold, and compare to what ThreaTrace's
fixed-T / Kairos's fixed-window would actually have waited for.
→ How much is there to gain? If fixed-T is already near the earliest possible detection
point, an adaptive policy has little room to add value — better to know that in week 1.

### E4 — Does "wait longer with Velox" substitute for "pay for Orthrus"?
Compare Velox's node score using only the first `k` edges vs. all edges, against
Orthrus's score at the same `k`, for increasing `k`.
→ This directly tests your question: is the escalation axis redundant with the
stopping-time axis, because Orthrus's advantage partly comes from implicitly
aggregating more of a node's own history (via `tgn_last_neighbor`) than Velox gets from
the same amount of *waiting*? If the gap closes fast with `k`, the two axes are
entangled — commit to one (stopping-time) for a focused paper rather than modeling both.

### E5 — Reward sparsity check
Count, per dataset, how many decision points would ever carry a true-positive label,
given the benchmark's documented malicious-node prevalence (~1:10,000 to 1:1,000,000).
→ Is plain RL/AIF training on real attack days even feasible, or do you need reward
shaping, synthetic anomalies calibrated on E1's real trajectories, or a proxy pretraining
task (e.g., predict eventual max loss from partial history) before the policy sits on top?

### E6 — Budget currency
From E2's timing data: are inter-edge gaps regular enough that "edges observed" and
"wall-clock time" are roughly interchangeable, or do they diverge enough (bursty
traffic, idle processes) that you need to pick one deliberately and justify it?

## 5. What "done" looks like for this pre-registration phase

One script producing: the E1 trajectory plot, the E2 histograms, and the E3 headroom
number, run on E3-CADETS first (smallest, fastest). That's enough to decide — before
writing any policy code — whether stopping-time should be the sole axis, whether it
should be combined with escalation, and what the budget's unit of currency should be.