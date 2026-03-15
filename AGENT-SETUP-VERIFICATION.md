# Agent Setup Verification Convention

Every qbg-dev repository must pass a two-agent setup test: clone the repo, hand it to an agent, the agent sets it up and runs it end-to-end with zero human intervention. Two different agents, both must succeed. Repos without this test are broken by default.

## The Test (1–8)

1. **The test is pass/fail.** Clone the repo into a fresh directory. Give the URL to an agent. The agent must install, build, run, and verify output with zero human help. If it fails, the repo is broken—not the agent.

2. **Two-agent rule.** Run the test with both a Claude Opus agent and an OpenAI Codex agent. Both must succeed independently. Single-agent testing is insufficient—different agents fail on different things.

3. **Fresh environment, every time.** The agent starts in an empty directory with nothing pre-installed beyond a base OS. No cached deps, no prior state, no warm Docker layers. `mktemp -d` is your friend.

4. **The agent gets only the repo URL.** No verbal instructions, no Slack context, no "you'll also need to..." hints. The README and repo contents are the entire interface.

5. **Success means verified output.** "It didn't error" is not success. The agent must produce observable proof: a health check response, a test suite passing, a CLI output matching expected, a running service responding on the documented port.

6. **Time-boxed.** The entire setup must complete within 15 minutes. If an agent needs longer, the setup process is too complex or under-documented.

7. **No network workarounds.** The agent must not need to find alternative download URLs, work around rate limits, or retry flaky mirrors. All critical resources must be pinned and reachable.

8. **Record the transcript.** Save the agent's full session log. When the test fails, the transcript is the bug report.

## What "Set Up" Means (9–16)

9. **For application repos: clone, install deps, build, run smoke test, verify output.** Five steps, all must succeed without manual intervention.

10. **For Docker projects: build, run, verify health.** `docker build && docker run` must work. Health endpoint or check command must return a passing result.

11. **For CLI tools: install, run --help, run primary command.** `--help` must exit 0. The primary command must produce documented output on sample input.

12. **For libraries: install, import, run example.** The example from the README must execute and produce the documented output.

13. **For services with dependencies (databases, queues): docker-compose up must bring everything up.** No manual `CREATE DATABASE` or seed steps outside the compose file.

14. **For monorepos: each package has its own setup path documented and tested independently.** "See the root README" is not a setup path.

15. **For repos requiring API keys: detect absence, print exact instructions, exit 1.** The agent cannot provision your API key, but it can verify that the error message tells it exactly what to do.

16. **For repos with hardware requirements: fail fast with a clear message.** "Requires CUDA 12.x" at import time, not a cryptic segfault 200 lines into execution.

## When to Run (17–21)

17. **Before every PR merge.** The agent-setup test is a required CI check. PRs that break setup do not merge.

18. **After every dependency update.** Bumping a version can break the setup path in ways unit tests miss. Run the agent test on the dependency-update branch.

19. **Monthly on a cron.** Upstream drift—expired certificates, yanked packages, rotated URLs—breaks repos silently. Catch it before a human discovers it in anger.

20. **After README changes.** The README is the setup contract. Changing it without re-running the agent test is changing the API without running the tests.

21. **On new OS releases.** macOS and Ubuntu LTS upgrades break system-level deps. Run the test on the new OS within one week of release.

## How to Define It (22–27)

22. **Use agent-eval.** Write the reproducibility test as a `.test.yaml` file using [qbg-dev/agent-eval](https://github.com/qbg-dev/agent-eval). This is the canonical format for agent-verifiable tests.

23. **Test lives in the repo.** Place it at `tests/agent-setup.test.yaml` or `agent-eval/setup.test.yaml`. The test is part of the repo, not an external artifact someone has to find.

24. **Test structure: setup, agent task, verification.** Setup creates a clean workspace. The agent task gives the repo URL and says "set it up." Verification checks that the expected outputs exist and are correct.

25. **Two test variants: one per agent.** `agent-setup-claude.test.yaml` and `agent-setup-codex.test.yaml`. Agent-specific flags and auth differ. Both must pass.

26. **Pin the agent model in the test.** `model: claude-sonnet-4-6` not "latest." Agent capability changes between model versions. Reproducibility requires pinning.

27. **Test timeout matches rule 6.** Set `timeout: 900` (15 minutes). If the test needs more, the repo needs simplifying.

## Failure Modes (28–35)

28. **Missing system deps.** The setup script assumes `jq`, `curl`, or `build-essential` are present. They are not on a fresh image. List every system dependency or install them in the setup script.

29. **Unpinned versions that drifted.** `pip install torch` installed 2.5 when you wrote the README and 2.6 when the agent ran it. Pin versions.

30. **Interactive prompts blocking automation.** `apt install` wants confirmation. `npm init` asks questions. Use `-y`, `--yes`, `DEBIAN_FRONTEND=noninteractive`, or pre-generate config files.

31. **Network-dependent steps at runtime.** Downloading models, datasets, or configs on first run. Pre-download during build or provide offline fallbacks.

32. **OS-specific paths.** `/usr/local/bin` on macOS, `/usr/bin` on Linux. `$HOME/.config` vs `$HOME/Library/Application Support`. Use portable path resolution or document platform requirements.

33. **Stale README instructions.** The README says `npm start` but the start script was renamed to `npm run dev` three PRs ago. The agent follows the README literally.

34. **Implicit environment variables.** The code reads `DATABASE_URL` with no default and no error message. The agent has no way to know what value to provide.

35. **Auth chicken-and-egg.** Setup requires a token that can only be obtained by running the service that setup is trying to start. Provide a bootstrap mechanism or mock auth for local development.

## The Convention (36–40)

36. **Every qbg-dev repo MUST have an agent-eval test that verifies end-to-end setup.** No exceptions. No "we'll add it later." If the test does not exist, the repo is broken.

37. **Both agents must pass.** A repo that only works with Claude or only works with Codex has an implicit dependency on agent-specific behavior. Fix the repo, not the agent.

38. **Agent-setup test failures block releases.** A failing agent-setup test is a P0 bug. It means new contributors, new team members, and automated systems cannot use the repo.

39. **The test is the source of truth for setup.** If the README and the test disagree, the test is right. Update the README.

40. **Treat agent-reproducible setup as a feature, not a chore.** Every minute spent on setup verification saves hours of "it works on my machine" debugging. The agent is your most demanding user—if it can set up the repo, anyone can.
