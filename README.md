# conventions

Engineering conventions for qbg-dev projects. Each convention is a skill-like directory with `frontmatter.yaml`, a `CONVENTION.md`, and optional scripts/references.

## Conventions

| Convention | Description |
|-----------|-------------|
| [convention-convention](convention-convention/) | Meta-convention: what a convention is, how to structure one, when to create vs. integrate. |
| [benchmarking](benchmarking/) | Creating and hardening agent benchmarks: design, build, the execute-harden loop, and review. Includes CI check scripts. |
| [agent-reproducibility](agent-reproducibility/) | Every repo must be agent-reproducible: clone → setup → run with zero human intervention. Two-agent verified. 52 rules. |
| [readme-conventions](readme-conventions/) | How to write READMEs that feel like infrastructure, not marketing. 100 rules. |

## Structure

```
conventions/
├── README.md                          # This index
├── convention-name/
│   ├── frontmatter.yaml               # Name, description, version — read to decide relevance
│   ├── CONVENTION.md                   # The convention itself
│   ├── scripts/                        # CI checks, validation scripts (optional)
│   └── references/                     # Reading lists, detailed tables (optional)
```

Read [convention-convention/CONVENTION.md](convention-convention/CONVENTION.md) for the full guide on creating and maintaining conventions.
