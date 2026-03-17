# Bench Supervision

How to operate a persistent agent that monitors and maintains benchmark pipelines. The supervisor runs on a cron tick (default 20 min), checks pipeline health, restarts failures, and reports status changes. It is the operational layer that sits above the benchmarking and iterative-hardening conventions.

## When This Applies

- You are a fleet worker with a `mission.md` that includes a cron checklist
- You are monitoring `bench-loop` pipelines running on EC2 or remote hosts
- You need to restart a stuck/dead benchmark pipeline
- You are diagnosing rate limits, model availability, or token exhaustion

---

## Architecture

```
┌─────────────────────────┐
│  bench-supervisor agent  │  ← cron tick every 20 min
│  (fleet worker + tmux)   │
└──────────┬──────────────┘
           │ monitors via
           │ bench-monitor.sh
           ▼
┌──────────────────────────────────────┐
│  bench-loop pipelines (tmux sessions) │
│  bench-greenfield  bench-slop  ...    │
│  Each: executor → hardener → loop     │
└──────────┬───────────────────────────┘
           │ run on
           ▼
┌──────────────────────┐
│  EC2 instances        │
│  Docker containers    │
│  (one per benchmark)  │
└───────────────────────┘
```

The supervisor does NOT run benchmarks directly. It monitors the pipelines that do.

---

## Cron Tick Checklist

Execute this checklist on every tick. The mission.md file is the source of truth for pipeline-specific commands—re-read it first.

### 1. Health Check

```bash
bash scripts/bench-monitor.sh --notify
```

The monitor reports each pipeline as one of:
- **RUN** — agent is actively processing (cost > $0, recent tool calls)
- **STUCK** — tmux session exists but agent is idle (rate limit, timeout, or long-running task)
- **DEAD** — no tmux session found

**STUCK does not always mean broken.** Check the executor pane before restarting:

```bash
tmux capture-pane -t bench-<name>:execute -p 2>&1 | tail -15
```

If the pane shows `(running)` with increasing cost, the agent is working on a long task. Leave it alone. If it shows `API Error: Rate limit reached` with $0.0000 cost, it's genuinely stuck.

### 2. Restart Dead/Stuck Pipelines

For genuinely stuck pipelines (rate limited, crashed, idle prompt):

1. Kill the session: `tmux kill-session -t bench-<name>`
2. Apply rate limit recovery (see below)
3. Relaunch from mission.md commands

For DEAD pipelines, also verify EC2 exists first:

```bash
ssh -o ConnectTimeout=5 ec2-user@<IP> 'uptime && docker ps'
```

If EC2 is unreachable, launch a new one before restarting the pipeline.

### 3. Report Status Changes

Send a Fleet Mail report **only if something changed**:
- Pipeline state changed (DEAD→RUN, RUN→STUCK)
- A round completed (new round-results.json)
- Pass rate changed

Don't spam the user with "no changes" reports.

### 4. Examine Convergence

When a benchmark reaches 3+ consecutive clean passes:
1. Check `stable-passes.json` for the count
2. Check `git diff` in the bench directory for hardener changes
3. If no changes needed → notify user, ask whether to stop
4. If hardener is still making changes → let it continue

### 5. Cost Management

- Track running EC2 instances and their hourly cost
- Propose stopping idle EC2 when all pipelines on that host converge
- Never leave EC2 running overnight without active pipelines

---

## Rate Limit Recovery

Rate limits are the most common failure mode. Recovery order:

### Step 1: Try a different model

```bash
fleet pipeline bench-loop --set 'model=opus[1m]' ...
```

sonnet[1m] and opus[1m] have independent rate limits. If sonnet is exhausted, opus often has capacity. Quote the brackets for zsh.

### Step 2: Try a different OAuth token

```bash
TOKEN=$(sed -n '3p' ~/.claude/sensitive/oauth-tokens.md | grep 'sk-ant')
export CLAUDE_CODE_OAUTH_TOKEN="$TOKEN"
fleet pipeline bench-loop ...
```

Tokens are in `~/.claude/sensitive/oauth-tokens.md`. Round-robin by pipeline to spread load.

### Step 3: Wait

If all models and tokens are exhausted, the limit will reset within 1-2 hours. Don't keep killing and relaunching—each restart wastes the agent's cached context.

### Adding More Tokens

Append new `sk-ant-oat01-...` tokens to `~/.claude/sensitive/oauth-tokens.md`, one per line.

---

## Progressive Difficulty Testing

Benchmarks should be hard enough that frontier models struggle. Follow this escalation:

1. **Sonnet baseline**: Run bench-loop with `sonnet[1m]` (default). Establish initial pass rates.
2. **Harden until sonnet struggles**: Target 30-50% pass rate on sonnet. If sonnet passes everything, cases aren't hard enough.
3. **Opus validation**: Switch to `--set 'model=opus[1m]'`. If opus passes easily, cases lack genuine difficulty.
4. **Calibrate**: The sweet spot is opus passing 60-80%, sonnet passing 30-50%.

### World Expansion Trigger

When a benchmark converges (all cases pass on the target model), consider expanding:
- Add new domains (e.g., vendor/finance for hotel management)
- Add harder cases (disambiguation, state reversal, cross-domain reasoning)
- Only expand after at least one full pass—don't expand while cases fail due to verifier bugs

---

## Pipeline Lifecycle

### Launching a new pipeline

```bash
export CLAUDE_CODE_OAUTH_TOKEN="<token>"
fleet pipeline bench-loop \
  --set benchDir=/path/to/benchmark \
  --set host=<EC2-IP> \
  --set sshUser=ec2-user \
  --set 'model=opus[1m]' \
  --session-name bench-<name>
```

### Running multiple pipelines for the same benchmark

Use different session names and tokens:

```bash
export CLAUDE_CODE_OAUTH_TOKEN="$TOKEN_A"
fleet pipeline bench-loop --set benchDir=... --session-name bench-slop

export CLAUDE_CODE_OAUTH_TOKEN="$TOKEN_B"
fleet pipeline bench-loop --set benchDir=... --session-name bench-slop-2
```

Both share the same EC2 host and Docker containers. The hardener in each pipeline will see each other's changes via git.

### Checking pipeline state

```bash
# Which round?
cat .claude/state/bench-loop/session-*/round.txt

# Round results?
cat .claude/state/bench-loop/session-*/round-results.json | python3 -m json.tool

# Stable passes?
cat .claude/state/bench-loop/session-*/stable-passes.json
```

### Graceful shutdown

1. Let the current round finish (don't kill mid-execution)
2. `tmux kill-session -t bench-<name>`
3. Terminate EC2 if no other pipelines need it

---

## Mission.md Structure

The supervisor's `mission.md` is the runtime configuration. It should contain:

1. **Role** — one-liner describing the supervisor's purpose
2. **Cron checklist** — the steps to execute on every tick (mirrors this convention)
3. **Pipeline commands** — exact `fleet pipeline bench-loop` commands per benchmark
4. **EC2 instance map** — which benchmarks share which hosts
5. **Rate limit recovery** — token rotation and model fallback specifics
6. **Key files** — paths to monitor scripts, pipeline programs, conventions

Update mission.md whenever:
- A new benchmark is added or removed
- EC2 hosts change
- Token rotation strategy changes
- New workflow steps are discovered

---

## Fleet Integration

### Communication

All status reports go through Fleet Mail:

```bash
fleet mail send "user@boring" "<subject>" "<body>"
```

### Worker identity

The supervisor is a fleet worker with its own mail name. It can send/receive messages from other fleet workers (executors, hardeners).

### Hooks

Static hooks handle:
- **PreCompact**: Re-inject mission.md into context on compaction
- **Stop**: Check for unpushed commits, drain inbox

Dynamic hooks can gate dangerous operations:
- Block `git push --force` globally
- Require confirmation before terminating EC2

---

## Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `$0.0000` cost, idle prompt | Rate limit on first API call | Switch model or token |
| STUCK for 2+ hours | Long-running task (training, Docker build) | Check pane—if `(running)`, wait |
| DEAD session, EC2 alive | Agent crashed or exited | Relaunch pipeline |
| DEAD session, EC2 unreachable | EC2 terminated or network issue | Launch new EC2, then pipeline |
| All tokens rate limited | Subscription-wide limit | Wait 1-2 hours, or add more tokens |
| Docker build fails on EC2 | Stale files, missing deps | `rsync --delete` + `docker build --no-cache` |
| Pipeline loops without progress | Hardener making no changes but not converging | Check stable-passes.json, may need manual review |

---

## Anti-Patterns

- **Don't restart long-running tasks.** A 2-hour opus agent monitoring training is normal, not stuck.
- **Don't spam status reports.** Only report when something changes.
- **Don't leave EC2 running without pipelines.** ~$0.34/hr per instance adds up overnight.
- **Don't keep relaunching on the same rate-limited token.** Each restart wastes cached context.
- **Don't amend commits.** The hardener creates new commits. Amending loses work.
