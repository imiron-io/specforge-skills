---
name: specforge
description: Author, repair and manage Lilo temporal specifications using the SpecForge CLI.
---

# SpecForge

SpecForge is a requirements-engineering platform for formal temporal specifications in
**Lilo**, its DSL. This skill drives the `specforge <command> [args]` CLI and the core
**edit → check → lint → analyze** loop. For deeper tasks, route to the companion skills below.

## The core loop

1. `specforge init <root> <name>` — scaffold a project (`specforge.toml` + source dir).
2. Edit `.lilo` files.
3. `specforge check` — **parse + typecheck. This is "validation."** Read errors as
   `file:line:col: <message>`, repair, re-run until clean.
4. `specforge lint` — once `check` is clean, surface warnings/hints.
5. `specforge analyze` — solver-backed semantic review.

## Three quality gates — don't conflate them

Run in this order; each is strictly deeper than the last.

- **`check`** — parse + typecheck. Fast correctness gate and core of the repair loop; nothing
  else is meaningful until it's clean.
- **`lint [--json] [--fail-on-info]`** — fast, **solver-free** warnings/hints across all source
  files (unused defs, style). Exits nonzero on any error/warning; `--fail-on-info` also fails on
  info/hint.
- **`analyze [--all] [-s SYS] [--spec S] [--timeout-seconds N] [--json]`** — solver-backed (Z3)
  analyses: **consistency** (system satisfiable?), **redundancy** (spec implied by others?), and
  **case-guard** checks (satisfiability / exhaustiveness / disjointness). Default prints only
  findings; `--all` also shows ✅ passes. Needs Z3 (`specforge doctor check-solver`). Exits
  nonzero unless all pass. Details: `specforge doc lilo-static-analysis`.

Supporting commands: `parse` (AST dump), `format` (pretty-print), `search --query Q`,
`schema --system S` (inspect signals/params), `schema-template --system S [csv|json]`
(generate data/param templates), `schema-diff --system S <data-or-param-file>` (compare
filled files against schema), `monitor`/`eval` (evaluate against CSV/JSON traces; needs SMT
solver), `flatten` (collapse hierarchy), `doc` (print Lilo docs — see lilo-lang).

## Naming traps

- **"Validate a project" ⇒ `check`, NOT `validate`.** `validate` is unrelated (antecedent⇒
  consequent query containment).
- `doctor` takes a sub-task: `doctor check-solver` (SMT) / `doctor check-llm` (LLM). It is
  **not** a bare project check.

## Spec Management

Done through docstrings, labels, aliases, field attributes attached to Lilo declarations. See `specforge doc spec-search` for details.

Three main commands:

1. **Search:** `specforge search --query <term>`
2. **Validate** Labeling Rules: `specforge validate --antecedent q1 --consequent q2`. Check if any declarations satisfy q1 without also satisfying q2.
3. **Bulk Label**: `specforge bulk-label --query SELECTOR_QUERY --label LABEL` Add new label to those matching the selector query.

## Which skill to use

| Situation                                                                                                               | Skill                            |
| ----------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| Writing/repairing Lilo formulas, syntax, temporal operators                                                             | **lilo-lang**                    |
| Setting up `specforge.toml`, `init`, project layout, components, and `schema`/`schema-template`/`schema-diff` workflows | **lilo-project**                 |
| Turning a requirements document (PDF/text) into a validated `.lilo` project                                             | **autoformalize** (user-invoked) |
