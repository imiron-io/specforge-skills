---
name: lilo-project
description: Set up and structure a SpecForge/Lilo project — `specforge.toml`, `specforge init`, source layout, single-file vs. multi-file component hierarchies, and generating monitoring data templates with `specforge schema-template`. Use when scaffolding a new project, organizing `.lilo` files, deciding on systems/components, or wiring up CSV/JSON trace data for `monitor`/`eval`.
---

# Lilo projects

How to bootstrap and lay out a SpecForge project. For formula authoring see **lilo-lang**;
for the CLI loop see **specforge**.

## Bootstrap

```bash
# Init into a fresh OR existing dir. Quote names with spaces.
# Always use the default src/ layout — do NOT pass -s . even for single-file projects.
specforge init . "My Project"
specforge check                      # REQUIRED immediately after init — do not skip or wait
```

`init` writes `specforge.toml`, `src/main.lilo` (a sample signal/param/def/spec), `.gitignore`, and
a `.lilo/` cache dir. The generated `main.lilo` is an unrelated placeholder — **leave it** if the
user wants to learn from it or no real spec is ready, **wipe it** down to `system main` and build
from scratch once real signals/specs are known.

## `specforge.toml`

Minimal:

```toml
[project]
name = "Tutorial: Vehicle Data"
source = "src"          # directory holding .lilo files
description = "..."
```

Optional analysis sections — `[consistency]`, `[redundancy]`, `[diagnostics]`,
`[language.interval]`, `[language.freeze]`, … — carry `enabled` flags and timeouts that tune what
`specforge lint`/`analyze` report and for how long. **Tune later**, not required to start. Full
reference: `specforge doc project-configuration`.

## Layout: single-file vs. multi-file

**Single system, one file** — start here for most projects. Put one `system <Name>` in
`main.lilo` with all signals/params/defs/specs.

**Multi-file component hierarchy** — when the real system decomposes into reusable parts.
One file per system; compose with `component` and wire signals/params through. For example, a
`Vehicle` system composing a `Battery` and a `PowerUnit`:

```lilo
system Vehicle
import MathUtil use { Mode }
signal ambient_temp: Float
#[default = 75.0]
param temp_threshold: Float

component battery: Battery                    // unmapped
component power: PowerUnit {                   // mapped: wire parent signals into child
  param max_output = 100.0
  signal temperature = ambient_temp + speed * 0.5
}

def low_battery: Bool = battery::level < 20.0  // qualified access with ::
spec vehicle_safe = always (low_battery => speed < 50.0)
```

Use `specforge flatten [--system S]` to inspect the composed system as one flat unit, and
`specforge search --query Q` to locate definitions across files.

## Monitoring data (optional)

Evaluating specs against real traces needs data files for signals and params. CSV/JSON may match params/signals  
via aliases declared using `alias(lang = "...")`. (See `specforge doc data-files`)

1. `specforge schema-template --system <S> [csv|json] [--only signal|param] [--samples N]` — generate
   templates for the system's signals/params. Generated samples are placeholders, and may not satisfy specs.
2. (Optional) `specforge schema --system <S>` — inspect the system schema before filling data.
3. (Optional) `specforge schema-diff --system <S> <data-or-param-file>` — compare filled files
   against the schema.
4. Fill with trace data (CSV or JSON — one column/key per signal).
5. `specforge monitor` (verdicts / robustness over the trace) or `specforge eval`.

References: `specforge doc data-files`, `specforge doc online-monitoring`.
