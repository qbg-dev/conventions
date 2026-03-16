# Benchmark Resources

Reading list for agent benchmark design. Companion to [BENCHMARKING.md](BENCHMARKING.md).

## Benchmark Design Methodology

| Resource | Key takeaway |
|----------|-------------|
| [TB3 Implementation Rubric](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_IMPLEMENTATION_RUBRIC.toml) | 19-criteria rubric with detailed guidance. The quality bar. |
| [TB3 Review Automation](https://github.com/harbor-framework/terminal-bench-3/blob/main/TASK_REVIEW_AUTOMATION.md) | Automated pipeline: static checks → execution checks → agent trials. |
| [APEX-Agents](https://arxiv.org/abs/2601.14242) (Mercor) | 480 tasks across 33 "worlds." Gold standard for world design. |
| [Archipelago](https://github.com/Mercor-Intelligence/archipelago) (Mercor) | Open-source Docker sandbox harness for APEX worlds. |
| [BetterBench](https://arxiv.org/abs/2411.12990) (Stanford, NeurIPS 2024) | 46-criteria framework. Most benchmarks fail to report statistical significance. |
| [Demystifying Evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (Anthropic, 2025) | Start with 20-50 tasks from real failures, grade outcomes not tool-call sequences, use pass@k. |
| [Challenges in Evaluating AI Systems](https://www.anthropic.com/research/evaluating-ai-systems) (Anthropic, 2023) | Multiple-choice formatting sensitivity shifts scores ~5%. Model-generated evals are circular. |

## Construct Validity

| Resource | Key takeaway |
|----------|-------------|
| [Measuring What Matters](https://arxiv.org/pdf/2511.04703) (NeurIPS 2025) | 445 benchmarks reviewed; 8 recommendations for valid measurement. |
| [Measurement to Meaning](https://arxiv.org/abs/2505.10573) (2025) | Distinguish narrow claims (math test scores) from broad claims (general reasoning). |
| [The Evolving Landscape of LLM Evaluation](https://newsletter.ruder.io/p/the-evolving-landscape-of-llm-evaluation) (Ruder, 2024) | 10% drops on GSM1k vs GSM8k reveal benchmark-specific overfitting. Assume contamination. |

## Anti-Gaming and Anti-Contamination

| Resource | Key takeaway |
|----------|-------------|
| [LiveCodeBench](https://arxiv.org/abs/2403.07974) | Time-segmented evaluation: only test on problems released after training cutoff. |
| [EvalPlus](https://arxiv.org/abs/2305.01210) (NeurIPS 2023) | Adding 80x more tests dropped pass rates 19-29%. Original test suites are always insufficient. |
| [Specification Gaming](https://deepmind.google/blog/specification-gaming-the-flip-side-of-ai-ingenuity/) (DeepMind) | Better algorithms find more creative loopholes. Specify outcomes comprehensively. |
| [Reward Hacking in RL](https://lilianweng.github.io/posts/2024-11-28-reward-hacking/) (Weng, 2024) | Models modify unit tests, exploit length/sophistication bias. |
| [Spec Gaming in Reasoning Models](https://arxiv.org/pdf/2502.13295) (Palisade, 2025) | Reasoning LLMs hack chess by modifying opponent's engine. Directly relevant to agent benchmarks. |

## Contributing Guides

| Benchmark | Contributing guide | Format |
|-----------|--------------------|--------|
| **Terminal-Bench 3** | [TB3 Contribution Call](https://www.tbench.ai/news/tb3-contribution-call) | [Harbor task format](https://harborframework.com/docs/tasks) |
| **SlopCodeBench** | [SCBench Contributing](https://github.com/SprocketLab/slop-code-bench/tree/main/docs/contributing-problems) | Checkpoint-based config.yaml |
| **METR** | [METR Task Standard](https://github.com/METR/task-standard) | METR task format |
