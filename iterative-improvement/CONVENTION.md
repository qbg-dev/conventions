# Iterative Improvement

End-to-end workflow for continuously improving agent harnesses. Combines adversarial red-teaming (probe→fix→prune) with pipeline supervision (health checks, restarts, convergence). Runs for arbitrary rounds via dynamic stop hooks or cron.

---

## The Loop

```
┌─────────────────────────────────────────────────────────────┐
│                   ITERATIVE IMPROVEMENT                      │
│                                                              │
│  ┌──────────────┐                                            │
│  │ 1. PROBE     │  Subagent swarm (concurrency=5)            │
│  │              │  Find 5 tests the harness fails            │
│  └──────┬───────┘                                            │
│         ▼                                                    │
│  ┌──────────────┐                                            │
│  │ 2. FIX       │  Optimize harness until new tests pass     │
│  │              │  Regression-sample 20% of existing suite   │
│  └──────┬───────┘                                            │
│         ▼                                                    │
│  ┌──────────────┐                                            │
│  │ 3. PRUNE     │  Remove duplicates + tests subsumed by     │
│  │              │  harder new additions                      │
│  └──────┬───────┘                                            │
│         ▼                                                    │
│  ┌──────────────┐                                            │
│  │ 4. SUPERVISE │  Health check, restart failures, check     │
│  │              │  convergence, manage cost                  │
│  └──────┬───────┘                                            │
│         ▼                                                    │
│  ┌──────────────┐                                            │
│  │ 5. VERIFY    │  Full E2E pass on deployed system          │
│  │              │  All channels, real user journeys          │
│  └──────┬───────┘                                            │
│         ▼                                                    │
│  ┌──────────────┐                                            │
│  │ 6. CONTINUE  │  Loop back to PROBE                        │
│  │              │  Stop via hook, cron, or convergence       │
│  └──────────────┘                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1: Probe

Launch a subagent swarm to adversarially design tests the harness fails. **This is non-negotiable.** Adversarial probing is the engine of iterative improvement—without it, the loop degenerates into re-running passing tests. Each probe round must actively search for new failures, not validate existing passes.

**Parameters:**
- Concurrency: 5 subagents in parallel
- Target: 5 confirmed failures (not 5 proposals—5 that actually fail)
- Each subagent independently designs and runs tests against the harness

**Distinguish real failures from bad tests.** A failing test is only valuable if the test itself is well-designed. Before counting a failure, verify:

| Check | Bad test signal |
|-------|----------------|
| Is the expected behavior clearly specified? | Vague or ambiguous success criteria |
| Is the test deterministic? | Flaky—passes/fails on identical input |
| Does the test match a real user scenario? | Contrived edge case no user would hit |
| Is the assertion correct? | Test expects wrong answer |
| Is the test self-contained? | Depends on external state or prior tests |

Discard tests that fail these checks. A subagent that produces 5 bad tests and 0 real failures has found nothing.

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

A single fix agent optimizes the harness until all new tests pass. Changes can include:

- Adding information (system prompts, context, examples)
- Designing new tools or modifying existing ones
- Changing the approach (routing, fallback logic, prompting strategy)
- Adjusting parameters (temperature, model, retry logic)

**After fixing, regression-sample existing tests.** Uniformly sample $n$ tests from the full existing suite and run them. Default: $n$ = 20% of total test count. If any sampled test regresses, fix before proceeding.

Why 20%: full suite runs are expensive. 20% catches most regressions while keeping cycle time short. Increase to 50% or 100% if the harness change was structural (new model, rewritten system prompt, changed tool set).

**Fix agent must:**
1. Run each new test individually to confirm it fails pre-fix
2. Make the minimal change that fixes the failure
3. Re-run all new tests to confirm they pass
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

## Step 4: Supervise

Between probe rounds, check pipeline health and manage resources. This step is essential for long-running improvement cycles (hours/days) and optional for short manual sessions.

### Health Check

Classify each pipeline/agent as:
- **RUN** — actively processing (recent tool calls, increasing cost)
- **STUCK** — session exists but idle (rate limit, timeout, or genuinely working on a long task)
- **DEAD** — no session found

**STUCK does not always mean broken.** Check the actual pane/process before restarting. If cost is increasing or the agent shows `(running)`, it's working on a long task—leave it alone. If it shows `Rate limit reached` with $0 cost, it's genuinely stuck.

### Restart Recovery

For genuinely stuck pipelines:
1. Kill the session
2. Apply rate limit recovery (see below)
3. Relaunch

### Rate Limit Recovery

Rate limits are the most common failure mode for LLM-backed pipelines.

**Step 1: Try a different model.** sonnet and opus have independent limits. If sonnet is exhausted, opus often has capacity.

**Step 2: Try a different token.** Round-robin OAuth tokens across pipelines to spread load.

**Step 3: Wait.** If all models and tokens are exhausted, limits reset within 1-2 hours. Don't keep killing and relaunching—each restart wastes cached context.

### Convergence Detection

When a harness reaches 3+ consecutive clean probe rounds (no new failures found within 20 attempts):
1. Check if the hardener is still making changes
2. If no changes needed → notify operator, propose stopping
3. If changes are still happening → let it continue

### Cost Management

- Track running compute (EC2, API spend) and hourly cost
- Propose stopping idle compute when pipelines converge
- Never leave resources running overnight without active pipelines

### Status Reporting

Report **only when something changes**:
- Pipeline state changed (DEAD→RUN, RUN→STUCK)
- A round completed with new results
- Pass rate changed significantly

Don't spam with "no changes" reports.

---

## Step 5: Verify End-to-End

After fixing and pruning, run a full E2E verification pass before declaring the round complete. This is not optional—untested improvements are not improvements.

**Verification must:**
1. Run the full existing E2E test suite (not just the new tests or the regression sample)
2. Test real user journeys through the actual deployed system (not mocked endpoints)
3. Cover all channels/interfaces the harness serves (e.g., for a chatbot: miniapp + KF + admin)
4. Include at least one happy-path and one adversarial scenario per channel
5. Produce evidence (response logs, screenshots, tool call traces, DB state checks)

### How to verify

**Set up as a real user.** Authenticate the same way production users do—self-signed JWTs, BOP tokens, session cookies. Never use mock auth or admin backdoors for E2E. If the harness serves multiple user types (resident, staff, admin), test as each.

**Call the actual endpoints.** Hit the deployed API (not localhost stubs). Use `curl` for API endpoints, browser automation (Playwright MCP, Claude-in-Chrome, or custom scripts) for UI verification. Take screenshots of key UI states as evidence.

**Test as both upstream and downstream user:**
- **Upstream (input):** Act as the person providing input—send messages, submit forms, trigger workflows. Verify the harness accepts, routes, and processes correctly.
- **Downstream (output):** Act as the person receiving the output—check that notifications arrive, dashboards update, work orders appear in the PM queue, handoff reaches the human agent. The fix isn't real until the downstream consumer sees it.

**Read the docs.** Before testing, re-read the relevant API docs, CLAUDE.md sections, and endpoint specifications. Don't test against stale assumptions about what an endpoint does.

**Verification methods by change type:**

| Change type | Verification method |
|-------------|-------------------|
| API/backend | curl endpoints with real auth, check response + DB state |
| UI/frontend | Browser automation (Playwright/Chrome MCP), take screenshots |
| Prompt/LLM | Send test messages, check tool_calls + response quality |
| Tool definition | Trigger the tool via conversation, verify side effects (DB, API calls) |
| Config/routing | Test each route with appropriate user role and permissions |

**If E2E fails:** Do not proceed to the next round. Fix the regression, re-verify, then continue.

**If E2E passes:** Log the verification result in `.hardening-log.jsonl` with `"e2e": "PASS"` and the test count. Only then proceed to the next probe round.

Why this matters: the probe→fix loop optimizes for the test suite, which can drift from real-world behavior. E2E verification anchors each round to production reality. Without it, you can "harden" a harness into a state that passes synthetic tests but breaks real users.

---

## Step 6: Continue

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
# Run one round every hour, auto-expire after 3 days
CronCreate("23 * * * *", "Run one iterative-improvement round")
```

**Cron-mission separation:** The cron prompt should be a thin pointer, not the protocol itself. Keep it to 3-5 lines that reference `mission.md` for the full procedure. This prevents drift between what the scheduler says and what the actual protocol is.

```
# Good: thin cron prompt
"Read mission.md. Execute the hardening protocol. Focus: SWT edge cases, export endpoints."

# Bad: 50+ line cron prompt duplicating mission.md content
"Step 1: Run probe with concurrency 5. Step 2: Fix failures using..."
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

## Progressive Difficulty Testing

Harnesses (especially benchmarks) should be hard enough that frontier models struggle.

1. **Baseline**: Run with the default model. Establish initial pass rates.
2. **Harden until baseline struggles**: Target 30-50% pass rate. If everything passes, cases aren't hard enough.
3. **Validate with stronger model**: Switch to a more capable model. If it passes easily, cases lack genuine difficulty.
4. **Calibrate**: The sweet spot is strong model passing 60-80%, baseline passing 30-50%.

### World Expansion Trigger

When a harness converges (all cases pass on the target model), expand:
- Add new domains or scenarios
- Add harder cases (disambiguation, state reversal, cross-domain reasoning)
- Only expand after at least one full clean pass—don't expand while cases fail due to verifier bugs

---

## Configurable Parameters

| Parameter | Default | When to change |
|-----------|---------|----------------|
| Probe concurrency | 5 | Lower if rate-limited, raise if fast harness |
| Failures to find | 5 | Lower for early rounds, raise for mature suites |
| Regression sample % | 20% | Raise after structural changes |
| Max rounds | unbounded | Set via hook or cron |
| Supervision interval | 20 min | Shorter for critical pipelines, longer for stable ones |

---

## Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Probes find only bad tests | Subagent prompts too vague | Add harness description + existing test examples |
| Fix introduces regressions | Change was too broad | Increase regression sample to 50%, make smaller changes |
| Suite grows without pruning | No pruning step | Run prune after every fix round |
| Pipeline STUCK for hours | Rate limit or long task | Check pane—if cost increasing, wait; if $0, switch model/token |
| Pipeline DEAD | Agent crashed | Relaunch, check if compute host is reachable |
| All tokens rate limited | Subscription-wide limit | Wait 1-2 hours, or add more tokens |
| Convergence without quality | Tests too easy | Raise difficulty, add harder probe scenarios |

---

## Adversarial Probing Best Practices (learned from ChengXing-Bot hardening)

1. **Give the probe agent full context**: existing test suite, system prompt, SQL/DB access, AND live API access. It needs to query the real system, not theorize.
2. **Require the agent to classify failures** using the taxonomy (capability/task_design/verifier_bug) in its output. This saves a classification step later.
3. **Maintain a probe query bank** (`probes.json`): 50+ hard queries categorized by type (aggregation, cross-reference, ambiguity, boundary, negative). The probe agent picks the 10 least-tested rather than generating from scratch.
4. **Require SQL groundtruth verification**: The agent must run the SQL query locally AND query the live API, then compare. This catches both "wrong answer" and "no answer" failures.
5. **Run against all models**: Test each failing query against all configured models to distinguish model-specific vs systemic failures.
6. **20-min agent is too slow for a cron cycle**: For recurring use, build a deterministic `scripts/probe.ts` that auto-generates and runs 5 queries. Reserve the full agent for deep investigation rounds.

---

## Anti-Patterns

- **Don't restart long-running tasks.** A multi-hour agent running complex work is normal, not stuck.
- **Don't spam status reports.** Only report when something changes.
- **Don't keep relaunching on the same rate-limited token.** Each restart wastes cached context.
- **Don't skip the prune step.** Unbounded test suites slow down every future round.
- **Don't count bad tests as failures.** Inflate failure counts without improving the harness.
- **Don't make structural changes to fix edge cases.** Minimal fixes prevent regressions.
- **Don't duplicate protocol in the cron prompt.** The cron/scheduler prompt should NOT contain the full hardening protocol—it should point to a `mission.md` that does. Duplicating protocol detail in the cron creates drift: the cron says one thing, `mission.md` says another, and neither is authoritative. Keep the cron prompt to 3-5 lines: "Read mission.md. Execute the protocol. Key focus: [1-3 bullets]."
- **Don't re-run passing tests and call it hardening.** Each round must spawn a test-finder subagent that actively queries the system with hard adversarial queries and identifies genuine new failures. If a probe round finds zero new failures, that's convergence detection (Step 4)—not a successful hardening round. The core loop is probe→find failures→write tests→fix→verify. Without new failures, there is no improvement.
