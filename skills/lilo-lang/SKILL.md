---
name: lilo-lang
description: Author and repair Lilo specifications — the SpecForge temporal-logic DSL. Use when writing or fixing `.lilo` files, declaring signals/params/defs/specs, using temporal operators (always, eventually, until, since, historically, next, previous), or interpreting `specforge check` errors (type mismatches, undefined signals, unused-def warnings, interval syntax). Pulls the authoritative language reference at runtime via `specforge doc`.
---

# Lilo language

Lilo is SpecForge's DSL for formal temporal specifications. This skill is a **cheat-sheet
plus repair guide**; the authoritative reference lives in the binary — pull it on demand
rather than guessing syntax.

## Get the authoritative reference

```bash
specforge doc                  # list all pages with one-line summaries
specforge doc <topic>          # print one page
```

Direct the output of `specforge doc ...` to temporary files or grep to prevent polluting the 
context window.

Authoring topics: `lilo-language` (types, expressions, temporal operators), `lilo-systems`
(signals/params), `lilo-modules`, `lilo-components`, `lilo-additional-features`,
`conventions`, `semantics`. There is **no `lang-ref` topic** — the reference is split across
these. **Read `specforge doc lilo-language` before writing anything non-trivial.**

## Cheat-sheet (orientation only — pull the docs for exact syntax & semantics)

- **Files:** `.lilo`, or `.lilo.md` (literate).
- **Top level:** `system <Name>` / `module <Name>`; `import M [as A] [use { … }]`.
- **Declarations:** `signal n: T` (time-varying input), `param n: T` (fixed config). Optionally add
  `#[default = …]` (params only) when a source-backed/agreed default exists; and `#[range(min = …, max = …)]`
  to indicate a constrained range (params/signals), `def f(x: T): R = …` (reusable,
  often a predicate), `spec name = φ` (requirement to check), `assumption name = φ`, plus `enum`,
  `type`, `unit`, `component c: Sys { … }`. → `lilo-systems`.
- **Spec Management:** Docstrings (`/// comment` before declarations). Do not write docstrings as attributes. Also use `#[alias(lang = "...")]`, `#[label(...)]`, etc. if relevant.
- **Types:** `Int`, `Float`, `Bool`, `String`, records `{ lat: Float, lon: Float }`, enums.
- **Units:** declare/import units (`unit km`, `unit h`), annotate Float types/literals
  (`Float<km/h>`, `10.0<km/h>`, `5.0<deg/s>`), and keep compatible units in arithmetic/comparisons.
- **Records:** group naturally coupled fields in a record-valued `signal`, `param`, or `type`, then
  project with dots (`gps.lat`, `pose.yaw`).
- **Temporal operators / Streaming Aggregates:** `always`, `eventually`, `until`, `since`, `historically`, `next`,
  `previous`, `{did,will}_change`, `{max,min}_{past,future}`, `sum_{past,future}` each with an optional `[a, b]` interval. Exact bounds & semantics → `semantics`.
- **Logical:** `&&` `||` `!` `=>` (implies) `<=>` (iff); comparisons chain (`34.63 < lat < 34.65`).
- **Matching and Case Expressions:** Prefer `match enum { #constructor -> expr; … }` and `cases { boolExpr -> boolExpr; … }` for enums and multi-branch boolean expressions, respectively
- **Gotchas worth memorising:** spaced names need backticks (`` `Oil and Gas` ``); component
  access uses `::` (`comp::sub::signal`, `#Mode::Eco`); mixing Int/Float needs `float(…)`.
- **Attributes:** `#[default = …]`, `#[disable(unused)]`, `#[field(priority = …)]`.

### Minimal example

```lilo
system main
unit W
unit deg
signal hour: Int
/// Solar generation in watts.
signal Solar: Int
signal battery_power: Float<W>
signal gps: { lat: Float<deg>, lon: Float<deg> }
def night = (hour < 6 || hour > 18)
spec noSolarAtNight = night => Solar == 0
spec frequentlyNight = eventually [0, 12] night
spec atDepot = always (34.63<deg> < gps.lat < 34.65<deg> && 135.99<deg> < gps.lon < 136.01<deg>)
```

## Common errors and repairs (`specforge check`)

- **`file:line:col: type mismatch`** — usually Int vs Float. Wrap the Int side in `float(…)`
  (e.g. `float(Consumption) > 1.2 * avg`), or align literal types.
- **Undefined signal/identifier** — declare the missing `signal`/`param`/`def`, or fix a
  typo / missing backticks for a spaced name. For component refs, check `::` qualification
  and that the component is declared.
- **Unused def/signal warning** — either use it, delete it, or silence intentionally with
  `#[disable(unused)]` on the declaration.
- **Interval syntax** — temporal bounds are `[a, b]` immediately after the operator
  (`always [0, 5] φ`), not parentheses. Omit the interval for the unbounded form.
- **Parse error** — check backticks around spaced names, operator spelling (`=>` not `->`),
  and that each `spec`/`def` has a single `= <expr>` body.

When unsure of an operator's exact semantics, run `specforge doc semantics` rather than
guessing — bounds are inclusive/exclusive in specific ways documented. Read a specific operator's 
entry (`specforge doc semantics | grep -A3 -i "<operator>"`).

## Beyond `check`

A clean `check` only proves the file parses and typechecks. Run `specforge lint` (solver-free
warnings/hints) and `specforge analyze` (Z3 semantic checks: inconsistent, redundant, or bad
case-guard specs) to catch logically broken specs the type system can't — see the **specforge**
skill for both. When `analyze` flags an inconsistency, the fix is semantic (relax or split the
spec), not syntax.
