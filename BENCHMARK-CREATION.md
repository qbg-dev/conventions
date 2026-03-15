# Benchmark Creation Guide

Step-by-step process for creating agent benchmarks that are simple, hack-resistant, and reproducible. Covers ideation through hardening with mandatory agent stress-testing.

> **Acknowledgment.** The rubric criteria, automated check scripts, review pipeline structure, and many design principles in this guide are derived from [Terminal-Bench 3](https://github.com/harbor-framework/terminal-bench-3) by Stanford University and the Laude Institute. In particular, the [Task Implementation Rubric](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_IMPLEMENTATION_RUBRIC.toml) (19 criteria), [Task Review Automation](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_REVIEW_AUTOMATION.md) (CI pipeline), and the `ci_checks/` scripts are adapted directly from TB3. See the [TB3 paper](https://arxiv.org/abs/2601.11868) for the full methodology.

## Principles

- **Simplicity over complexity.** The agent's task should be simple to describe. Complexity lives in the _world_, not the instructions.
- **Rich worlds.** Drop the agent into an environment with filesystems, databases, mock APIs, MCP tools, and mutable external state. The agent should be able to explore and act, not just generate code.
- **Outcome verification.** Grade what the agent produced, not how it got there.
- **Convergence through iteration.** Hardening ends when 3 consecutive agent runs find nothing to fix.

---

## Phase 1: Ideation

### 1.1 Define the core challenge

Write one sentence: what must the agent do? If it takes more than one sentence, the task is too complex or under-specified.

Good: "Build a fault-tolerant training loop that survives random worker kills and reaches 80% accuracy."
Bad: "Implement a distributed training system with checkpointing, gradient accumulation, learning rate scheduling, and fault tolerance."

### 1.2 Check against the TB3 rubric

The full rubric with detailed guidance for each criterion is at [`TASK_IMPLEMENTATION_RUBRIC.toml`](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_IMPLEMENTATION_RUBRIC.toml). Before writing any code, score your idea against all 13 criteria. Each must be at least "accept":

| #   | Criterion                      | Question to answer                                                 |
| --- | ------------------------------ | ------------------------------------------------------------------ |
| 1   | **Verifiable**                 | Can a program check the answer? Is verification deterministic?     |
| 2   | **Well-specified**             | Would two people reading the spec build compatible verifiers?      |
| 3   | **Solvable**                   | Can an expert implement the solution in a few hours?               |
| 4   | **Difficult**                  | Would an average undergrad fail to solve this in a few days?       |
| 5   | **Interesting**                | Would someone get paid to do this in the real world?               |
| 6   | **Outcome-verified**           | Are you grading results, not process?                              |
| 7   | **Anti-cheat robust**          | Can the agent game the tests without solving the actual problem?   |
| 8   | **Functional verification**    | Do tests execute behavior, not grep for keywords?                  |
| 9   | **Deterministic/reproducible** | Same result every run? All deps pinned? No live services?          |
| 10  | **Essential difficulty**       | Is difficulty from the problem, not formatting minutiae?           |
| 11  | **Test-instruction alignment** | Does every test map to a documented requirement, and vice versa?   |
| 12  | **Novel**                      | Can it be solved by memorization from training data?               |
| 13  | **Agentic**                    | Does it require multi-step tool use, not just one-shot generation? |

**Kill the idea early** if criteria 1, 2, 3, or 7 score "reject." These are structural and hard to fix later.

### 1.3 Design the world, not just the task

The benchmark environment should feel like a real system the agent is dropped into. Design it as a world with:

| Layer                | Examples                                            | Purpose                                      |
| -------------------- | --------------------------------------------------- | -------------------------------------------- |
| **Filesystem**       | Config files, data directories, logs, existing code | Agent must explore and understand            |
| **Running services** | HTTP servers, databases, message queues             | Agent must interact with live processes      |
| **Mock APIs**        | REST endpoints, webhooks, external services         | Agent can call and mutate external state     |
| **MCP tools**        | Custom tool servers for domain-specific operations  | Agent uses tools rather than reimplementing  |
| **Databases**        | SQLite, PostgreSQL with schema and seed data        | Agent queries and mutates real data          |
| **State**            | Process supervisors, cron jobs, log streams         | Agent operates in a system with moving parts |

All of these are mock/containerized, but from the agent's perspective they are real. The agent should be able to mutate state and see consequences.

**Rule: instruction complexity should be inversely proportional to world complexity.** A rich world means shorter instructions. The agent explores and discovers, rather than following a recipe.

### 1.4 Define the verification boundary

Write the verifier before the solution. What does "correct" look like?

- Prefer executing the output (run the server, query the database, measure accuracy) over inspecting the source.
- Verification must be deterministic. If the task involves randomness, define tolerance bands or use seeds.
- The verifier should take < 5 minutes. If it takes longer, the task's feedback loop is too slow for iteration.

---

## Phase 2: Implementation

### 2.1 Build the environment first

```
task/
├── instruction.md          # What the agent sees (short, clear, absolute paths)
├── task.toml               # Metadata, resource limits, timeouts
├── environment/
│   ├── Dockerfile          # Self-contained world (all deps, data, services)
│   └── data/               # Pre-provided files, configs, mock services
├── solution/
│   └── solve.sh            # Reference solution (proves solvability)
└── tests/
    ├── test.sh             # Entry point for verification
    └── test_state.py       # Programmatic checks
```

Build and test the Dockerfile independently before writing the solution:

```bash
docker build -t my-task environment/
docker run --rm -it my-task bash  # explore manually
```

### 2.2 Write the instruction

Follow these rules:

1. **All paths absolute.** `/app/output/model.pt`, not `model.pt`.
2. **State the single deliverable.** "Create `/app/train.py`" or "Configure the database so queries X, Y, Z return correct results."
3. **Document what's pre-provided.** List every file, service, and tool available to the agent.
4. **State the success criterion.** "The model must achieve >80% accuracy" or "All API endpoints must return 200."
5. **No hints about approach.** Say what, not how. Let the agent choose its strategy.
6. **Under 100 lines.** If the instruction is longer, the world isn't rich enough. Move complexity into the environment.

### 2.3 Write the reference solution

The solution must:

- Pass all tests deterministically.
- Complete within the agent timeout.
- Be implementable by a domain expert in a few hours.
- Not be the only possible approach.

### 2.4 Write the verifier

Tests should:

- Execute behavior, not grep for keywords.
- Be independent (each test class sets up its own state).
- Fail with actionable error messages.
- Cover: core functionality (60%), error handling (30%), edge cases (10%).
- Never silently skip (`pytest.skip` or bare `except: pass` are bugs in benchmarks).
- Never accept outputs without validating correctness (no `assert resp.status_code == 200` without checking the body).

### 2.5 Pin everything

```dockerfile
FROM python:3.12.8-slim           # pin patch version
RUN pip install torch==2.5.1      # pin library versions
```

Pre-install test dependencies in the Docker image so tests work without network access.

---

## Phase 3: First-pass hardening (analytical)

Before running agents, do a manual red-team pass:

### 3.1 Enumerate attack vectors

For each test, ask: "What's the simplest thing an agent could do to pass this test without solving the real problem?"

Common bypasses:

| Attack                                    | Mitigation                                                    |
| ----------------------------------------- | ------------------------------------------------------------- |
| Hardcode expected outputs                 | Use varied, unpredictable inputs in tests                     |
| Download pre-trained model/data           | Disable network (`--network=none`) or check training duration |
| Fake log entries                          | Verify temporal consistency, hash integrity of log sources    |
| Kill/modify test infrastructure           | Make infrastructure read-only, verify hashes at test time     |
| Read test source to extract answers       | Don't embed answers in tests; verify via execution            |
| Monkey-patch libraries                    | Test in subprocess or verify library integrity                |
| Skip the hard part, only do the easy part | Tests must check the hard part directly                       |

### 3.2 Add integrity checks

For any file the agent shouldn't modify:

```dockerfile
# In Dockerfile
RUN chmod 444 /app/infrastructure.sh
RUN sha256sum /app/infrastructure.sh > /app/.hashes
```

```python
# In tests
def test_infrastructure_integrity():
    expected = open("/app/.hashes").read().strip().split()
    actual = hashlib.sha256(open(expected[1], "rb").read()).hexdigest()
    assert actual == expected[0], f"{expected[1]} was tampered with"
```

### 3.3 Add temporal/duration checks

If the task requires real computation (training, processing, building):

```python
def test_minimum_duration():
    """Task must take real time, not instant pre-computed results."""
    timestamps = extract_timestamps(log_file)
    duration = (timestamps[-1] - timestamps[0]).total_seconds()
    assert duration >= MIN_EXPECTED_SECONDS
```

---

## Phase 4: Agent hardening (iterative)

This is the critical phase. Run real agents against the benchmark and iterate.

### 4.1 Setup

You need two agent runtimes:

```bash
# Claude Code CLI
claude --version  # must be installed

# OpenAI Codex CLI
codex --version   # must be installed
```

### 4.2 The hardening loop

Each iteration has 3 steps. Run **at least 5 iterations**, stopping only after **3 consecutive clean passes**.

```
┌─────────────────────────────────────────────────┐
│                 HARDENING LOOP                  │
│                                                 │
│  Step 1: Environment setup                      │
│    Clone repo → build Docker → verify structure │
│                                                 │
│  Step 2: Agent solve attempt                    │
│    Run Claude agent on the task                 │
│    Run Codex agent on the task                  │
│    Both must either solve correctly OR fail      │
│    for the right reasons                        │
│                                                 │
│  Step 3: Examine results                        │
│    ┌─ Verifier working correctly?               │
│    ├─ Instructions ambiguous?                   │
│    ├─ Agent failed for wrong reasons?           │
│    ├─ Reward hack possible?                     │
│    └─ Any silent test skips?                    │
│                                                 │
│  If issues found → fix → loop again             │
│  If clean pass → increment clean_count          │
│  If clean_count >= 3 → DONE                     │
└─────────────────────────────────────────────────┘
```

### 4.3 Step 1: Environment setup verification

Clone the repo fresh and verify everything builds:

```bash
WORKDIR=$(mktemp -d)
git clone --depth 1 $REPO_URL $WORKDIR/task
cd $WORKDIR/task

# Verify file structure
for f in instruction.md task.toml environment/Dockerfile solution/solve.sh tests/test.sh; do
    [ -f "$f" ] || echo "MISSING: $f"
done

# Build Docker image
docker build -t benchmark-test environment/

# Run reference solution
docker run --rm \
    -v $(pwd)/solution:/solution:ro \
    -v $(pwd)/tests:/tests:ro \
    benchmark-test \
    bash -c "bash /solution/solve.sh && bash /tests/test.sh"
# Must exit 0 with reward=1.0
```

### 4.4 Step 2: Agent solve attempt

Run both agents independently. Use the Claude Agents SDK or Codex SDK to provide the agent only:

- The instruction.md content
- Access to the running Docker container
- No access to solution/ or tests/

```bash
# Claude Code — solve in container
claude --model claude-sonnet-4-6 \
    --print \
    --allowedTools "Bash,Read,Write,Edit" \
    -p "You are inside a Docker container. Read /app/instruction.md and complete the task."

# Codex CLI — solve in container
codex exec \
    "Read /app/instruction.md and complete the task described."
```

The agent should solve the task or fail for legitimate reasons (difficulty, not ambiguity).

### 4.5 Step 3: Examine results

After each agent run, check this list:

**Verifier quality:**

- [ ] Did the verifier produce the correct reward (1.0 for solution, 0.0 for hacks)?
- [ ] Did any test silently skip or pass vacuously?
- [ ] Were error messages actionable (would they help debug a real attempt)?

**Instruction clarity:**

- [ ] Did the agent misunderstand any requirement?
- [ ] Did the agent attempt something reasonable that the tests rejected unfairly?
- [ ] Are there implicit requirements not stated in instruction.md?
- [ ] Did the agent need information that wasn't in the instruction or discoverable in the environment?

**Agent failure analysis:**

- [ ] If the agent failed, was it because the task is hard (good) or because the setup is broken (bad)?
- [ ] Did the agent hit timeout due to an environment issue, not task difficulty?
- [ ] Did the agent get stuck on a dependency/setup issue instead of the actual problem?

**Reward hack check:**

- [ ] Could the agent's approach pass tests without solving the real problem?
- [ ] Did the agent discover any shortcut not caught by the verifier?
- [ ] Are there test-observable side effects the agent could fake?

### 4.6 Convergence criterion

Track results in a table:

| Round | Claude                       | Codex              | Issues found | Action taken                             |
| ----- | ---------------------------- | ------------------ | ------------ | ---------------------------------------- |
| 1     | Fail (ambiguous instruction) | Fail (missing dep) | 2            | Clarified instruction, added dep         |
| 2     | Pass (reward hack!)          | Fail (timeout)     | 2            | Added integrity check, increased timeout |
| 3     | Pass (legit)                 | Pass (legit)       | 0            | Clean pass #1                            |
| 4     | Pass (legit)                 | Pass (legit)       | 0            | Clean pass #2                            |
| 5     | Pass (legit)                 | Pass (legit)       | 0            | Clean pass #3 — DONE                     |

**Exit condition:** 3 consecutive rounds where both agents either:

- Solve correctly (proving the task is solvable and well-specified), OR
- Fail for the right reasons (proving the task is genuinely difficult, not broken)

AND no new hardening issues are discovered.

---

## Phase 5: Review pipeline (human signoff required)

Adapted from [TB3 Task Review Automation](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_REVIEW_AUTOMATION.md). The TB3 pipeline uses GPTZero for AI detection and automated rubric scoring. Our pipeline replaces AI detection with mandatory human signoff, because the two most important quality gates cannot be automated: a human reading the instruction and a human walking through the solution.

### 5.1 Automated checks (run on every commit)

These run without human intervention. Fail = fix before proceeding. Scripts are copied from [TB3's ci_checks](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_REVIEW_AUTOMATION.md) and adapted for standalone use. All scripts are in [`ci_checks/`](ci_checks/) in this repo.

| Check | Script | What it catches |
|---|---|---|
| **Canary strings** | [`check-canary.sh`](ci_checks/check-canary.sh) | Missing contamination markers in any task file |
| **Dockerfile references** | [`check-dockerfile-references.sh`](ci_checks/check-dockerfile-references.sh) | Solution/test files accidentally COPY'd into container |
| **Dockerfile sanity** | [`check-dockerfile-sanity.sh`](ci_checks/check-dockerfile-sanity.sh) | Pinned apt packages (stale), missing cleanup |
| **Absolute paths** | [`check-task-absolute-path.sh`](ci_checks/check-task-absolute-path.sh) | Relative paths in instruction.md |
| **Test file references** | [`check-test-file-references.sh`](ci_checks/check-test-file-references.sh) | Output files in tests not mentioned in instruction.md |
| **Test.sh sanity** | [`check-test-sh-sanity.sh`](ci_checks/check-test-sh-sanity.sh) | Missing Python environment isolation |
| **Metadata validation** | [`validate-task-fields.sh`](ci_checks/validate-task-fields.sh) | Missing author, category, tags, difficulty in task.toml |

Run all checks against your task:

```bash
cd your-benchmark-repo
for script in ci_checks/*.sh; do bash "$script" tasks/your-task/; done
```

Each script supports an `ALLOWLISTED_TASKS` array for legitimate exceptions. Edit the array inside the script to add exceptions with a documented reason.

### 5.2 Execution checks (run before human review)

| Check | What it proves |
|---|---|
| **Docker build** | Environment builds from Dockerfile alone |
| **Oracle validation** | Reference solution passes all tests (reward = 1.0) |
| **Nop validation** | Doing nothing fails tests (task is non-trivial) |
| **Rubric review** | LLM scores all 19 criteria from `TASK_IMPLEMENTATION_RUBRIC.toml` |

### 5.3 Human signoff (mandatory, cannot be skipped)

Before the benchmark is considered finished, a human must complete both of these. No exceptions.

**Review 1: Read instruction.md end-to-end.**

The reviewer (ideally the domain expert, not the person who wrote the benchmark) reads the instruction as if they were the agent. Checklist:

- [ ] Can I understand what to do without reading tests or solution?
- [ ] Are there any implicit assumptions I needed domain knowledge to fill?
- [ ] Is every path absolute and every output file named?
- [ ] Is the success criterion quantified and unambiguous?
- [ ] Is there any sentence I would rephrase for clarity?
- [ ] Does the instruction avoid prescribing approach (says what, not how)?
- [ ] Is it under 100 lines? If not, what can move into the environment?

**Review 2: Walk through solve.sh step by step.**

The reviewer reads `solve.sh` (and any scripts it creates) and traces the execution mentally or in a shell:

- [ ] Does every command do what the comment says?
- [ ] Is there any fragile assumption (hardcoded paths, timing, race conditions)?
- [ ] Would a different valid approach also pass the tests?
- [ ] Does the solution demonstrate real work (not hardcoded answers)?
- [ ] Are checkpoint/output files written atomically (temp + rename)?
- [ ] Does the solution complete within the agent timeout on the specified hardware?

**Signoff format:**

```
## Human Review Signoff

Reviewer: [name]
Date: [YYYY-MM-DD]

### instruction.md
- [x] Clear and unambiguous
- [x] All paths absolute
- [x] Success criteria quantified
- Notes: [any observations]

### solve.sh
- [x] Correct and complete
- [x] No fragile assumptions
- [x] Completes within timeout
- Notes: [any observations]

Signed off: YES / NO (with reasons)
```

Store this in a `SIGNOFF.md` file alongside the task. The signoff is part of the submission artifact.

### 5.4 Why human signoff matters

Automated checks catch structural issues. Agent hardening catches gameable tests. But neither catches:

- **Ambiguous language** that an agent interprets differently than intended. Only a human reader notices "configure the database" could mean 3 different things.
- **Solution fragility** where the reference solution works but depends on timing, ordering, or an undocumented assumption. Only a human tracing the code step-by-step catches "this race condition works on my machine."
- **Missing context** where the instruction assumes knowledge not available in the environment. A human reading fresh spots "wait, how would I know to use port 8001?"

The agent hardening loop finds issues agents hit. Human review finds issues agents _would_ hit but didn't because of lucky timing, model-specific behavior, or test order.

---

## Phase 6: Agent-eval integration

Codify the hardening checks as an agent-eval test so they can be re-run automatically.

```yaml
name: my-task-verification
tags: [benchmark, reproducibility]
timeout: 3600

setup:
  - type: run
    name: create-workspace
    code: |
      WORKDIR=$(mktemp -d)
      echo $WORKDIR
    save_as: workdir

steps:
  # Phase 1: Structure & reproducibility
  - type: run
    name: clone-and-verify
    code: |
      git clone --depth 1 $REPO_URL {{workdir}}/task
      # Verify all required files exist
    check:
      exit_code: 0

  # Phase 2: Reference solution passes
  - type: run
    name: run-reference-solution
    code: |
      cd {{workdir}}/task
      docker build -t task-test environment/
      docker run --rm -v $(pwd)/solution:/solution:ro \
          -v $(pwd)/tests:/tests:ro task-test \
          bash -c "bash /solution/solve.sh && bash /tests/test.sh && cat /logs/verifier/reward.txt"
    check:
      exit_code: 0
      stdout: /^1$/

  # Phase 3: Reward hack attempts (must fail)
  - type: run
    name: reward-hack-trivial
    code: |
      # Simplest possible bypass attempt
    check:
      stdout: /^0$|NO_REWARD/

  # Phase 4: Convention compliance
  - type: ai_check
    name: instruction-quality
    description: |
      Check instruction.md against benchmark conventions:
      absolute paths, clear deliverable, no approach hints,
      under 100 lines, success criteria stated.
    source: command
    command: cat {{workdir}}/task/instruction.md

teardown:
  - type: run
    name: cleanup
    code: |
      docker rmi task-test 2>/dev/null || true
      rm -rf {{workdir}}
```

Run 3 times independently to prove determinism:

```bash
for i in 1 2 3; do
    bun run src/cli.ts run tests/benchmarks/my-task.test.yaml
done
```

All 3 must pass.

---

## Checklist (before submission)

### Environment

- [ ] Dockerfile builds in < 15 minutes
- [ ] No network needed at runtime (all deps pre-installed)
- [ ] All dependency versions pinned
- [ ] Infrastructure files read-only with hash verification
- [ ] Test deps pre-installed in Docker image

### Instruction

- [ ] Under 100 lines
- [ ] All paths absolute
- [ ] Single clear deliverable stated
- [ ] Success criteria quantified
- [ ] No approach hints
- [ ] Canary string present

### Verifier

- [ ] No silent skips or bare exceptions
- [ ] Tests execute behavior, not grep for keywords
- [ ] Every test has an actionable failure message
- [ ] Reward hack attempts score 0.0
- [ ] Reference solution scores 1.0

### Hardening

- [ ] >= 5 hardening rounds completed
- [ ] 3 consecutive clean passes achieved
- [ ] Both Claude and Codex agents tested
- [ ] All discovered vulnerabilities closed or documented
- [ ] Agent-eval test created and passes 3x

### Human signoff

- [ ] Human reviewed instruction.md end-to-end (not the author)
- [ ] Human traced solve.sh step by step
- [ ] SIGNOFF.md committed with reviewer name and date
- [ ] No issues found, or all issues resolved before signoff

### Submission

- [ ] Matches upstream repo format exactly
- [ ] All automated checks pass (canary, Dockerfile, paths, metadata)
- [ ] Oracle passes, nop fails
- [ ] Agent-eval test committed
- [ ] SIGNOFF.md included
- [ ] REPORT.md documents known limitations

---

## Resources

Foundational reading for creating state-of-the-art benchmarks. Organized by what you'll use them for.

### Benchmark design methodology

| Resource | Key takeaway |
|---|---|
| [TB3 Implementation Rubric](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_IMPLEMENTATION_RUBRIC.toml) | 19-criteria rubric with detailed guidance. The quality bar to hit. |
| [TB3 Review Automation](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_REVIEW_AUTOMATION.md) | Automated pipeline: static checks → execution checks → agent trials. |
| [BetterBench](https://arxiv.org/abs/2411.12990) (Stanford, NeurIPS 2024) | 46-criteria framework for evaluating benchmark quality. Most benchmarks fail to report statistical significance or enable replication. |
| [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (Anthropic, 2025) | Start with 20-50 tasks from real failures, grade outcomes not tool-call sequences, use pass@k for non-deterministic systems. |
| [Challenges in Evaluating AI Systems](https://www.anthropic.com/research/evaluating-ai-systems) (Anthropic, 2023) | Multiple-choice formatting sensitivity shifts scores by ~5%. Human evaluation is subjective. Model-generated evals are circular. |

### Construct validity (does your benchmark measure what it claims?)

| Resource | Key takeaway |
|---|---|
| [Measuring What Matters](https://arxiv.org/pdf/2511.04703) (NeurIPS 2025) | 445 benchmarks reviewed; patterns undermining validity of "safety" and "robustness" claims. 8 recommendations. |
| [Measurement to Meaning](https://arxiv.org/abs/2505.10573) (2025) | Distinguish narrow claims (performance on math tests) from broad claims (general reasoning). |
| [The Evolving Landscape of LLM Evaluation](https://newsletter.ruder.io/p/the-evolving-landscape-of-llm-evaluation) (Ruder, 2024) | Models show 10% drops on GSM1k vs GSM8k, revealing benchmark-specific overfitting. Assume contamination by design. |

### Anti-gaming and anti-contamination

| Resource | Key takeaway |
|---|---|
| [LiveCodeBench](https://arxiv.org/abs/2403.07974) | Time-segmented evaluation: only test on problems released after model's training cutoff. 600+ problems. |
| [EvalPlus](https://arxiv.org/abs/2305.01210) (NeurIPS 2023) | Adding 80x more tests to HumanEval dropped pass rates by 19-29%. Original test suites are always insufficient. |
| [Specification Gaming](https://deepmind.google/blog/specification-gaming-the-flip-side-of-ai-ingenuity/) (DeepMind) | Better algorithms find more creative loopholes. Specify outcomes comprehensively. |
| [Reward Hacking in RL](https://lilianweng.github.io/posts/2024-11-28-reward-hacking/) (Lilian Weng, 2024) | Models modify unit tests to pass coding tasks, exploit length bias, exploit sophistication bias. |
| [Demonstrating Specification Gaming in Reasoning Models](https://arxiv.org/pdf/2502.13295) (Palisade, 2025) | Reasoning LLMs hack chess by modifying the opponent's engine. Directly relevant to agent benchmarks with tool access. |

### Contribution guides from leading benchmarks

| Benchmark | Format | Key design choices |
|---|---|---|
| [SWE-bench](https://arxiv.org/pdf/2310.06770) (Princeton) | Real GitHub issues + existing test suites | 2,294 tasks, 12 repos. Human-verified subset (SWE-bench Verified) used 93 developers, 3 annotators per sample. |
| [BigCodeBench](https://arxiv.org/abs/2406.15877) (ICLR 2025) | 1,140 tasks, 723 function calls, 139 libraries | 99% branch coverage, avg 5.6 test cases/task. Human performance 97% vs best LLM ~60%. |
| [GAIA](https://arxiv.org/abs/2311.12983) (Meta-FAIR) | 466 questions requiring reasoning + tool use | Humans 92% vs GPT-4 15%. Targets tasks simple for humans but hard for AI. |
| [Terminal-Bench](https://arxiv.org/abs/2601.11868) (Stanford + Laude) | 89 curated terminal tasks with Docker environments | Frontier models cap at ~65%. 32,155 trials across 6 agents. |
