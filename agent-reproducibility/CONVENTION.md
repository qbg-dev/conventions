# Agent Reproducibility Convention

Every qbg-dev repository must be set up end-to-end by an agent given only the repo URL. No human intervention, no hidden prerequisites, no tribal knowledge. Verified by a two-agent setup test: both Claude and Codex must succeed independently.

## The Test (1–8)

1. **Pass/fail.** Clone the repo into a fresh directory. Give the URL to an agent. The agent must install, build, run, and verify output with zero human help. If it fails, the repo is broken—not the agent.

2. **Two-agent rule.** Run with both a Claude Opus agent and an OpenAI Codex agent. Both must succeed independently. Different agents fail on different things.

3. **Fresh environment, every time.** Empty directory, nothing pre-installed beyond base OS. No cached deps, no prior state, no warm Docker layers. `mktemp -d` is your friend.

4. **Agent gets only the repo URL.** No verbal instructions, no Slack context, no "you'll also need to..." hints. README and repo contents are the entire interface.

5. **Success means verified output.** "It didn't error" is not success. The agent must produce observable proof: health check response, test suite passing, CLI output matching expected, or running service responding on the documented port.

6. **Time-boxed to 15 minutes.** If an agent needs longer, the setup is too complex or under-documented.

7. **No network workarounds.** The agent must not need to find alternative download URLs, work around rate limits, or retry flaky mirrors.

8. **Record the transcript.** Save the agent's full session log. When the test fails, the transcript is the bug report.

## What "Set Up" Means (9–16)

9. **Application repos:** clone → install deps → build → run smoke test → verify output. Five steps, all without manual intervention.

10. **Docker projects:** `docker build && docker run` must work. Health endpoint or check command must return passing.

11. **CLI tools:** install → `--help` (must exit 0) → run primary command with sample input.

12. **Libraries:** install → import → run example from README with documented output.

13. **Services with dependencies:** `docker-compose up` brings everything up. No manual `CREATE DATABASE` or seed steps.

14. **Monorepos:** each package has its own setup path documented and tested independently.

15. **Repos requiring API keys:** detect absence, print exact instructions, exit 1. The agent can't provision keys but can verify the error message tells it what to do.

16. **Repos with hardware requirements:** fail fast with a clear message. "Requires CUDA 12.x" at import time, not a cryptic segfault 200 lines later.

## Repo Structure (17–30)

17. **Single entry point.** One command from clone to running state. Document it in the first 5 lines of the README.

18. **`Quick Start` is a contract.** If the quick start fails on a fresh machine, the repo is broken.

19. **List every dependency.** Runtime, build-time, system-level. No "you probably already have X installed."

20. **Pin versions.** `torch==2.5.1`, not `torch`. `python:3.12-slim`, not `python:3-slim`. Lockfiles committed.

21. **Separate install from run.** `install.sh` handles deps. The main command assumes deps are present.

22. **No manual configuration steps.** Generate configs with defaults. Detect missing API keys and print what to do.

23. **State minimum hardware.** CPU, RAM, disk, GPU (or lack thereof).

24. **Document expected output of every command.** Show what success looks like. Show what common failures look like.

25. **Every env var has a default or a clear error.** Never silently use an empty string.

26. **Dockerfiles are self-contained.** `docker build && docker run` must work with no external volumes or host networking for basic operation.

27. **No interactive prompts in setup.** Use flags, env vars, or config files.

28. **Idempotent setup.** Running setup twice must not break anything. Agents retry.

29. **Fail fast, fail loud.** `set -euo pipefail` in shell scripts. Crash on import errors.

30. **No hardcoded absolute paths.** Use `$HOME`, `$(pwd)`, `$SCRIPT_DIR`. Agents run in arbitrary directories.

## Agent-Specific (31–40)

31. **Assume no prior context.** README + repo contents is all the agent gets. No Slack threads, no meeting notes.

32. **Error messages must be actionable.** "Connection refused: is the server running? Start it with `./start.sh`" not just "Connection refused."

33. **Health checks for services.** Provide a health endpoint or check command. Agents need to verify "it's working" programmatically.

34. **Deterministic outputs where possible.** Same input, same output. Document randomness, provide seed mechanisms.

35. **Log to stdout/stderr, not files.** Agents read streams. If logging to files, document the paths.

36. **Exit codes are semantic.** 0 = success, 1 = user error, 2 = system error. Not "always 0."

37. **Workspace isolation.** The repo must work in `/tmp/random-dir/`, not just `~/projects/my-repo/`.

38. **Document what changes system state.** Global packages, shell profiles, launchd services—say so explicitly.

39. **Provide a smoke test.** A 10-second command proving core functionality works. Not the full test suite.

40. **Test commands documented.** How to run tests, what passing looks like, what the exit code means.

## When to Run (41–45)

41. **Before every PR merge.** Agent-setup test is a required CI check. PRs that break setup do not merge.

42. **After every dependency update.** Bumping versions can break setup in ways unit tests miss.

43. **Monthly on a cron.** Upstream drift—expired certs, yanked packages, rotated URLs—breaks repos silently.

44. **After README changes.** The README is the setup contract. Changing it without re-running the agent test is changing the API without running the tests.

45. **On new OS releases.** macOS and Ubuntu LTS upgrades break system-level deps. Run the test within one week.

## Failure Modes (46–52)

46. **Missing system deps.** Setup assumes `jq`, `curl`, or `build-essential` are present. They're not on a fresh image.

47. **Unpinned versions that drifted.** `pip install torch` installed 2.5 when you wrote the README and 2.6 when the agent ran it.

48. **Interactive prompts blocking automation.** `apt install` wants confirmation. Use `-y`, `DEBIAN_FRONTEND=noninteractive`.

49. **Network-dependent steps at runtime.** Downloading models on first run. Pre-download during build or provide offline fallbacks.

50. **OS-specific paths.** `/usr/local/bin` on macOS, `/usr/bin` on Linux. Use portable path resolution.

51. **Stale README instructions.** The README says `npm start` but the start script was renamed to `npm run dev` three PRs ago.

52. **Auth chicken-and-egg.** Setup requires a token from the service setup is trying to start. Provide a bootstrap mechanism or mock auth.
