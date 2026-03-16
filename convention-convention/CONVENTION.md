# Convention Convention

How to create, structure, and maintain conventions in this repo. A convention is a skill for workflows—it encodes domain knowledge, business logic, and step-by-step process that agents and humans follow when doing a specific type of work.

## What a Convention Is

A convention is a **standard workflow** that gives specific, important domain and business logic about how things should be done. It's not documentation, not a style guide, not a reference page. It's an opinionated, actionable playbook for a recurring activity.

Good conventions answer: "I need to do X. What are the steps, what are the rules, and what are the pitfalls?" They encode the kind of knowledge that would otherwise live in someone's head or get re-explained in every code review.

**A convention is not:**
- A list of links or resources (that's a reference file)
- An explanation of how something works (that's documentation)
- A set of abstract principles without concrete steps (that's a manifesto)
- A coding style guide (use linters)

## One Convention Per Workflow

There should be a **single convention for each workflow**. Claude (and humans) have a tendency to create new conventions when they encounter new information. Resist this. Before creating a new convention, check if the new material belongs in an existing one.

**The integration test:** If you're about to create `FOO-BAR.md` and `FOO-BAZ.md` already exists, ask: "Would someone doing FOO need to read both?" If yes, merge them. Two conventions that must be read together are really one convention that's been split badly.

**When to create a new convention vs. integrate:**
- New convention: the workflow is genuinely distinct—different triggers, different actors, different outputs
- Integrate: the new material is a phase, variant, or edge case of an existing workflow

**Examples:**
- Benchmarking creation + benchmarking execution → one convention (same workflow, different phases)
- Agent reproducibility + agent setup verification → one convention (same goal, different angles)
- README conventions + benchmarking conventions → two conventions (different workflows entirely)

## Directory Structure

Each convention lives in its own directory, structured like an agent skill:

```
convention-name/
├── frontmatter.yaml      # Metadata — name, description, type, version
├── CONVENTION.md          # The convention itself (required)
├── scripts/               # Executable scripts for deterministic tasks (optional)
│   └── ci_checks/         # Example: CI validation scripts
├── references/            # Docs loaded into context as needed (optional)
│   └── resources.md       # Example: reading list, links
└── assets/                # Templates, examples, fixtures (optional)
```

### frontmatter.yaml

Every convention directory has a `frontmatter.yaml` that an agent can read to decide relevance without loading the full convention:

```yaml
name: convention-name
description: >
  One-paragraph description of what this convention covers and when to read it.
  Be specific enough that an agent can decide "do I need this?" from the
  description alone. Err on the side of triggering — include the contexts
  and keywords where this convention applies.
type: convention
version: "1.0"
owner: warren
files:
  convention: CONVENTION.md
  scripts: scripts/           # optional
  references: references/     # optional
  assets: assets/             # optional
```

The `description` field is the primary mechanism for deciding when to load the convention. Write it the way you'd write a skill trigger description—include what the convention does AND the specific situations where it applies.

### CONVENTION.md

The convention body. This is the actionable playbook. Guidelines:

- **Progressive disclosure.** The frontmatter (always loaded, ~50 words) tells agents whether to read more. The CONVENTION.md (~300-500 lines) is loaded when relevant. Reference files are loaded on demand.
- **Imperative form.** "Pin all versions" not "Versions should be pinned."
- **Explain the why.** Don't just say MUST/NEVER—explain why the rule exists. Agents with context make better judgment calls than agents following rote rules.
- **Concrete examples.** Show good and bad. Real code, real filenames, real error messages.
- **Checklists where useful.** Especially for multi-step phases. But don't make the whole convention a checklist—narrative context matters.
- **Target 300-500 lines.** If longer, split supporting material into `references/`. If shorter, the convention might be too narrow to justify its own directory.

### scripts/

Executable code for deterministic, repetitive tasks within the workflow. CI checks, validation scripts, setup automation. These should be runnable without reading the full convention.

### references/

Supporting material that's useful but not essential on every read. Reading lists, detailed tables, API references, research notes. The CONVENTION.md should link to these with guidance on when to read them.

## Naming

Directory names are `kebab-case`. Convention names match directory names. Keep names short and descriptive of the workflow, not the artifact: `benchmarking` not `benchmark-creation-and-execution-guide`.

## Maintaining Conventions

- **Update, don't duplicate.** When you learn something new about a workflow, update the existing convention. Don't create a `v2` or an addendum.
- **Keep frontmatter in sync.** When the convention content changes significantly, update the description in `frontmatter.yaml`.
- **Version bumps.** Bump the version in `frontmatter.yaml` when the convention changes in ways that affect existing workflows (not for typo fixes).
- **Deprecation.** If a convention becomes obsolete, delete it. Don't mark it deprecated—dead conventions confuse agents.

## The Convention Index

The repo `README.md` serves as the index. It lists every convention with a one-line description and a link. When you create or rename a convention, update the README.
