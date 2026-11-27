# API Design Reference Layer

This directory holds the API design reference layer in two independent parts:

| Part | File(s) | Role |
| ---- | ------- | ---- |
| Normative philosophy | `philosophy.md` | Seven decidable principles, shared vocabulary, exclusion boundary, worked examples. Always loaded. |
| Case-study subcategories | `<case>.md` | How a specific real-world API instantiates or violates the principles. Informative only; never normative. |

Keep these layers independent: adding a case study must never require editing
`philosophy.md` or `SKILL.md` beyond the index row.

## Precedence

From highest to lowest:

1. The project's own RFCs, interface contracts, and configuration (they
   override everything below on conflict; cite them in the answer).
2. `philosophy.md` — the normative invariants and acceptance criteria.
3. Case-study subcategories — illustrative patterns and anti-patterns.

A case study may add specificity but MUST NOT weaken, override, or contradict
`philosophy.md`. If they appear to disagree, the philosophy wins and the case
study should be corrected.

## Dispatch

1. Always load `philosophy.md` for every mode.
2. Load this README before any case study.
3. Load a case study only when the user asks for a comparison, when the project
   explicitly adopts that API as a reference, or when a principle's worked
   example is insufficient for the task.
4. Never treat a case study as a source of requirements for a project. When a
   case study shows a shape that the exclusion boundary forbids, report it as
   an anti-pattern to learn from, not a pattern to copy.

## Subcategories

| Subcategory | File | Covers |
| ----------- | ---- | ------ |
| Google BigQuery API design | `google-bigquery-api-design.md` | P1 unified `jobs` resource with typed `configuration`; P5 constant envelope and open `labels`; P3 cursor pagination; P4 caller-chosen `jobReference.jobId`; P7 `projection` and `dryRun` as parameters; the explicit "borrow philosophy, not shape" exclusion list. |

## Add a new subcategory

1. Create `references/api-design/<case>.md` with: a source/attribution header,
   a principle-by-principle mapping table, an explicit "do not copy" section,
   and targeted grep patterns for large references.
2. Add one row to the subcategory table above and one row to the `SKILL.md`
   reference-subcategories table. No philosophy or workflow changes.
3. Smoke-test: load the new case study in `principle` mode for at least one
   principle and confirm the answer still cites `philosophy.md` as authority.

## What a case study must never contain

- Normative language directed at a project (`MUST` / `MUST NOT` aimed at the
  user's contract). Use "in this API…" / "this is an anti-pattern here…".
- A claim that a vendor shape is required by the philosophy.
- Duplicated principle statements from `philosophy.md`; reference the principle
  tag instead.
- Credentials, environment-variable secret names, or internal endpoints.
