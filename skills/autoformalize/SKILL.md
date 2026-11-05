---
name: autoformalize
description: Turn an informal requirements document (PDF, text, HTML) into a validated Lilo project, with every formal spec traceable back to its source text. User-invoked via /autoformalize. Two modes — autonomous (end-to-end, flags assumptions for later review) and interactive (pauses at checkpoints). Use when asked to formalize requirements, convert a spec document to Lilo, or autoformalize a requirements PDF.
disable-model-invocation: true
---

# Autoformalize

Convert a natural-language requirements document into a **validated** Lilo project. The
ordering is load-bearing — **architecture and schema before formulas, traceability
throughout** — to avoid the error-compounding of formalizing prematurely.

Companion skills: **lilo-project** (layout/`specforge.toml`/`schema`/`schema-template`/`schema-diff`), **lilo-lang** (syntax +
repair), **specforge** (the `check` loop).

Before drafting schema or formulas, read the compact language guidance in **lilo-lang** and pull
the current reference with `specforge doc lilo-systems` and `specforge doc lilo-language`.
Use `specforge doc lilo-additional-features` too when units, attributes, or other advanced syntax
appear. This is part of the run, not an optional repair step.

## Pick a mode first

Ask the user (unless they already said). Both run the same pipeline; they differ only in
whether each ⏸ checkpoint is a hard stop or a logged decision.

- **Autonomous** — run end-to-end with no prompts, formalizing best-effort and recording every
  assumption/ambiguity. Deliver the validated project **plus a flagged-assumptions report**.
  Good for a fast first pass or large doc sets.
- **Interactive** — pause at each **⏸** and ask before proceeding. Slower, but resolves
  ambiguity before it propagates.

## Flagging uncertainty

Three markers keep uncertain work visible without breaking `check`:

- **`#[review = "<question>"]`** on any declaration — the "a human should verify this" marker.
  In interactive mode it's the ⏸ checkpoint; in autonomous mode you log it and continue.
- **`param` without a default** — a numeric threshold/configuration value the document leaves
  unspecified (tolerance, bound, rate, deadline, dwell time). Lift it into a `param`, write the
  formula against it, and add `#[review]` explaining what value is needed. `check` still passes;
  monitoring/eval can provide the parameter value from a data file.
- **Defaulted `param`** — use `#[default = <value>]` only when the document gives a default/nominal
  value, a stakeholder supplied one, or you deliberately chose a review-flagged value for an
  autonomous first pass. Do not add defaults to every parameter just to make monitoring easier.
- **Stub** (bodyless `spec`/`assumption`, treated as `true`) — only for requirements with no
  measurable criterion at all (e.g. "conserve energy"), not for a merely-missing number.
  `assumption` for an environment given, `spec` for a system guarantee.

Decision rule: _could I write the formula if that value were supplied?_ Yes → `param`, usually
without a default. Add `#[default]` only for a source-backed or explicitly reviewed value. No
measurable criterion at all → stub.

**`param` vs `def` turns on the value's _role_, not on whether the document names it.** A stated
number is not automatically a `def`. Tunable configuration — thresholds, limits, bounds,
tolerances, setpoints, deadlines, rates, dwell times — should almost always be a `param` even when
the doc gives a value: record that value as `#[default = <value>]` so the formula reads concretely
while the knob stays exposed. Reserve `def` for genuine constants (`pi`, unit conversions, values
that are part of the requirement's _meaning_, not a setting). When unsure, prefer a defaulted
`param`. Note: you can constrain params using `assumption`s.

```lilo
#[source = "§3.2"]
#[review = "Unclear whether 'within 5 s' is a hard deadline or a target; assumed hard."]
spec ResponseTimeout = always (request => eventually[0, 5] ack)

unit degC

/// Observed room temperature (§6.1).
#[source = "§6.1"]
signal room_temp: Float<degC>

/// Maximum temperature change between consecutive samples (§6.1).
#[source = "§6.1"]
#[review = "No temperature-step threshold given; provide a parameter value for monitoring."]
param max_temp_step: Float<degC>

#[source = "§6.1"]
assumption gradual_temperature = always (abs(room_temp - previous room_temp) <= max_temp_step)
```

Every `#[review]` item, parameter needing an external value/default decision, and unfilled stub is
collected in the final report.

## Schema modeling reminders

- Use units for physical quantities. Declare/import `unit`s, annotate `Float` types and literals
  (`Float<km/h>`, `10.0<km/h>`, `5.0<deg/s>`), and do not erase dimensions into comments.
- Package tightly related quantities as records when they are one conceptual signal/configuration
  (for example `gps: { lat: Float<deg>, lon: Float<deg> }` or a vehicle pose with position,
  velocity, and yaw). Use dot projection (`gps.lat`) in formulas. Keep fields separate only when
  they are truly independent signals with different ownership or sampling.
- Use `/// comment` docstrings before declarations. Do not write docstrings as attributes.

## Pipeline (copy this checklist into your response and tick it off)

```
- [ ] 1. Bootstrap project + confirm check passes
- [ ] 2. Ingest document → normalized Markdown with source refs
- [ ] 3. Architecture decision (cluster + decide)   ⏸ system/component shape
- [ ] 4. Schema: signals/params/types (fan out → judge + merge)   ⏸ schema sign-off
- [ ] 5. Shared code: reusable types + `def`s
- [ ] 6. Spec stubs: docstring + attributes, no bodies (fan out → judge)   ⏸ coverage sign-off
- [ ] 7. Fill specs in batches of ~5 related specs   ⏸ on each ambiguity
- [ ] 8. Review pass: improve + reconcile all the work
- [ ] 9. Validate + repair until clean (check → lint → analyze)
```

Steps 4–8 follow a **fan-out → judge** rhythm: sub-agents work in parallel within a phase, the
main agent judges and merges between phases (see **Scaling** for sizing, and when to collapse it
inline). At every ⏸, **write the artifact to files and point the user to them** — never reproduce
tables, signal lists, or spec text in chat. A one-liner like "Set up `specforge.toml` with three
components — see `src/`. Does the split look right?" is enough.

1.  **Bootstrap.** Look for `specforge.toml` in or above the working dir; if absent, `specforge init
<root> <name>`. Either way run `specforge check` to confirm clean before adding anything.
2.  **Ingest.** Produce one normalized `ingested.md` with stable source refs (page/section/line)
    so every spec can cite its origin. If the input is already Markdown, copy/symlink as-is;
    otherwise extract and reformat, preserving headings/numbering as refs. Delegate extraction to
    a sub-agent for large/numerous docs; inline for one small doc.
3.  **Architecture decision. ⏸** Identify entities, signals, modes, constraints; decide whether a
    `system`/`component` hierarchy is warranted or a single system suffices; scaffold it
    (`specforge.toml` entries + empty `.lilo` files). **Write no formulas yet.** For large/complex docs,
    launch an `Explore`-style sub-agent to extract candidate components first; inline for a small
    doc.
4.  **Schema (fan out → judge + merge). ⏸** Define the `signal`/`param`/`type` set across the
    **whole project** before any `spec`. Schema sub-agents (one per component, or per requirements
    cluster for a single system) draft their slice with source refs; the **main agent judges and
    merges** — **one canonical signal per real-world quantity**, unified naming/units,
    record-vs-scalar modeling, and resolved type conflicts. If agents surface new component
    boundaries, feed them back into step 3.
5.  **Shared code.** With the schema settled, create the **shared** code specs build on: `type`s
    not already in the schema, and `def`s multiple specs will reuse (e.g. `comfortable`,
    `too_cold`). Strictly DRY — one `def` per reusable concept, extracted, not re-derived per spec.
6.  **Spec stubs (fan out → judge). ⏸** Create a **stub for every requirement** — `spec`/
    `assumption`, no body, carrying a docstring lifted from the source, `#[source = "<ref>"]`, and
    `#[review]` where uncertain. The **main agent judges the set**: exactly one stub per
    requirement, no overlap, every stub traced, step-5 `def`s actually reused. Pause for a coverage
    check (every requirement represented?).
7.  **Fill specs in batches.** Group stubs into sets of **~5 related specs**, one sub-agent per
    batch to write bodies. Each agent calls `specforge doc <topic>` for any uncertain syntax —
    **never read sibling example projects as a substitute**. Keep every `#[source]`; preserve/
    refine docstrings. Don't formalize an ambiguous requirement without `#[review]` (then ⏸ in
    interactive, log in autonomous). Missing numbers become `param`s; add `#[default]` only for
    source-backed or explicitly reviewed defaults. Reserve stubs for the genuinely unquantifiable.
8.  **Review pass.** Once all batches are filled, run a dedicated review (one reviewer, or several
    split by area): each spec checked against its source docstring for fidelity, cross-batch
    inconsistencies hunted (duplicated logic that should be a shared `def`, clashing naming/units,
    contradictions), traceability confirmed. The main agent applies the agreed improvements.
9.  **Validate + repair.** Three commands, different roles. `check` is the one **hard gate** (must
    typecheck); `lint` and `analyze` are **advisory** — read each finding, act on the pertinent
    ones, deliberately keep (record or `#[disable]`) the rest. Not acceptable: skipping them, or
    silencing a finding you never looked at.
    1.  **`specforge check`** — fix each `file:line:col` error (see **lilo-lang**), re-run until
        clean. Run `check` continuously throughout steps 4–8 too; don't save repair for the end.
    2.  **`specforge lint`** — once clean, clear the warnings/hints worth clearing (unused/dead defs
        from drafting and the step-4 merge, style); you may judge a hint not pertinent and leave it.
    3.  **`specforge analyze`** — solver-backed (Z3) review, the payoff for schema-and-stubs-first
        ordering: it catches formalization mistakes a clean typecheck hides. **Treat output as
        advice, not verdicts** — findings are hedged ("**may** be redundant") because they come from
        heuristics and a _bounded_ solver. - **Inconsistent** → you likely formalized something unsatisfiable (contradicting specs, or
        an empty interval). Usually real; treat like `#[review]` — fix the formula or flag the
        conflict. - **Redundant** → two requirements collapsed. Your call: (a) genuine duplicate → merge;
        (b) `def`s not shared → refactor; (c) **intentional** (e.g. defense-in-depth safety
        invariant) → keep and attach `#[disable(redundancy)]`. Don't delete a valid safety spec to
        quiet the tool. - **Case-guard** → non-exhaustive or overlapping mode logic.

              `#[disable(<analysis>)]` suppresses one analysis for one spec — use for a finding you've
              examined and chosen to keep, never one you haven't. Needs Z3 (`specforge doctor

        check-solver`). Scope slow runs with `-s <system>`/`--spec`and **keep`--timeout-seconds`
        ≤ ~10s\*\* — a finding that only appears or vanishes as you push the timeout higher is
        solver-limited and not trustworthy. Roll every real-but-unfixed finding into the report.

    Optionally generate `schema-template` files (and use `schema` / `schema-diff` as needed)
    and run `monitor`/`eval` if trace data exists.

## Scaling with sub-agents

Match the fan-out to the work:

- **Single small document** (≲1 component, ≲20 signals / ≲15 requirements): collapse the phases
  **inline**, no sub-agents. The phase _ordering_ still holds; you just don't delegate.
- **Multi-component or large document**: fan out. Default sizing — one schema agent per component
  (step 4), one stub agent per component or major section (step 6), one fill agent per ~5 related
  specs (step 7), one reviewer per area (step 8). The main agent always judges between phases.

The non-negotiable invariant regardless of size: **schema judged before stubs; stubs judged
before bodies; bodies reviewed before sign-off.** Sub-agents parallelize _within_ a phase; they
never start a later phase on an unjudged earlier one.

## Hard constraints (do not violate)

- Phase order is fixed: architecture → schema → shared code → stubs → bodies → review, each
  judged/merged by the main agent before the next begins.
- One canonical signal per real-world quantity; preserve physical dimensions with units; package
  tightly related fields as records; shared concepts in a reusable `def`, never copy-pasted.
- Every `spec` cites `#[source = "<ref>"]`. Never formalize an ambiguous requirement without
  `#[review]`. A missing number becomes a `param`, not a hardcoded literal or a stub; add
  `#[default]` only for source-backed or explicitly reviewed defaults. Reserve bodyless stubs for
  requirements with no measurable criterion.
- At ⏸ checkpoints, write artifacts to files and point the user to them — no tables/long lists in
  chat.
- Use `specforge doc <topic>` for syntax questions; never read sibling example projects instead.
- The Lilo output is a standalone artifact. No meta-comments about the formalization process
  ("values the document leaves unspecified", "written during autoformalize", section banners
  grouping params by how they were derived). Traceability lives in `#[source]`/`#[review]`
  attributes and docstrings; comments should explain the spec, not its provenance in the run.
- A clean `check` is not "done": always also run `lint` and `analyze` and act on what they
  surface (see step 9 — advisory, not gates).

## Final deliverable

A project that passes `specforge check`, with `lint` and `analyze` run over it (pertinent
findings acted on, the rest kept with a note or `#[disable]`), **plus** an assumptions/ambiguities
report. The report ties every interpretation to its source ref so a reviewer can confirm the
formalization against the original, and includes a dedicated section — grouped by file — for every
`#[review]` item, every parameter that needs an external value/default decision, every unfilled
stub, and every `analyze` finding you judged real but couldn't cleanly resolve or deliberately
suppressed, each with its question and source ref.
