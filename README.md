# Badjano Stack

Public Agent Skills for Cursor (and compatible tools). Opinionated engineering habits — code craft, process, and Unity — packaged so agents follow them consistently.

Each folder is a skill with a `SKILL.md` entry point.

## Code & architecture

| Skill | Purpose |
|---|---|
| [`code-architecture`](code-architecture/) | SOLID, services, events, DI, logging discipline |
| [`code-authoring`](code-authoring/) | Naming, cognitive load, surgical diffs, readable craft |
| [`flexible-coding-principles`](flexible-coding-principles/) | Hierarchy, composition, DI, decoupling patterns |

## Process

| Skill | Purpose |
|---|---|
| [`commit-scope`](commit-scope/) | Keep commits scoped to one logical change |
| [`conflict-resolution`](conflict-resolution/) | Systematic merge-conflict handling |
| [`precision-executor`](precision-executor/) | Break failure loops with minimal viable actions |

## Unity

| Skill | Purpose |
|---|---|
| [`unity-architecture`](unity-architecture/) | Unity architecture, SerializeField rules, pooling, perf, SO data patterns |
| [`unity-runtime-guardian`](unity-runtime-guardian/) | Runtime debugging: console, asserts, logging discipline |

## Install

Copy or symlink the skill folders you want into:

- **Cursor (personal):** `~/.cursor/skills/`
- **Cursor (project):** `.cursor/skills/`

On Windows, personal skills live under `%USERPROFILE%\.cursor\skills\`.
