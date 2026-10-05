---
name: unity-mcp-orchestrator
description: Orchestrate Unity Editor via MCP (Model Context Protocol) tools and resources. Use when working with Unity projects through MCP for Unity - creating/modifying GameObjects, editing scripts, managing scenes, running tests, or any Unity Editor automation. Provides best practices, tool schemas, and workflow patterns for effective Unity-MCP integration.
---

# Unity-MCP Operator Guide

This skill helps you use the Unity Editor through the **current** MCP tool surface. Always discover live tool schemas (`GetDynamicTools` / namespace `unity-mcp`) before calling — names and args can differ by package version.

## Current tool surface (Unity 6 MCP — 2026)

| Tool | Use for |
|------|---------|
| `Unity_RunCommand` | Compile + execute a C# `IRunCommand` script in the Editor (create/modify objects, PrefabUtility, TestRunner kickoff, project settings, sanity type loads) |
| `Unity_GetConsoleLogs` | Console errors / warnings / messages + stack traces after every change |
| `Unity_Camera_Capture` | Capture from a Camera instance ID, or Scene View if omitted |
| `Unity_SceneView_Capture2DScene` | Orthographic 2D region capture |
| `Unity_SceneView_CaptureMultiAngleSceneView` | 2×2 multi-angle Scene View (3D layout checks) |
| `Unity_AssetGeneration_*` | Asset generation only when the user explicitly asks |

There is **no** `refresh_unity`, `read_console`, `manage_gameobject`, `manage_scene`, or `batch_execute` on this surface. Prefer `Unity_RunCommand` for Editor automation and `Unity_GetConsoleLogs` for console. Older `mcpforunity://…` resource / tool names in `references/` are **templates for other packages** — do not call them here unless discovery shows they exist.

## Quick Start

```
1. Confirm project     → Unity_RunCommand logs Application.dataPath (+ productName / unityVersion)
2. Act                 → Unity_RunCommand (IRunCommand script)
3. Verify compile/logs → Unity_GetConsoleLogs (errors + warnings)
4. Visual check        → Unity_Camera_Capture or SceneView capture tools when needed
```

## Critical Best Practices

### 0. Confirm MCP Is Attached to the Target Project (Mandatory)

Before any compile/console/smoke claim:

1. `Unity_RunCommand` that logs `Application.dataPath` (and ideally `Application.productName` / `Application.unityVersion`).
2. If the path is **not** the project you edited, **stop** — switch MCP to the target project, or fall back to **unity-cli** (only if that project’s Editor is closed / MCP cannot attach).
3. After script/DLL/package changes: wait for compile, then require no relevant console **errors** via `Unity_GetConsoleLogs`.

This gate is shared with **unity-architecture** (§0) and **unity-cli**.

### 1. After Writing/Editing Scripts: Always Sanity-Compile and Check Console

Use `Unity_RunCommand` with a tiny type/load script (e.g. `typeof(SomeType).FullName` or a no-op that touches edited assemblies), then:

```
Unity_GetConsoleLogs(maxEntries=50, includeStackTrace=true, logTypes="error")
Unity_GetConsoleLogs(maxEntries=50, includeStackTrace=false, logTypes="warning")
```

**Why:** Unity must compile before claims are valid. Fix errors you introduced in the same turn.

If MCP cannot attach to this project, follow **unity-cli**. Only claim “unverified” if **both** MCP and CLI failed.

### 2. `Unity_RunCommand` Golden Template

```csharp
using UnityEngine;
using UnityEditor;

internal class CommandScript : IRunCommand
{
    public void Execute(ExecutionResult result)
    {
        // 1. Logic
        // 2. result.RegisterObjectCreation / RegisterObjectModification / DestroyObject
        result.Log("dataPath={0}", Application.dataPath);
    }
}
```

Rules: class **must** be named `CommandScript`; use `internal`; use `result` for undo tracking and logging.

### 3. Screenshots for Visual Results

- `Unity_Camera_Capture` — specific Camera instance ID, or omit for Scene View
- `Unity_SceneView_CaptureMultiAngleSceneView` — 3D layout validation (expensive)
- `Unity_SceneView_Capture2DScene` — orthographic region

Prefer captures only when layout/visual proof is needed. for UI prefab reviews, prefer the project’s HideAndDontSave capture helpers when they exist.

### 4. Play Mode

- Prefer Play from **`Bootstrap`**, not bare `Test_Map_Scene`.
- Hard-timeout Play checks (e.g. 10–15s) then force `EditorApplication.isPlaying = false`.
- Never block the main thread with `Thread.Sleep` while TestRunnerApi runs — deadlocks MCP.

### 5. Asset generation

Only call `Unity_AssetGeneration_*` when the user explicitly asks to generate/modify an asset. Follow the tool’s consent rules on first use in a conversation.

## Parameter / discovery notes

- Always inspect the live `unity-mcp` namespace schema before calling.
- `Unity_RunCommand` args are typically `Code` + optional `Title` (casing may vary — match schema).
- `Unity_GetConsoleLogs`: `maxEntries`, `includeStackTrace`, `logTypes`.

## Fallback

| Situation | Action |
|-----------|--------|
| MCP on wrong project | Stop; reattach or use **unity-cli** |
| Editor closed / MCP down | **unity-cli** compile |
| Both blocked | Say unverified in chat (and vault note if delivery skill applies) |

## Template Notice

Files under `references/` may describe older MCP-for-Unity APIs. Treat them as optional patterns. **This SKILL.md + live tool discovery win** for the active Unity project.
