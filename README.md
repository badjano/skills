# Skills

Public Agent Skills for Cursor (and compatible tools). Each folder is a skill with a `SKILL.md` entry point.

## Code & architecture

| Skill | Purpose |
|---|---|
| [`code-architecture`](code-architecture/) | SOLID, services, events, DI, logging discipline |
| [`code-authoring`](code-authoring/) | Naming, cognitive load, surgical diffs, readable craft |
| [`flexible-coding-principles`](flexible-coding-principles/) | Hierarchy, composition, DI, decoupling patterns |

## Process

| Skill | Purpose |
|---|---|
| [`commit-scope`](commit-scope/) | Auto-commit every change on `feature/*` (never `develop`); scoped staging; reject → hard reset |
| [`conflict-resolution`](conflict-resolution/) | Systematic merge-conflict handling |
| [`precision-executor`](precision-executor/) | Break failure loops with minimal viable actions |

## Unity

| Skill | Purpose |
|---|---|
| [`unity-architecture`](unity-architecture/) | Unity architecture, SerializeField rules, pooling, SO data patterns, verification gate |
| [`unity-awaitable`](unity-awaitable/) | Unity 6 `Awaitable` async rules (PlayerLoop, cancellation, pooling) |
| [`unity-canvas-ui-expert`](unity-canvas-ui-expert/) | uGUI Canvas layout, prefab YAML, soft-close overlays, surgical prefab edits |
| [`unity-cli`](unity-cli/) | Headless Editor CLI compile/build when MCP cannot verify the target project |
| [`unity-dots`](unity-dots/) | DOTS/ECS: chunks, structural changes, `IJobEntity`, Burst |
| [`unity-editor-extensibility`](unity-editor-extensibility/) | Custom inspectors, SerializedObject, UI Toolkit, Scene View Overlays |
| [`unity-mcp-project-settings`](unity-mcp-project-settings/) | Unity MCP config only inside Unity projects (`UserSettings/mcp.json`) |
| [`unity-mcp-skill`](unity-mcp-skill/) | Orchestrate Unity Editor via MCP tools and resources |
| [`unity-optimization`](unity-optimization/) | Profile-first FPS/GC/GPU optimization playbook |
| [`unity-render-graph`](unity-render-graph/) | Unity 6 URP/HDRP Render Graph custom passes |
| [`unity-runtime-guardian`](unity-runtime-guardian/) | Runtime debugging: console, asserts, logging discipline |
| [`unity-test-framework`](unity-test-framework/) | EditMode/PlayMode UTF tests, AAA, domain-reload-safe state |

## Install

Copy or symlink the skill folders you want into:

- **Cursor (personal):** `~/.cursor/skills/`
- **Cursor (project):** `.cursor/skills/`

On Windows, personal skills live under `%USERPROFILE%\.cursor\skills\`.
