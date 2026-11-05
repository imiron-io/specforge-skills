# SpecForge Skills

Agentic skills for working with SpecForge and the Lilo specification language.

[SpecForge](https://imiron.io/specforge/) is a platform for authoring, managing, and monitoring temporal specifications.


## Prerequisites

The skills drive the `specforge` CLI, so it must be installed and on your `PATH`. Check with:

```bash
specforge --version
```

See the [SpecForge documentation](https://docs.imiron.io/v/latest/en/setting-up.html) for more details.

## Installation

In Claude Code, add this repository as a marketplace and install the plugin:

```
/plugin marketplace add imiron/specforge-skills
/plugin install specforge@specforge-skills
```

## Skills

| Skill | Purpose |
| --- | --- |
| `specforge` | The core edit → check → lint → analyze loop with the SpecForge CLI. |
| `lilo-lang` | Write and repair Lilo specs: syntax, temporal operators, and `specforge check` errors. |
| `lilo-project` | Scaffold and organize projects: `specforge.toml`, source layout, components, data templates. |
| `autoformalize` | Turn an informal requirements document into a validated, traceable Lilo project. Invoke explicitly with `/autoformalize`. |

Once installed, skills can be invoked via commands like `/specforge:autoformalize` through your agent's interface.


