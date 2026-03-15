# Benchmark Creation Guide

Step-by-step process for creating agent benchmarks that are simple, hack-resistant, and reproducible. Covers ideation through hardening with mandatory agent stress-testing.

> **Acknowledgment.** The rubric criteria, automated check scripts, review pipeline structure, and many design principles in this guide are derived from [Terminal-Bench 3](https://github.com/harbor-framework/terminal-bench-3) by Stanford University and the Laude Institute. In particular, the [Task Implementation Rubric](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_IMPLEMENTATION_RUBRIC.toml) (19 criteria), [Task Review Automation](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_REVIEW_AUTOMATION.md) (CI pipeline), and the `ci_checks/` scripts are adapted directly from TB3. See the [TB3 paper](https://arxiv.org/abs/2601.11868) for the full methodology.

## Principles

**Simplicity and rich worlds.** The agent's task should be simple to describe—one sentence. Complexity lives in the _world_, not the instructions. Drop the agent into an environment with filesystems, databases, mock APIs, MCP tools, and mutable external state. The agent explores and acts, not just generates code. Instruction complexity should be inversely proportional to world complexity. Read [APEX-Agents](https://arxiv.org/abs/2601.14242) (480 tasks, 33 "worlds") and the [Archipelago](https://github.com/Mercor-Intelligence/archipelago) evaluation framework for the gold standard on world design.

**Outcome verification through iteration.** Grade what the agent produced, not how it got there. Hardening is iterative: run real agents (Claude + Codex), find vulnerabilities, fix them, repeat. Stop only after 3 consecutive clean passes from both agents. This convergence criterion is non-negotiable—analytical hardening alone is insufficient.

**Empirical verification.** Every conclusion must be accompanied by end-to-end testing. Mocks do not work, because benchmark validity is the first-order concern. Nothing else matters as much as validity. If a test claims to catch reward hacking, prove it by attempting the hack. If the environment claims to be self-contained, build it from scratch and verify.

---

## Phase 1: Ideation

### 1.1 Define the core challenge

- [ ] Write one sentence describing what the agent must do
- [ ] If it takes more than one sentence, the task is too complex or under-specified

Good: "Build a fault-tolerant training loop that survives random worker kills and reaches 80% accuracy."
Bad: "Implement a distributed training system with checkpointing, gradient accumulation, learning rate scheduling, and fault tolerance."

### 1.2 Score against the TB3 rubric

The full rubric with detailed guidance is at [`TASK_IMPLEMENTATION_RUBRIC.toml`](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_IMPLEMENTATION_RUBRIC.toml). Score your idea against all 13 criteria. Each must be at least "accept":

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

- [ ] All 13 criteria scored
- [ ] Criteria 1, 2, 3, and 7 are NOT "reject" (these are structural and hard to fix later—kill the idea early if any fail)

### 1.3 Design the world

The benchmark environment should feel like a real system. Design it as a world:

| Layer                | Examples                                            | Purpose                                      |
| -------------------- | --------------------------------------------------- | -------------------------------------------- |
| **Filesystem**       | Config files, data directories, logs, existing code | Agent must explore and understand             |
| **Running services** | HTTP servers, databases, message queues             | Agent must interact with live processes       |
| **Mock APIs**        | REST endpoints, webhooks, external services         | Agent can call and mutate external state      |
| **MCP tools**        | Custom tool servers for domain-specific operations  | Agent uses tools rather than reimplementing   |
| **Databases**        | SQLite, PostgreSQL with schema and seed data        | Agent queries and mutates real data           |
| **State**            | Process supervisors, cron jobs, log streams         | Agent operates in a system with moving parts  |

All layers are mock/containerized, but from the agent's perspective they are real. The agent mutates state and sees consequences.

- [ ] World has at least 2 layers from the table above
- [ ] Environment is sanitized: no git history, commit messages, build logs, or other artifacts that leak solution information (agents will read anything available—`.git/`, build outputs, log files with stack traces)
- [ ] Instruction complexity is inversely proportional to world complexity

### 1.4 Define the verification boundary

- [ ] Verifier written before the solution
- [ ] Verifier executes outputs (runs the server, queries the database, measures accuracy) rather than inspecting source
- [ ] Verification is deterministic (if randomness is involved, define tolerance bands or use seeds)
- [ ] Verifier runs in under 5 minutes

---

## Phase 2: Implementation

### 2.1 Build the environment

Use the [Harbor task format](https://harborframework.com/docs/tasks) exactly:

```
task/
├── instruction.md          # What the agent sees
├── task.toml               # Metadata (see validate-task-fields.sh for required fields)
├── environment/
│   ├── Dockerfile          # Self-contained world (all deps, data, services)
│   └── data/               # Pre-provided files, configs, mock services
├── solution/
│   └── solve.sh            # Reference solution (proves solvability)
└── tests/
    ├── test.sh             # Entry point: runs tests, writes reward to /logs/verifier/reward.txt
    └── test_state.py       # Programmatic checks (pytest)
```

Key conventions from the Harbor format:
- `task.toml` must include: `author`, `category`, `tags`, `difficulty`, `timeout`, `resources`
- `test.sh` must write `1` or `0` to `/logs/verifier/reward.txt`
- `test.sh` must produce CTRF-format JSON at `/logs/verifier/ctrf.json`
- All test dependencies (pytest, etc.) must be pre-installed in the Docker image
- A canary GUID must appear in every task file (instruction.md, test.sh, test_state.py, Dockerfile)

Verification:

```bash
docker build -t my-task environment/
docker run --rm -it my-task bash  # explore manually, verify world state
```

- [ ] Dockerfile builds successfully
- [ ] Container starts with all services running
- [ ] No network access needed at runtime
- [ ] Canary GUID present in all task files

### 2.2 Write the instruction

- [ ] All paths absolute (`/app/output/model.pt`, not `model.pt`)
- [ ] Single deliverable stated ("Create `/app/train.py`" or "Configure the database so queries X, Y, Z return correct results")
- [ ] Documents what's pre-provided (every file, service, and tool available)
- [ ] States success criterion with a number ("model must achieve >80% accuracy", "all API endpoints must return 200")
- [ ] No hints about approach (says what, not how)
- [ ] Under 100 lines (if longer, the world isn't rich enough—move complexity into the environment)

### 2.3 Write the reference solution

- [ ] Passes all tests deterministically
- [ ] Completes within the agent timeout
- [ ] Implementable by a domain expert in a few hours
- [ ] Is not the only possible approach

### 2.4 Write the verifier

- [ ] Tests execute behavior, not grep for keywords
- [ ] Each test class sets up its own state (tests are independent)
- [ ] Every test has an actionable failure message explaining what went wrong and what was expected
- [ ] Coverage: core functionality (60%), error handling (30%), edge cases (10%)
- [ ] No silent skips (`pytest.skip` or bare `except: pass` are bugs in benchmarks)
- [ ] No shallow assertions (`assert resp.status_code == 200` without checking the body)

### 2.5 Pin everything

```dockerfile
FROM python:3.12.8-slim           # pin patch version
RUN pip install torch==2.5.1      # pin library versions
```

- [ ] Base image pinned to patch version
- [ ] All pip/apt packages pinned
- [ ] Test dependencies pre-installed in Docker image (no network needed for tests)

---

## Phase 3: Analytical hardening

Before running agents, do a manual red-team pass.

### 3.1 Enumerate attack vectors

For each test, ask: "What's the simplest thing an agent could do to pass this test without solving the real problem?"

| Attack                                    | Mitigation                                                    |
| ----------------------------------------- | ------------------------------------------------------------- |
| Hardcode expected outputs                 | Use varied, unpredictable inputs in tests                     |
| Download pre-trained model/data           | Disable network (`--network=none`) or check training duration |
| Fake log entries                          | Verify temporal consistency, hash integrity of log sources    |
| Kill/modify test infrastructure           | Make infrastructure read-only, verify hashes at test time     |
| Read test source to extract answers       | Don't embed answers in tests; verify via execution            |
| Monkey-patch libraries                    | Test in subprocess or verify library integrity                |
| Skip the hard part, only do the easy part | Tests must check the hard part directly                       |
| Read environment artifacts for clues      | Sanitize `.git/`, build logs, stack traces, temp files        |

- [ ] Every test has a corresponding "cheapest bypass" identified
- [ ] Every bypass has a mitigation implemented or documented as accepted risk

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

- [ ] All infrastructure files (scripts, configs the agent shouldn't touch) are read-only (chmod 444)
- [ ] SHA-256 hashes computed at build time and verified at test time

### 3.3 Add temporal/duration checks

If the task requires real computation (training, processing, building):

```python
def test_minimum_duration():
    """Task must take real time, not instant pre-computed results."""
    timestamps = extract_timestamps(log_file)
    duration = (timestamps[-1] - timestamps[0]).total_seconds()
    assert duration >= MIN_EXPECTED_SECONDS
```

- [ ] Minimum wall-clock duration enforced if applicable
- [ ] Timestamps verified for temporal consistency (not fabricated)

---

## Phase 4: Agent hardening (iterative)

This is the critical phase. Run real agents against the benchmark and iterate.

### 4.1 Setup

You need two agent runtimes from different providers to stress-test from different angles.

**Claude Code CLI** (recommended for Claude—uses existing auth):

```bash
# Run inside the Docker container's mounted workspace
claude -p "You are inside a Docker container. Read /app/instruction.md and complete the task." \
  --model claude-sonnet-4-6 \
  --allowedTools bash,read,write,edit \
  --max-turns 100
```

Or programmatically via the **Claude Agents SDK** ([docs](https://docs.anthropic.com/en/docs/agents-sdk)):

```python
from claude_agent_sdk import Agent, Task

agent = Agent(
    model="claude-sonnet-4-6",
    tools=["bash", "read", "write", "edit"],
    max_tokens=16384,
)
result = agent.run(Task(
    prompt="You are inside a Docker container. Read /app/instruction.md and complete the task.",
    working_directory="/app",
))
```

**Codex CLI** (recommended for OpenAI—uses existing ChatGPT OAuth):

```bash
# Codex CLI uses ChatGPT subscription auth (no API key needed)
# Auth: `codex login` → OAuth device flow → stored in ~/.codex/auth.json
codex exec "Read /app/instruction.md and complete the task." \
  --model gpt-5.4 \
  --writable-root /app
```

> **Auth note:** Codex CLI's ChatGPT OAuth (`chatgpt.com/backend-api/codex/`) and the OpenAI Agents SDK (`api.openai.com/v1/`) use different endpoints and billing. Codex CLI works with your ChatGPT Pro subscription. The Agents SDK requires a separate `sk-...` API key with pay-per-token billing. Use the CLI for hardening unless you have an API key.

If you have an OpenAI API key, the **OpenAI Agents SDK** ([docs](https://openai.github.io/openai-agents-python/)) is also an option:

```python
from agents import Agent, Runner

agent = Agent(
    name="benchmark-solver",
    model="gpt-5.4",
    instructions="You are inside a Docker container. Read /app/instruction.md and complete the task.",
)
result = Runner.run_sync(agent)  # requires OPENAI_API_KEY env var
```

Configure both with:
- [ ] Latest available model specified explicitly
- [ ] High effort/reasoning mode enabled
- [ ] Standard tool set (bash, read, write, edit)
- [ ] MCP server tools if the task provides them
- [ ] No access to `solution/` or `tests/` directories
- [ ] Each run starts from a clean environment (no shared state between trials—see [Anthropic's eval guidance](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents))

### 4.2 The hardening loop

Run **at least 5 iterations**, stopping only after **3 consecutive clean passes**.

```
┌─────────────────────────────────────────────────┐
│                 HARDENING LOOP                  │
│                                                 │
│  Step 1: Environment setup                      │
│    Clone fresh → build Docker → verify structure│
│                                                 │
│  Step 2: Agent solve attempt                    │
│    Run Claude agent on the task                 │
│    Run Codex agent on the task                  │
│    Both must either solve correctly OR fail     │
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

```bash
WORKDIR=$(mktemp -d)
git clone --depth 1 $REPO_URL $WORKDIR/task
cd $WORKDIR/task

# Verify all required files exist
for f in instruction.md task.toml environment/Dockerfile solution/solve.sh tests/test.sh; do
    [ -f "$f" ] || echo "MISSING: $f"
done

# Build and run reference solution
docker build -t benchmark-test environment/
docker run --rm \
    -v $(pwd)/solution:/solution:ro \
    -v $(pwd)/tests:/tests:ro \
    benchmark-test \
    bash -c "bash /solution/solve.sh && bash /tests/test.sh"
# Must exit 0 with reward=1.0
```

- [ ] Fresh clone builds successfully
- [ ] Reference solution passes all tests (reward = 1.0)

### 4.4 Step 2: Agent solve attempt

Run both agents independently. Provide only:
- The instruction.md content
- Access to the running Docker container
- No access to solution/ or tests/

- [ ] Claude agent ran to completion
- [ ] Codex agent ran to completion
- [ ] Agent solved correctly OR failed for legitimate reasons (difficulty, not ambiguity)

### 4.5 Step 3: Examine results

After each agent run, check:

**Verifier quality:**
- [ ] Verifier produced correct reward (1.0 for solution, 0.0 for hacks)
- [ ] No test silently skipped or passed vacuously
- [ ] Error messages are actionable (would they help debug a real attempt?)

**Instruction clarity** (distinguish good failures from bad—see [Mechanize](https://www.mechanize.work/what-working-here-is-like/)):

Good failures (real capability gaps):
- Agent failed to follow existing codebase conventions when implementing a new feature
- Agent failed to proactively communicate a crucial design decision to stakeholders
- Agent failed to reuse a function it defined earlier (code erosion over checkpoints)

Bad failures (task design bugs—fix these):
- Grader checked for a specific query parameter name the agent had no way of knowing
- Agent interpreted "push to main" as forking then pushing, grader only checked origin/main → score 0
- Tests require a specific approach when multiple valid approaches exist

- [ ] Agent did not misunderstand any requirement
- [ ] Agent did not attempt something reasonable that tests rejected unfairly
- [ ] No implicit requirements unstated in instruction.md
- [ ] Agent did not need information unavailable in the instruction or environment
- [ ] If the agent failed, it was a _good_ failure (capability gap), not a _bad_ failure (task design bug)

**Failure analysis:**
- [ ] If agent failed, it was because the task is hard (good), not because setup is broken (bad)
- [ ] Agent did not hit timeout due to environment issue
- [ ] Agent did not get stuck on dependency/setup instead of the actual problem

**Reward hack check:**
- [ ] Agent's approach cannot pass tests without solving the real problem
- [ ] Agent did not discover shortcuts not caught by verifier
- [ ] No test-observable side effects the agent could fake

### 4.6 Convergence criterion

Track results:

| Round | Claude                       | Codex              | Issues found | Action taken                             |
| ----- | ---------------------------- | ------------------ | ------------ | ---------------------------------------- |
| 1     | Fail (ambiguous instruction) | Fail (missing dep) | 2            | Clarified instruction, added dep         |
| 2     | Pass (reward hack!)          | Fail (timeout)     | 2            | Added integrity check, increased timeout |
| 3     | Pass (legit)                 | Pass (legit)       | 0            | Clean pass #1                            |
| 4     | Pass (legit)                 | Pass (legit)       | 0            | Clean pass #2                            |
| 5     | Pass (legit)                 | Pass (legit)       | 0            | Clean pass #3—DONE                       |

**Exit condition:** 3 consecutive rounds where both agents either solve correctly (proving solvability) or fail for the right reasons (proving genuine difficulty), AND no new hardening issues are discovered.

---

## Phase 5: Review pipeline (human signoff required)

Adapted from [TB3 Task Review Automation](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_REVIEW_AUTOMATION.md). Our pipeline replaces AI detection with mandatory human signoff, because the two most important quality gates cannot be automated: a human reading the instruction and a human walking through the solution.

### 5.1 Automated checks (run on every commit)

Fail = fix before proceeding. Scripts adapted from [TB3's ci_checks](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_REVIEW_AUTOMATION.md). All scripts in [`ci_checks/`](ci_checks/) in this repo.

| Check | Script | What it catches |
|---|---|---|
| **Canary strings** | [`check-canary.sh`](ci_checks/check-canary.sh) | Missing contamination markers in any task file |
| **Dockerfile references** | [`check-dockerfile-references.sh`](ci_checks/check-dockerfile-references.sh) | Solution/test files accidentally COPY'd into container |
| **Dockerfile sanity** | [`check-dockerfile-sanity.sh`](ci_checks/check-dockerfile-sanity.sh) | Pinned apt packages (stale), missing cleanup |
| **Absolute paths** | [`check-task-absolute-path.sh`](ci_checks/check-task-absolute-path.sh) | Relative paths in instruction.md |
| **Test file references** | [`check-test-file-references.sh`](ci_checks/check-test-file-references.sh) | Output files in tests not mentioned in instruction.md |
| **Test.sh sanity** | [`check-test-sh-sanity.sh`](ci_checks/check-test-sh-sanity.sh) | Missing Python environment isolation |
| **Metadata validation** | [`validate-task-fields.sh`](ci_checks/validate-task-fields.sh) | Missing author, category, tags, difficulty in task.toml |

```bash
# Run all checks against your task
cd your-benchmark-repo
for script in ci_checks/*.sh; do bash "$script" tasks/your-task/; done
```

- [ ] All 7 automated checks pass
- [ ] Any exceptions added to `ALLOWLISTED_TASKS` with documented reasons

### 5.2 Execution checks (run before human review)

- [ ] **Docker build**: Environment builds from Dockerfile alone
- [ ] **Oracle validation**: Reference solution passes all tests (reward = 1.0)
- [ ] **Nop validation**: Doing nothing fails tests (task is non-trivial)
- [ ] **Rubric review**: All 13 criteria from `TASK_IMPLEMENTATION_RUBRIC.toml` scored

### 5.3 Human signoff (mandatory, cannot be skipped)

Before the benchmark is considered finished, a human must complete both reviews. No exceptions.

**Review 1: Read instruction.md end-to-end.** The reviewer (ideally not the author) reads the instruction as if they were the agent.

- [ ] Can I understand what to do without reading tests or solution?
- [ ] Are there any implicit assumptions requiring domain knowledge to fill?
- [ ] Is every path absolute and every output file named?
- [ ] Is the success criterion quantified and unambiguous?
- [ ] Is there any sentence I would rephrase for clarity?
- [ ] Does the instruction avoid prescribing approach (says what, not how)?
- [ ] Is it under 100 lines? If not, what can move into the environment?

**Review 2: Walk through solve.sh step by step.** The reviewer traces execution mentally or in a shell.

- [ ] Does every command do what the comment says?
- [ ] Is there any fragile assumption (hardcoded paths, timing, race conditions)?
- [ ] Would a different valid approach also pass the tests?
- [ ] Does the solution demonstrate real work (not hardcoded answers)?
- [ ] Are checkpoint/output files written atomically (temp + rename)?
- [ ] Does the solution complete within the agent timeout on the specified hardware?

**Why human signoff matters:** Automated checks catch structural issues. Agent hardening catches gameable tests. Neither catches ambiguous language an agent interprets differently than intended, solution fragility depending on timing or undocumented assumptions, or missing context where the instruction assumes knowledge not in the environment.

**Signoff format** (store as `SIGNOFF.md` alongside the task):

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

---

## Phase 6: Agent-eval integration

Codify the hardening checks as an agent-eval test for automated re-runs.

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
  # Structure & reproducibility
  - type: run
    name: clone-and-verify
    code: |
      git clone --depth 1 $REPO_URL {{workdir}}/task
    check:
      exit_code: 0

  # Reference solution passes
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

  # Reward hack attempt (must fail)
  - type: run
    name: reward-hack-trivial
    code: |
      # Simplest possible bypass attempt
    check:
      stdout: /^0$|NO_REWARD/

  # Convention compliance
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

- [ ] Agent-eval test runs 3 times independently
- [ ] All 3 runs pass (proving determinism)

---

## Final checklist (before submission)

### Environment
- [ ] Dockerfile builds in < 15 minutes
- [ ] No network needed at runtime (all deps pre-installed)
- [ ] All dependency versions pinned
- [ ] Infrastructure files read-only with hash verification
- [ ] Test deps pre-installed in Docker image
- [ ] Environment sanitized (no `.git/`, build logs, or artifacts leaking solution info)

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
- [ ] Both Claude and Codex agents tested (via SDKs)
- [ ] All discovered vulnerabilities closed or documented
- [ ] Agent-eval test created and passes 3x

### Human signoff
- [ ] Human reviewed instruction.md end-to-end (not the author)
- [ ] Human traced solve.sh step by step
- [ ] SIGNOFF.md committed with reviewer name and date
- [ ] No issues found, or all issues resolved before signoff

### Submission
- [ ] Matches upstream repo format exactly (see contributing guides below)
- [ ] All 7 automated checks pass (canary, Dockerfile, paths, metadata)
- [ ] Oracle passes, nop fails
- [ ] Agent-eval test committed
- [ ] SIGNOFF.md included
- [ ] REPORT.md documents known limitations

---

## Contributing guides

When submitting to existing benchmarks, follow their specific format:

| Benchmark | Contributing guide | Format |
|---|---|---|
| **Terminal-Bench 3** | [TB3 Contribution Call](https://www.tbench.ai/news/tb3-contribution-call) | [Harbor task format](https://harborframework.com/docs/tasks) |
| **SlopCodeBench** | [SCBench Contributing](https://github.com/SprocketLab/slop-code-bench/tree/main/docs/contributing-problems) | Checkpoint-based config.yaml |
| **METR** | [METR Task Standard](https://github.com/METR/task-standard) | METR task format |

---

## Resources

Foundational reading for state-of-the-art benchmarks. Organized by use case.

### Benchmark design methodology

| Resource | Key takeaway |
|---|---|
| [TB3 Implementation Rubric](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_IMPLEMENTATION_RUBRIC.toml) | 19-criteria rubric with detailed guidance. The quality bar. |
| [TB3 Review Automation](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_REVIEW_AUTOMATION.md) | Automated pipeline: static checks → execution checks → agent trials. |
| [APEX-Agents](https://arxiv.org/abs/2601.14242) (Mercor) | 480 tasks across 33 "worlds." Professional experts create scenarios, define grading. Gold standard for world design. |
| [Archipelago](https://github.com/Mercor-Intelligence/archipelago) (Mercor) | Open-source Docker sandbox harness for running agent evaluations inside APEX worlds. |
| [BetterBench](https://arxiv.org/abs/2411.12990) (Stanford, NeurIPS 2024) | 46-criteria framework. Most benchmarks fail to report statistical significance or enable replication. |
| [Demystifying Evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (Anthropic, 2025) | Start with 20-50 tasks from real failures, grade outcomes not tool-call sequences, use pass@k. |
| [Challenges in Evaluating AI Systems](https://www.anthropic.com/research/evaluating-ai-systems) (Anthropic, 2023) | Multiple-choice formatting sensitivity shifts scores ~5%. Model-generated evals are circular. |

### Construct validity (does your benchmark measure what it claims?)

| Resource | Key takeaway |
|---|---|
| [Measuring What Matters](https://arxiv.org/pdf/2511.04703) (NeurIPS 2025) | 445 benchmarks reviewed; 8 recommendations for valid measurement. |
| [Measurement to Meaning](https://arxiv.org/abs/2505.10573) (2025) | Distinguish narrow claims (math test scores) from broad claims (general reasoning). |
| [The Evolving Landscape of LLM Evaluation](https://newsletter.ruder.io/p/the-evolving-landscape-of-llm-evaluation) (Ruder, 2024) | 10% drops on GSM1k vs GSM8k reveal benchmark-specific overfitting. Assume contamination. |

### Anti-gaming and anti-contamination

| Resource | Key takeaway |
|---|---|
| [LiveCodeBench](https://arxiv.org/abs/2403.07974) | Time-segmented evaluation: only test on problems released after training cutoff. |
| [EvalPlus](https://arxiv.org/abs/2305.01210) (NeurIPS 2023) | Adding 80x more tests dropped pass rates 19-29%. Original test suites are always insufficient. |
| [Specification Gaming](https://deepmind.google/blog/specification-gaming-the-flip-side-of-ai-ingenuity/) (DeepMind) | Better algorithms find more creative loopholes. Specify outcomes comprehensively. |
| [Reward Hacking in RL](https://lilianweng.github.io/posts/2024-11-28-reward-hacking/) (Weng, 2024) | Models modify unit tests, exploit length bias, exploit sophistication bias. |
| [Spec Gaming in Reasoning Models](https://arxiv.org/pdf/2502.13295) (Palisade, 2025) | Reasoning LLMs hack chess by modifying opponent's engine. Directly relevant to agent benchmarks. |

### Leading benchmark contribution guides

| Benchmark | Format | Key design choices |
|---|---|---|
| [SWE-bench](https://arxiv.org/pdf/2310.06770) (Princeton) | Real GitHub issues + existing test suites | 2,294 tasks, 12 repos. Human-verified subset used 93 developers, 3 annotators/sample. |
| [BigCodeBench](https://arxiv.org/abs/2406.15877) (ICLR 2025) | 1,140 tasks, 723 function calls, 139 libraries | 99% branch coverage, avg 5.6 test cases/task. |
| [GAIA](https://arxiv.org/abs/2311.12983) (Meta-FAIR) | 466 questions requiring reasoning + tool use | Humans 92% vs GPT-4 15%. Simple for humans, hard for AI. |
| [Terminal-Bench](https://arxiv.org/abs/2601.11868) (Stanford + Laude) | 89 curated terminal tasks with Docker environments | Frontier models cap at ~65%. 32,155 trials across 6 agents. |
