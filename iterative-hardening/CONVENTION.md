# Iterative Hardening

Red-team → fix → prune loop for agent harnesses. Each round finds new failures, fixes the harness, and trims redundant tests. Runs for arbitrary rounds via dynamic stop hooks or cron.

---

## The Loop

```
┌─────────────────────────────────────────────────────────────┐
│                   ITERATIVE HARDENING                       │
│                                                             │
│  ┌──────────────┐                                           │
│  │ 1. PROBE     │  Subagent swarm (concurrency=5)           │
│  │              │  Find 5 tests the harness fails           │
│  └──────┬───────┘                                           │
│         ▼                                                   │
│  ┌──────────────┐                                           │
│  │ 2. FIX       │  Optimize harness until new tests pass    │
│  │              │  Regression-sample 20% of existing suite  │
│  └──────┬───────┘                                           │
│         ▼                                                   │
│  ┌──────────────┐                                           │
│  │ 3. PRUNE     │  Remove duplicates + tests subsumed by    │
│  │              │  harder new additions                     │
│  └──────┬───────┘                                           │
│         ▼                                                   │
│  ┌──────────────┐                                           │
│  │ 4. CONTINUE  │  Loop back to PROBE                       │
│  │              │  Stop via hook or cron                    │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1: Probe

Launch a subagent swarm to adversarially design tests the harness fails.

**Parameters:**
- Concurrency: 5 subagents in parallel
- Target: 5 confirmed failures (not 5 test proposals—5 that actually fail)
- Each subagent independently designs and runs tests against the harness

**Distinguish real failures from bad tests.** A failing test is only valuable if the test itself is well-designed. Before counting a failure, verify:

| Check | Bad test signal |
|-------|----------------|
| Is the expected behavior clearly specified? | Vague or ambiguous success criteria |
| Is the test deterministic? | Flaky—passes/fails on identical input |
| Does the test match a real user scenario? | Contrived edge case no user would hit |
| Is the assertion correct? | Test expects wrong answer |
| Is the test self-contained? | Depends on external state or prior tests |

Discard tests that fail these checks. They inflate failure counts without improving the harness. A subagent that produces 5 bad tests and 0 real failures has found nothing.

**Subagent prompt structure:**
```
You are red-teaming a [harness type]. Your goal: design a test case that
the harness handles poorly. The test must be:
- A realistic user scenario (not adversarial gibberish)
- Deterministic and self-contained
- Clearly specified with unambiguous expected behavior

[Harness description / API / tool list]
[Existing test examples for calibration]

Design a test, run it, and report: test definition, actual output,
expected output, and why this is a real failure (not a bad test).
```

---

## Step 2: Fix

A single fix agent optimizes the harness until all 5 new tests pass. Changes can include:

- Adding information (system prompts, context, examples)
- Designing new tools or modifying existing ones
- Changing the approach (routing, fallback logic, prompting strategy)
- Adjusting parameters (temperature, model, retry logic)

**After fixing, regression-sample existing tests.** Uniformly sample $n$ tests from the full existing suite and run them. Default: $n$ = 20% of total test count. If any sampled test regresses, fix before proceeding.

Why 20%: full suite runs are expensive. 20% catches most regressions while keeping cycle time short. Increase to 50% or 100% if the harness change was structural (new model, rewritten system prompt, changed tool set).

**Fix agent must:**
1. Run each new test individually to confirm it fails pre-fix
2. Make the minimal change that fixes the failure
3. Re-run all 5 new tests to confirm they pass
4. Sample and run $n$ existing tests
5. If regressions found, fix and re-sample until clean

---

## Step 3: Prune

After new tests are added and the harness is fixed, prune the test suite.

**Remove:**
- **Exact duplicates:** identical input and expected output
- **Near-duplicates:** same scenario with trivial variation (rephrased question, different names)
- **Subsumed tests:** an existing test that is strictly easier than a newly added test covering the same capability

**Keep:**
- Tests covering distinct capabilities or failure modes
- Tests at different difficulty levels if they exercise different code paths
- Regression tests for past bugs (even if "easy")—mark these explicitly

**Pruning heuristic:** For each new test added, scan existing tests that exercise the same capability. If an existing test is a strict subset (same skill, easier scenario, no unique code path), remove it.

---

## Step 4: Continue

Loop back to Step 1. The number of rounds is controlled externally—the loop itself has no built-in termination.

### Termination via dynamic stop hook

```bash
# Stop after N rounds
fleet hook add --event PreToolUse \
  --check 'test $(cat .hardening-round) -lt 5' \
  --description "Stop hardening after 5 rounds"
```

Or gate on a quality signal:

```bash
# Stop when probe can't find 5 failures within 20 attempts
fleet hook add --event PreToolUse \
  --check 'test $(cat .probe-miss-rate) -lt 75' \
  --description "Stop when harness is robust"
```

### Termination via cron

```bash
# Run one round every 30 minutes, stop after 3 hours
fleet cron add --every 30m --max-runs 6 \
  "fleet run --spec hardening-round.agent.yaml"
```

### Round tracking

Persist round state in the working directory:

```
.hardening-round          # current round number
.hardening-log.jsonl      # per-round summary (tests added, pruned, regressions)
tests/                    # the growing/shrinking test suite
```

Each round appends to `.hardening-log.jsonl`:
```json
{"round": 3, "new_tests": 5, "pruned": 2, "regressions": 0, "suite_size": 47}
```

---

## Configurable Parameters

| Parameter | Default | When to change |
|-----------|---------|----------------|
| Probe concurrency | 5 | Lower if rate-limited, raise if fast harness |
| Failures to find | 5 | Lower for early rounds, raise for mature suites |
| Regression sample % | 20% | Raise after structural changes |
| Max rounds | unbounded | Set via hook or cron |
