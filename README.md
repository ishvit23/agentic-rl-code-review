# Why Code Review Agents Are Leaving Their Best Signal on the Table

**Ishvit** · AI/ML Intern · Working with agentic AI systems and internal code review tooling

---

## The Problem I Kept Running Into

For the past few months I've been building and iterating on an internal code review agent in Python at my company. At some point the same frustration kept surfacing: the agent would flag something real, a developer would fix it, and that signal just... disappeared. Nothing fed back into the model. The next PR started from zero.

That's the core architectural gap in every code review agent right now — CodeRabbit, Qodo, all of them. They're stateless inference pipelines. They don't learn from what developers actually accept or reject.

This is a writeup of what I think closes that loop.

---

## What the Current Architecture Looks Like

Every code review agent today is roughly this:

```
PR opened → webhook fires → diff assembled → LLM called → comments posted
```

That's it. No memory between PRs. No learning from outcomes. The model is the same after reviewing 10,000 PRs as it was on PR #1.

The signal being wasted is enormous:
- Which comments got resolved vs. dismissed vs. ignored
- Whether a flagged issue was actually fixed in a follow-up commit
- Whether a suggested fix was applied verbatim, modified, or rejected
- Developer reply sentiment

All of this is sitting in GitHub's API right now. None of it is being used for training.

---

## Why This Is an RL Problem, Not a Prompting Problem

You can't prompt your way out of this. No matter how good your system prompt is, the model has no mechanism to learn that *this codebase's team hates nitpicky style comments* or *this class of bug is always a false positive in Python async code*. That requires a feedback loop.

The reason code review is a particularly good fit for RL — better than most NLP tasks — is that it has **delayed but verifiable ground truth**. When the agent flags a bug and the developer fixes it in the next commit, that's a clean reward signal. You almost never get that in open-ended generation.

The missing piece is infrastructure to capture it and a training loop to use it.

---

## How Trinity-RFT Maps to This Problem

[Trinity-RFT](https://github.com/agentscope-ai/Trinity-RFT) is a framework from Alibaba's AgentScope team that decouples exploration, experience storage, and training into three components. It maps almost directly onto the code review domain:

### Explorer → The Review Agent

Instead of a single LLM call, the Explorer runs a multi-step agentic workflow:

```python
# Conceptual — not the actual Trinity-RFT API
trajectory = []

diff = tools.parse_diff(pr)                        # step 1
callers = tools.lookup_callers(diff.changed_fns)   # step 2
tests = tools.check_test_coverage(diff)            # step 3
history = tools.get_file_history(diff.files)       # step 4
comment = agent.generate_review(diff, callers,
                                 tests, history)   # step 5

trajectory.append((diff, callers, tests,
                   history, comment))
```

Every tool call and intermediate state gets recorded as a rollout. This is what makes the Explorer different from a plain LLM call — the full reasoning trace is captured, not just the output.

### Buffer → Experience Store with Delayed Rewards

The Buffer stores rollouts and attaches reward signals when they arrive — which in code review can be hours or days later:

```python
# Reward function — this is the key design decision
def compute_reward(comment, outcome):
    if outcome == "fix_committed":       return +1.0   # bug was real
    if outcome == "suggestion_applied":  return +0.8   # fix was good
    if outcome == "resolved":            return +0.3   # soft signal
    if outcome == "dismissed":           return -0.5   # false positive
    if outcome == "ignored":             return -0.2   # noise
    return 0.0
```

The Buffer supports off-policy storage — you can use historical PR data with reconstructed rewards to bootstrap training before you have live signal.

### Trainer → GRPO on Accepted Reviews

The Trainer runs Group Relative Policy Optimization (GRPO) on batches pulled from the Buffer. The key property of GRPO here is that it's more stable than PPO for long-horizon tasks and handles sparse rewards well — both relevant for code review where reward signals arrive asynchronously.

---

## The Reward Signal Design

This is where most implementations would fail. There are two classes of signal:

**Hard rewards (verifiable):**
- Bug flagged → fixed in follow-up commit: `+1.0`
- Security vuln confirmed: `+1.0`
- Suggested code applied verbatim: `+0.8`

**Soft rewards (preference-based):**
- Comment resolved: `+0.3`
- Comment dismissed: `-0.5`
- LLM-as-judge score on review quality: `±0.0–0.5`
- Developer reply sentiment: `±0.1–0.3`

The ratio matters. Too much soft reward and the model learns to write confident-sounding comments that get resolved even if they're wrong. You need hard reward to anchor it.

One subtle problem: GitHub's API doesn't distinguish *"resolved because acted on"* from *"resolved to get rid of it."* Building that distinction into your telemetry is a product problem, not an ML problem — but it's the prerequisite for everything else.

---

## Expected Improvements (Realistic Estimates)

Based on analogous RL fine-tuning results in code generation tasks:

| Dimension | Expected Gain | Confidence |
|---|---|---|
| False positive reduction | −30 to −50% | High |
| Comment acceptance rate | +25 to +40% | High |
| Bug detection precision | +20 to +35% | Medium |
| Context relevance | +30 to +50% | Medium |
| Security vuln catch rate | +15 to +25% | Medium |

The false positive reduction is the highest-confidence gain because the reward signal is clearest (dismissed = bad) and the current baseline is weakest. Developers stop reading reviews when noise is high — fixing this matters more than marginal accuracy gains.

---

## The Real Blockers

**1. Reward telemetry infrastructure**
Capturing clean accept/reject/commit-follow-up signals at scale is 2–3 months of engineering before any ML work starts. Most teams underestimate this.

**2. Model ownership**
You can't run GRPO on a model you're calling via API. This means committing to a self-hosted open model — Qwen2.5-Coder 32B is the obvious candidate (strong coding baseline, Apache-2 license). At scale the economics work out, but it's a significant architectural commitment.

**3. Reward hacking**
An RL agent will learn to game whatever proxy you give it. If "resolved" is your signal, it learns to write things that get resolved. Careful reward design and regular human eval are non-negotiable.

**4. Distribution shift**
A model trained on public GitHub repos behaves differently on an enterprise Java monorepo. Trinity-RFT's BOTS algorithm (dynamic task curriculum based on difficulty) is directly applicable here.

---

## The Strategic Picture

The tools in this space — CodeRabbit, Qodo, others — are sitting on proprietary training data that no academic lab has: millions of PR reviews, developer reactions, follow-up commits. If either of them built this pipeline, the moat from that data combined with RL would be substantial.

The flywheel: more PRs → richer reward signal → better model → higher acceptance rate → more adoption → more PRs.

The framework (Trinity-RFT) exists. The algorithms (GRPO) are proven in adjacent domains. The missing piece is the telemetry infrastructure and the organizational will to own model weights.

---

## What I've Been Working With

- Built an internal code review agent in **Python** using an agentic architecture
- Spent the last several months iterating on context assembly, tool orchestration, and developer feedback integration
- This writeup came out of trying to answer: *"why doesn't our agent get better over time?"*

That question led here.

---

*Ishvit — AI/ML Intern working on agentic systems*  
*Open to conversations about agentic AI, RL for developer tooling, and code intelligence*
