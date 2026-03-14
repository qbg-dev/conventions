# Agent Reproducibility Convention

Every qbg-dev repository must be set up end-to-end by an agent given only the repo URL. No human intervention, no hidden prerequisites, no tribal knowledge.

## Structure (1–15)

1. **Single entry point.** One command from clone to running state. Document it in the first 5 lines of the README.
2. **`Quick Start` is a contract.** If the quick start fails on a fresh machine, the repo is broken.
3. **List every dependency.** Runtime, build-time, system-level. No "you probably already have X installed."
4. **Pin versions for reproducibility.** `torch==2.5.1`, not `torch`. `python:3.12-slim`, not `python:3-slim`. Lockfiles committed.
5. **Separate install from run.** `install.sh` or `setup.sh` handles deps. The main command assumes deps are present.
6. **No manual configuration steps.** If a config file is needed, generate it with defaults. If an API key is needed, detect its absence and print exactly what to do.
7. **State the minimum hardware.** CPU count, RAM, disk, GPU (or lack thereof). An agent cannot guess your memory requirements.
8. **Document the expected output of every command.** Show what success looks like. Show what common failures look like.
9. **Every environment variable has a default or a clear error.** Never silently use an empty string.
10. **Dockerfiles are self-contained.** `docker build && docker run` must work. No external volumes, bind mounts, or host networking required for basic operation.
11. **Test commands documented.** How to run tests, what passing looks like, what the exit code means.
12. **CI reproduces the README.** If CI does different steps than the README, one of them is wrong.
13. **No OS-specific instructions without detection.** If you support Linux only, fail fast on macOS with a clear message. Don't let agents debug phantom errors.
14. **Offline-capable where possible.** Pre-download datasets, models, and large dependencies during build, not at runtime.
15. **Clean uninstall path.** Document what to delete. Agents cannot guess where you scattered files.

## Agent-Specific (16–30)

16. **Assume the agent has no prior context.** README + repo contents is all it gets. No Slack threads, no meeting notes, no "ask Kevin."
17. **Error messages must be actionable.** "Connection refused" is useless. "Connection refused: is the server running? Start it with `./start.sh`" is useful.
18. **No interactive prompts in setup.** Use flags, env vars, or config files. `echo y | ./install.sh` is a hack, not a solution.
19. **Idempotent setup.** Running setup twice must not break anything. Agents retry.
20. **Health checks for services.** If your repo runs a server, provide a health endpoint or check command. Agents need to verify "it's working" programmatically.
21. **Deterministic outputs where possible.** Same input, same output. If randomness is involved, document it and provide a seed mechanism.
22. **Fail fast, fail loud.** `set -euo pipefail` in shell scripts. Crash on import errors, not 200 lines later with a confusing traceback.
23. **No git submodules without justification.** They break shallow clones, confuse agents, and add failure modes. Prefer vendoring or package managers.
24. **Timeouts documented.** If a build takes 10 minutes, say so. Agents have timeouts and will kill "stuck" processes.
25. **Log to stdout/stderr, not files.** Agents read streams. If you must log to files, document the paths.
26. **Exit codes are semantic.** 0 = success, 1 = user error, 2 = system error. Not "always 0" or "random."
27. **No hardcoded absolute paths.** Use `$HOME`, `$(pwd)`, `$SCRIPT_DIR`. Agents run in arbitrary directories.
28. **Workspace isolation.** The repo should work in `/tmp/random-dir/`, not just `~/projects/my-repo/`.
29. **Document what changes system state.** If setup installs global packages, modifies shell profiles, or creates launchd services, say so explicitly.
30. **Provide a smoke test.** A 10-second command that proves the core functionality works. Not the full test suite — just "is it alive?"

## Verification (31–40)

31. **The README is tested.** Extract commands from README, run them in CI on a fresh image. If they fail, the README is wrong.
32. **Fresh-clone test.** CI job that clones the repo into an empty directory and runs the quick start. Monthly at minimum.
33. **Dependency drift detection.** Pin deps, but also run `unpinned` builds weekly to catch breakage early.
34. **Multi-platform CI where claimed.** If you say "works on macOS and Linux," test on both.
35. **Agent dry-run.** Periodically give the repo URL to a Claude/GPT agent and ask it to set up and use the software. File bugs for every failure.
36. **Build time budget.** Track build times. If Docker build goes from 2 min to 20 min, that's a bug.
37. **No network in tests.** Tests that require network access are flaky by definition. Mock, record, or pre-download.
38. **Cleanup verification.** After uninstall, verify no orphan files, processes, or ports remain.
39. **Version matrix.** Document which Python/Node/Rust versions work. Test the boundaries.
40. **Regression on setup.** If a contributor's PR breaks the setup path, CI catches it before merge.
