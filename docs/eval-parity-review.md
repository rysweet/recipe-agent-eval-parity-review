# 100-Agent Distributed Eval Restoration: Critical Review & Recommendations

**Date:** 2026-03-19
**Context:** Live 100-agent eval scored 0.209 overall — 9 perfect answers, 39 zero-score answers, 30 explicit no-answer timeouts. Fleet health: 100/100 online, 100/100 ready, 5000 progress events emitted normally.

---

## Executive Summary

The 0.209 score is not a correctness regression — it is a measurement failure caused by four compounding infrastructure problems that together prevented valid answers from being counted:

1. Fanout too low → wrong shards queried → retrieval misses
2. Hard answer timeout too short → stale-but-valid answers discarded
3. Concurrency too high → tail latency drives answers past the timeout
4. Failover gate too narrow → semantic abstentions not retried

Fixing these four issues should dramatically close the gap with the single-agent baseline before any deeper architectural changes are needed.

---

## (a) Obvious Pre-Rerun Fixes

These must be confirmed **before any rerun** — local patches that are not deployed are worth nothing.

### Fix 1: Raise AMPLIHACK_MEMORY_QUERY_FANOUT to 20–40

**Current state:** Default fanout is 5. On a 100-agent fleet, each query touches only 5 shards — **5% coverage**. A question whose answer lives in any of the other 95 shards will reliably return a retrieval miss.

**Recommendation:** Set `AMPLIHACK_MEMORY_QUERY_FANOUT=20` as the minimum for 100-agent parity runs. Prefer 30–40 to reach 30–40% shard coverage, which is the inflection point where hit-rate starts to saturate for uniformly distributed memory.

**Rationale:** At fanout=5 the expected recall is ~5%. At fanout=20 it is ~20%. Given 39 zero-score answers, even a linear lift to 20% coverage — holding everything else fixed — would transform a large fraction of those into scoreable answers. This is the single highest-leverage fix.

**Deploy check:** Verify `AMPLIHACK_MEMORY_QUERY_FANOUT` is wired in the deploy script and that the fleet is actually restarted after the change. A running agent with a stale env-var will ignore the new value.

### Fix 2: Set AMPLIHACK_SHARD_QUERY_TIMEOUT_SECONDS Below the Answer Timeout

**Current state:** Unknown whether `AMPLIHACK_SHARD_QUERY_TIMEOUT_SECONDS` is set relative to the answer timeout. If shard timeouts exceed the answer timeout, a slow shard causes the **outer** answer to timeout rather than allowing the shard miss to be logged and skipped gracefully.

**Recommendation:** Set `AMPLIHACK_SHARD_QUERY_TIMEOUT_SECONDS` to at most `answer_timeout - 30s`. With the answer timeout removed (see section b), use an absolute value of 60–90s per shard. This ensures shard timeouts surface as retrieval misses (recoverable) rather than hard answer-level failures (unrecoverable).

**Deploy check:** Confirm this variable is set in the deployed environment, not just locally.

### Fix 3: Confirm the Patched Eval Runner is Deployed

**Current state:** The eval runner has been patched locally to forward `parallel_workers` and `failover_retries` knobs, but has **not been redeployed**.

**Recommendation:** Deploy before rerunning. This is a mandatory gate — running the unpatched runner against a 100-agent fleet with the new env-vars will produce the same failure modes as the original run.

**Deploy check:**
- The `parallel_workers` knob is present and defaults to 4
- The `failover_retries` knob is present and defaults to 2
- Scale-aware defaults activate at fleet size ≥ N (confirm threshold)
- Verify with a dry-run invocation that prints resolved config

### Fix 4: Confirm 300s Shell-Runner Timeout is in Deployed Config

**Current state:** The shell-runner timeout has been bumped to 300s locally for 100-agent runs. This bump is not confirmed to be deployed.

**Recommendation:** Before rerunning, print the active shell-runner config from the deployed environment and confirm `answer_timeout_seconds=300` (or equivalent). A deployed runner with 180s timeout will discard any answer that arrives between 180s and 300s — exactly the pattern seen in the stale-answer logs.

---

## (b) Eval-Level Answer Timeout: Disable Entirely for Correctness-First Parity

**Recommendation: Remove the hard per-question wall-clock timeout for correctness-first parity runs. Replace with a soft warning log.**

### Rationale

The user requirement is explicit: **zero timeouts**. The live run produced 30 no-answer entries described as explicit timeout events. Several of those answers arrived "a few seconds" after the 180s deadline — meaning the underlying agents produced valid answers, but the eval runner discarded them.

This is the proximate cause of at least the timeout bucket of failures. A timeout that discards a valid answer is not a quality signal — it is a measurement artifact.

**Hard timeout harms correctness-first evaluation in two ways:**

1. **Direct discard:** Answers that arrive 1–5s late are counted as failures regardless of content quality.
2. **Concurrency interaction:** At concurrency=10 (see section c), the P99 latency of any batch of 10 questions is driven by the slowest question. With 100 agents each running 10 concurrent questions, the system has ~1000 in-flight questions simultaneously. Tail latency at this scale routinely exceeds a hard 180s deadline even when individual questions are fast.

**Replacement:** Log a warning when any answer exceeds a soft threshold (e.g., 120s), but do not discard it. This preserves observability without introducing artificial failures.

**When to reintroduce a hard timeout:** After parity is confirmed on correctness, a hard timeout can be reintroduced for throughput-constrained production runs, but should be set at ≥ 2× the observed P95 latency of the slowest question type.

---

## (c) Question Concurrency: Drop to 1–2 for Correctness-First Parity

**Recommendation: Set `long_horizon_memory` question concurrency to 1 or 2 workers for parity runs.**

### Rationale

The current default of 10 concurrent workers was designed for throughput on small fleets. On a 100-agent distributed fleet, it creates a tail-latency amplification problem:

| Concurrency | In-flight questions (100 agents × N) | P99 latency pressure |
|-------------|--------------------------------------|---------------------|
| 10          | ~1000                                | Very high — 100 slow questions can stall all agents |
| 2           | ~200                                 | Moderate            |
| 1           | ~100                                 | Low — each agent serializes its questions |

With concurrency=10 and a 180s timeout, any question that hits a slow shard or a degraded agent takes a slot for up to 180s, starving the other 9 concurrent questions of resources and causing cascading latency.

**For correctness-first parity, the goal is answer quality, not throughput.** Run at concurrency=1 for the initial parity validation, then raise to 2–4 in a second pass after confirming scores match the single-agent baseline.

**Concurrency=1 also simplifies debugging:** if a question fails, the failure is isolated and traceable. At concurrency=10, failure attribution is noisy.

---

## (d) Failover on Semantic Abstention: Yes, Trigger It

**Recommendation: Expand the failover trigger to include semantic abstention patterns in addition to transport timeout.**

### Rationale

Some zero-score answers were semantic abstentions — the agent responded with text like:

- "The provided facts do not contain information about..."
- "I cannot find any information about..."
- "No information is available in the context..."

These are **retrieval failures masquerading as answers.** The agent received a valid response from one shard but that shard did not hold the relevant memory. The eval runner counted the response as an answer and scored it zero.

With fanout=5 on 100 shards, the probability that the queried shards miss the relevant memory is ~95%. Many of these zero-score semantic abstentions are correct diagnoses of shard misses — the agent is accurately reporting that *its* shard doesn't have the answer. The fix is to failover to a different shard/agent, not to accept the abstention.

### Patterns to Detect

```python
ABSTENTION_PATTERNS = [
    "the provided facts do not contain",
    "i cannot find",
    "no information available",
    "not mentioned in the",
    "the context does not include",
    "i don't have information about",
    "based on the information provided, i cannot",
]
```

Match case-insensitively on the first 200 characters of the answer.

### Failover Budget

`failover_retries=2` is sufficient for this phase. Each retry should target a **different agent** (not the same one), to maximize the chance of hitting a different shard. The routing logic should:

1. On first abstention, retry against a randomly selected different agent
2. On second abstention, retry against the agent with the highest memory coverage metric (if available), otherwise random

**Important caveat:** Do not trigger failover on genuine negative answers — e.g., "The answer to your question is no." Ensure the abstention detector fires only on the patterns above, not on short or negative responses generally.

---

## (e) Safe Incremental Rerun Sequence

Run these steps **in order**, treating each as a gate. Do not proceed to the next step until the current one passes its validation criteria.

### Step 1: Single-Agent Smoke Test (10 questions)

**Purpose:** Confirm the single-agent baseline still holds after local patches. This rules out regressions introduced by the patches themselves.

**Config:**
- 1 agent
- 10 representative questions from the eval set (include at least 2 that failed in the live run)
- Patches applied locally but not yet deployed to fleet
- Use the same scoring logic as the live eval

**Pass criteria:**
- Score ≥ single-agent baseline ± 0.02
- Zero timeouts, zero no-answer entries
- All 10 questions receive non-abstention answers

**If Step 1 fails:** The patches introduced a regression. Do not proceed. Debug locally before touching the fleet.

---

### Step 2: 5-Agent Run (10 questions, new fanout/timeout settings)

**Purpose:** Verify that the new fanout and timeout settings allow distributed retrieval to reach parity on a small fleet.

**Config:**
- 5 agents
- Same 10 questions as Step 1
- `AMPLIHACK_MEMORY_QUERY_FANOUT=20`
- `AMPLIHACK_SHARD_QUERY_TIMEOUT_SECONDS=60`
- Answer timeout: **disabled** (soft warning log only)
- Question concurrency: 1
- Patched eval runner deployed

**Pass criteria:**
- Score ≥ single-agent baseline ± 0.02
- Zero timeouts
- Zero no-answer entries
- All stale-answer warnings (if any) resolve within 30s of soft threshold

**If Step 2 fails:** Check that env-vars are deployed to the 5-agent fleet. Inspect shard coverage logs to confirm fanout is active. Do not proceed to Step 3.

---

### Step 3: 20-Agent Run (50 questions, semantic-abstention failover enabled)

**Purpose:** Validate that semantic-abstention failover eliminates zero-score abstention answers at moderate fleet scale.

**Config:**
- 20 agents
- 50 questions (include all questions that produced zero-score answers in the live run)
- `AMPLIHACK_MEMORY_QUERY_FANOUT=20`
- `AMPLIHACK_SHARD_QUERY_TIMEOUT_SECONDS=60`
- Answer timeout: disabled
- Question concurrency: 1
- Semantic-abstention failover: enabled, `failover_retries=2`

**Pass criteria:**
- Score ≥ single-agent baseline ± 0.02
- **Zero timeouts**
- **Zero no-answer entries**
- Zero semantic-abstention zero-score answers (abstentions should all result in successful failover)
- Failover rate < 30% of questions (if higher, fanout is still too low)

**If Step 3 fails on abstentions:** Raise fanout to 30–40 and re-run Step 3 before advancing.

---

### Step 4: 100-Agent Run (100 questions, mid-scale gate)

**Purpose:** Confirm parity holds at full fleet scale before committing to a 5000-turn eval.

**Config:**
- 100 agents
- 100 questions (representative sample across question types)
- `AMPLIHACK_MEMORY_QUERY_FANOUT=30`
- `AMPLIHACK_SHARD_QUERY_TIMEOUT_SECONDS=60`
- Answer timeout: disabled
- Question concurrency: 1–2
- Semantic-abstention failover: enabled, `failover_retries=2`

**Pass criteria:**
- Score ≥ single-agent baseline ± 0.02
- Zero timeouts
- Zero no-answer entries
- P99 answer latency < 120s (soft threshold)
- Failover rate < 20% of questions

**If Step 4 fails:** Do not run the full 5000-turn eval. Diagnose using the 100-question result. Common failure modes at this scale:
- Score below baseline: Check shard distribution; some agents may hold denser memories than others
- High failover rate (> 30%): Fanout still too low for this question set; raise to 40
- Latency regression: Raise concurrency back to 2 if 1-worker serialization is the bottleneck

---

### Step 5: Full 100-Agent 5000-Turn Eval

**Gate condition:** Step 4 passes all criteria above.

**Config:**
- 100 agents
- 5000 turns (full eval)
- Same settings as Step 4, with `parallel_workers=4` restored for throughput now that correctness is confirmed
- Semantic-abstention failover enabled
- Answer timeout: disabled (soft warning log at 120s)
- Monitoring: track per-question failover rate, shard-hit distribution, and P99 latency in real time

**Expected outcome:** Score ≥ single-agent baseline ± 0.02. If the score drifts below this after passing Steps 1–4, the issue is likely the deeper architectural mismatch (distributed retrieval path vs. single-agent, distributed data vs. distributed graph) — which is explicitly out of scope for this iteration.

---

## Summary Table

| Dimension | Current State | Recommended Change |
|-----------|---------------|-------------------|
| Memory fanout | 5 (5% coverage) | 20–40 (20–40% coverage) |
| Shard timeout | Unknown/unset | 60–90s, < answer timeout |
| Answer timeout | 180s hard | Disabled; soft warning at 120s |
| Question concurrency | 10 workers | 1 (then 2 after parity confirmed) |
| Failover trigger | Transport timeout only | + Semantic abstention patterns |
| Eval runner | Patched locally, not deployed | Deploy before rerunning |
| Shell-runner timeout | 300s locally, not confirmed deployed | Confirm in deployed config |

---

## Risk Register

| Risk | Mitigation |
|------|-----------|
| Disabling answer timeout causes eval to hang indefinitely | Add per-question soft timeout with warning log; add global eval wall-clock timeout (e.g., 2× expected runtime) |
| Fanout=30 increases agent memory load | Monitor per-agent memory pressure during Step 3; reduce fanout if agents OOM |
| Semantic-abstention detector fires on genuine negative answers | Pattern-match only on the specific strings listed; review 10 flagged samples before enabling in Step 3 |
| Step 5 score drifts after Steps 1–4 pass | This indicates the deeper architectural mismatch; flag as a separate investigation, do not re-tune the knobs from this plan |
