# Oh-My-Coding

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Agent Skills Format](https://img.shields.io/badge/format-Agent%20Skills%201.0-blue)](https://agentskills.io)

Oh-My-Coding (OMC) is an Agent Skill that supplies **decidable API design
guidance**. It turns API design from taste into seven checkable invariants and
keeps real-world API knowledge in isolated, informative case studies.

> **Borrow the philosophy, never the shape.**

## What it provides

- A shape-neutral **philosophy** of seven principles — one operation envelope,
  resource-oriented bounded verbs, cursor-first pagination, request-scoped
  idempotency, uniform envelope with additive evolution, two-layer
  machine-readable errors, and projection as parameter — each with rules and an
  acceptance test.
- A **case-study subcategory** for Google BigQuery API design, including the
  explicit list of vendor shapes that must not be copied.
- A strict separation of layers: project contracts > philosophy > case studies.
  Adding a case study never changes the philosophy.

## Structure

```text
skills/oh-my-coding/
├── SKILL.md                              # skill entry point, modes, workflow
└── references/
    └── api-design/
        ├── README.md                     # reference-layer contract, dispatch, subcategory index
        ├── philosophy.md                 # normative seven principles (shape-neutral)
        └── google-bigquery-api-design.md # informative case study subcategory
```

## Modes

| Mode | Trigger | Scope |
| ---- | ------- | ----- |
| `design` (default) | "design this API", "add a new capability" | Apply all seven principles to a proposed contract |
| `review` | "review this API design" | Audit a contract for violations, with evidence |
| `principle` | "explain P3" | One principle only |

## Quick start

```text
/plugin marketplace add <owner>/oh-my-coding
/plugin install oh-my-coding@oh-my-coding
/reload-plugins
```

Then talk naturally: "Review this API design against the seven principles",
"How should I add a post-processing action without breaking clients?", or
"Explain P4 with the BigQuery case study."

## Adding a reference subcategory

1. Add `skills/oh-my-coding/references/api-design/<case>.md` with a principle
   mapping table and an explicit "do not copy" section.
2. Add one row to the subcategory tables in `references/api-design/README.md`
   and `SKILL.md`.

No philosophy or workflow changes are needed.
