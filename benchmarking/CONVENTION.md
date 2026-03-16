# Benchmarking

Step-by-step process for creating and hardening agent benchmarks. The hardening loop (Phase 4) is the centerpiece—everything else serves it.

> **Acknowledgment.** Rubric criteria, CI scripts, and review pipeline adapted from [Terminal-Bench 3](https://github.com/harbor-framework/terminal-bench-3) (Stanford/Laude Institute). See the [TB3 paper](https://arxiv.org/abs/2601.11868).

## Principles

**Simple tasks, rich worlds.** The agent's task is one sentence. Complexity lives in the environment—filesystems, databases, mock APIs, MCP tools, mutable state. Instruction complexity is inversely proportional to world complexity. See [APEX-Agents](https://arxiv.org/abs/2601.14242) (480 tasks, 33 worlds) for the gold standard.

**Outcome verification through iteration.** Grade what the agent produced, not how it got there. Hardening is iterative: run real agents (Claude + Codex), find vulnerabilities, fix them, repeat. Stop only after 3 consecutive clean passes. Analytical hardening alone is insufficient.

**Empirical verification.** Every conclusion needs end-to-end testing. If a test claims to catch reward hacking, prove it by attempting the hack. If the environment claims to be self-contained, build it from scratch and verify.

---

## Infrastructure

All benchmark execution happens in Docker containers on remote hosts. **Never locally.**

```bash
# Launch EC2 instance (c5.2xlarge: 8 vCPU, 16GB RAM, ~$0.34/hr)
bash scripts/bench-ec2-launch.sh c5.2xlarge <benchmark-name>

# Or use Hetzner (root@5.161.107.142)
ssh root@5.161.107.142

# Sync benchmark to remote
rsync -az --delete /path/to/benchmark/ $USER@$IP:/tmp/benchmark/

# Build Docker image
ssh $USER@$IP 'docker build -t benchmark /tmp/benchmark/'
```

**Mandatory requirements:**
- Remote host (EC2 or Hetzner)—never run benchmarks on local machine
- Docker for all agent execution—containers are disposable, one per case
- `rsync --delete` before every build—stale files cause subtle failures
- Always terminate EC2 instances when done

---

## Phase 1: Design

### 1.1 Define the core challenge

Write one sentence describing what the agent must do. If it takes more than one sentence, the task is too complex.

- Good: "Build a fault-tolerant training loop that survives random worker kills and reaches 80% accuracy."
- Bad: "Implement a distributed training system with checkpointing, gradient accumulation, learning rate scheduling, and fault tolerance."

### 1.2 Score against the TB3 rubric

Score your idea against all 13 criteria from [`TASK_IMPLEMENTATION_RUBRIC.toml`](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_IMPLEMENTATION_RUBRIC.toml). Each must be at least "accept." Kill the idea early if criteria 1 (Verifiable), 2 (Well-specified), 3 (Solvable), or 7 (Anti-cheat robust) fail.

### 1.3 Design the world

The environment should feel like a real system. Include at least 2 layers:

| Layer | Examples |
|-------|----------|
| **Filesystem** | Config files, data directories, logs, existing code |
| **Running services** | HTTP servers, databases, message queues |
| **Mock APIs** | REST endpoints, webhooks |
| **MCP tools** | Custom tool servers for domain-specific operations |
| **Databases** | SQLite/PostgreSQL with schema and seed data |
| **State** | Process supervisors, cron jobs, log streams |

All layers are mock/containerized but appear real to the agent. Sanitize the environment: no `.git/`, build logs, stack traces, or artifacts that leak solution information.

### 1.4 Define the verification boundary

- Verifier written before the solution
- Verifier executes outputs (runs the server, queries the DB, measures accuracy)—not source inspection
- Verification is deterministic (tolerance bands or seeds for randomness)
- Verifier runs in under 5 minutes

---

## Phase 2: Build

### 2.1 Harbor task format

```
task/
├── instruction.md          # What the agent sees
├── task.toml               # Metadata (author, category, tags, difficulty, timeout, resources)
├── environment/
│   ├── Dockerfile          # Self-contained world (all deps, data, services)
│   └── data/               # Pre-provided files, configs, mock services
├── solution/
│   └── solve.sh            # Reference solution (proves solvability)
└── tests/
    ├── test.sh             # Entry point: writes reward to /logs/verifier/reward.txt
    └── test_state.py       # Programmatic checks (pytest)
```

Key conventions: `test.sh` writes `1` or `0` to `/logs/verifier/reward.txt` and produces CTRF JSON at `/logs/verifier/ctrf.json`. All test deps pre-installed in Docker. Canary GUID in every task file.

### 2.2 Instruction rules

- All paths absolute (`/app/output/model.pt`, not `model.pt`)
- Single deliverable stated
- Documents what's pre-provided (every file, service, tool)
- Success criterion with a number ("model must achieve >80% accuracy")
- No hints about approach (says what, not how)
- Under 100 lines (if longer, move complexity into the environment)

### 2.3 Verifier rules

- Tests execute behavior, not grep for keywords
- Each test sets up its own state (independent tests)
- Every test has an actionable failure message
- Coverage: core (60%), error handling (30%), edge cases (10%)
- No silent skips (`pytest.skip` or bare `except: pass` are bugs)
- No shallow assertions (`assert resp.status_code == 200` without checking body)

### 2.4 Pin everything

```dockerfile
FROM python:3.12.8-slim           # pin patch version
RUN pip install torch==2.5.1      # pin library versions
```

Base image pinned to patch. All pip/apt packages pinned. No network needed at runtime.

---

## Phase 3: Analytical Hardening

Before running agents, do a manual red-team pass.

For each test, ask: "What's the simplest thing an agent could do to pass this without solving the real problem?"

| Attack | Mitigation |
|--------|------------|
| Hardcode expected outputs | Use varied, unpredictable inputs |
| Download pre-trained model | Disable network (`--network=none`) |
| Fake log entries | Verify temporal consistency, hash integrity |
| Kill/modify test infrastructure | Make infrastructure read-only, verify hashes |
| Read test source for answers | Don't embed answers in tests; verify via execution |
| Monkey-patch libraries | Test in subprocess or verify library integrity |
| Skip hard part, do easy part | Tests check the hard part directly |
| Read `.git/` for clues | Sanitize all environment artifacts |

Add integrity checks for infrastructure files:

```dockerfile
RUN chmod 444 /app/infrastructure.sh
RUN sha256sum /app/infrastructure.sh > /app/.hashes
```

---

## Phase 4: The Hardening Loop

This is the critical phase. Everything before this is preparation; everything after is verification.

### 4.1 Agent setup

Two agent runtimes from different providers. Both use subscription OAuth—no API keys.

**Claude: [`@anthropic-ai/claude-code`](https://www.npmjs.com/package/@anthropic-ai/claude-code) (Claude Code SDK)**

```typescript
import { query } from "@anthropic-ai/claude-code";

const result = await query({
  prompt: "Read /app/instruction.md and complete the task.",
  options: {
    model: "claude-sonnet-4-6",
    maxTurns: 100,
    allowedTools: ["bash", "read", "write", "edit"],
    dangerouslySkipPermissions: true,
  },
});
```

Auth: `CLAUDE_CODE_OAUTH_TOKEN` environment variable (OAuth token from Anthropic Max subscription). Never use `ANTHROPIC_API_KEY`.

**Codex: [`@openai/codex-sdk`](https://www.npmjs.com/package/@openai/codex-sdk) (Codex SDK)**

```typescript
import Codex from "@openai/codex-sdk";

const codex = new Codex();
const thread = codex.startThread();
const result = await thread.run(
  "Read /app/instruction.md and complete the task.",
);
```

Auth: ChatGPT Pro OAuth via `codex login` → `~/.codex/auth.json`. The OpenAI _Agents SDK_ (`api.openai.com/v1/`) is a **separate** system requiring `sk-...` API key—use `@openai/codex-sdk` instead.

**Both agents must have:**
- Latest model specified explicitly
- High effort/reasoning mode
- Standard tools (bash, read, write, edit) + MCP tools if task provides them
- No access to `solution/` or `tests/`
- Fresh container per run (no shared state between trials)

### 4.2 The loop

Run **at least 5 iterations**, stopping only after **3 consecutive clean passes**.

```
┌──────────────────────────────────────────────────┐
│              THE HARDENING LOOP                  │
│                                                  │
│  1. EXECUTE                                      │
│     rsync → build Docker → fresh container       │
│     Run Claude agent → collect pass/fail         │
│     Run Codex agent  → collect pass/fail         │
│                                                  │
│  2. CLASSIFY failures                            │
│     capability   → no action (legitimate)        │
│     task_design  → fix instructions              │
│     verifier_bug → fix verifier/tests            │
│     reward_hack  → harden verifier               │
│     infra        → note for next run             │
│                                                  │
│  3. FIX task_design / verifier_bug / reward_hack │
│     Verify oracle still passes after changes     │
│     Run all 7 CI checks                          │
│     Commit with round number                     │
│                                                  │
│  4. CONVERGE?                                    │
│     clean_count < 3 → loop back to EXECUTE       │
│     clean_count >= 3 → DONE                      │
└──────────────────────────────────────────────────┘
```

### 4.3 Failure taxonomy

| Category | Meaning | Action |
|----------|---------|--------|
| **capability** | Agent couldn't do it | No action—legitimate difficulty |
| **task_design** | Instructions unclear/ambiguous | Fix task description |
| **verifier_bug** | Wrong/strict/loose checks | Fix verifier/tests |
| **reward_hack** | Agent gamed the test | Harden verifier |
| **infra** | Rate limit, Docker crash, timeout | Note for next run |

### 4.4 Verifier hardening rules

From 13 rounds of hardening across 3 benchmarks:

**Free-form text: NEVER exact match.** Use `LIKE '%keyword%'` not `= 'exact string'`. Agents produce reasonable but non-identical wording.

**Name variance: LIKE patterns.** `WHERE name LIKE '%Smith%'` not `WHERE name = 'John Smith'`. Agents use "J. Smith", "Mr. Smith", "john smith".

**MCP action strings: audit the source.** Don't assume action names—grep the server source for what strings it actually logs.

**Underspecified tasks: make deliverables explicit.** If the verifier checks for X but the instruction doesn't mention X, the agent can't know to do it.

### 4.5 Convergence criterion

**Stop after 3 consecutive clean rounds** where:
- No `verifier_bug` or `task_design` fixes needed
- All failures are `capability` or `infra`
- Both Claude and Codex either solve correctly or fail for legitimate reasons
- Oracle passes all tests

| Round | Claude | Codex | Issues | Action |
|-------|--------|-------|--------|--------|
| 1 | Fail (ambiguous) | Fail (missing dep) | 2 | Fixed instruction, added dep |
| 2 | Pass (hack!) | Fail (timeout) | 2 | Added integrity check, increased timeout |
| 3 | Pass (legit) | Pass (legit) | 0 | Clean #1 |
| 4 | Pass (legit) | Pass (legit) | 0 | Clean #2 |
| 5 | Pass (legit) | Pass (legit) | 0 | Clean #3—DONE |

### 4.6 Pipeline automation

For automated execute→harden loops, use `bench-loop.program.ts`:

```bash
# Location: ~/.claude-fleet/programs/bench-loop.program.ts
# Also deployed: /opt/fleet-server/programs/bench-loop.program.ts (Hetzner 5.161.107.142)

# Launch on EC2:
fleet pipeline bench-loop \
  --set benchDir=/path/to/benchmark \
  --set host=$EC2_IP \
  --set sshUser=ec2-user

# Launch on Hetzner:
fleet pipeline bench-loop \
  --set benchDir=/path/to/benchmark \
  --set host=5.161.107.142

# With Codex runtime:
fleet pipeline bench-loop \
  --set benchDir=/path/to/benchmark \
  --set host=$IP \
  --set runtime=codex

# Pause/resume:
touch $SESSION_DIR/paused.flag    # pause after current cycle
rm $SESSION_DIR/paused.flag       # resume
```

The pipeline handles token rotation (5 OAuth accounts), model fallback (sonnet → sonnet[1m] → opus[1m] → opus), round tracking, and stable-pass skipping automatically.

---

## Phase 5: Review

### 5.1 Automated CI checks

Run all 7 checks from `scripts/ci_checks/` before AND after every change:

```bash
for script in scripts/ci_checks/*.sh; do bash "$script" tasks/your-task/; done
```

| Check | What it catches |
|-------|-----------------|
| `check-canary.sh` | Missing contamination markers |
| `check-dockerfile-references.sh` | Solution/test files in Dockerfile |
| `check-dockerfile-sanity.sh` | Unpinned packages, `latest` tags |
| `check-task-absolute-path.sh` | Relative paths in instruction.md |
| `check-test-file-references.sh` | Output files in tests not in instructions |
| `check-test-sh-sanity.sh` | Missing Python environment isolation |
| `validate-task-fields.sh` | Missing metadata in task.toml |

### 5.2 Human signoff (mandatory)

Before the benchmark is finished, a human (not the author) must:

1. **Read instruction.md end-to-end** as if they were the agent. Check: clear without tests/solution? All paths absolute? Success criterion quantified? Under 100 lines?

2. **Walk through solve.sh step by step.** Check: every command correct? Multiple valid approaches pass? Completes within timeout?

Store as `SIGNOFF.md` alongside the task with reviewer name and date.

---

## Resources

See [references/resources.md](references/resources.md) for the full reading list covering benchmark design methodology, construct validity, anti-gaming, and contributing guides.
