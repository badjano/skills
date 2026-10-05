---
name: unity-cli
description: >-
  Compile and automate Unity projects via Editor CLI (batchmode). Use whenever
  Unity MCP cannot verify the *target* project (wrong MCP project, Editor
  closed, or user asks for CLI compile): script errors, batchmode builds/tests,
  locate Unity.exe, parse Editor logs for error CS. Pair with unity-mcp-skill
  and unity-architecture mandatory verification — do not skip post-change proof.
---

# Unity Editor CLI

**Verification is mandatory after Unity changes** (scripts, DLL drops, upgrades). Prefer **Unity MCP** when the Editor is open *and* MCP is attached to the **same** project you edited. Prefer **Editor MenuItems** for setup/validation/recompile. Use this skill when the project is closed, MCP is down / pointed elsewhere, or the user wants a headless compile/build.

## Hard preference (all Unity games)

- **Do not** add project `.ps1` / `.bat` / `.sh` wrappers for Unity tasks.
- Put workflow in `Assets/**/Editor/` as `[MenuItem]` (+ optional static method for `-executeMethod`).
- Agent runs `Unity.exe ... -executeMethod Namespace.Class.Method` directly when headless is required — never invent a repo script.

Official reference: [Unity Editor command line arguments](https://docs.unity3d.com/Manual/EditorCommandLineArguments.html)

---

## Locate the Editor

1. Read `ProjectSettings/ProjectVersion.txt` → `m_EditorVersion` (e.g. `6000.5.9f1`).
2. Resolve `Unity.exe` in this order:

| Priority | Path pattern |
|----------|----------------|
| 1 | Custom editors root if configured (e.g. `G:\UnityEditors\{version}\Editor\Unity.exe`) |
| 2 | `%ProgramFiles%\Unity\Hub\Editor\{version}\Editor\Unity.exe` |
| 3 | `%LOCALAPPDATA%\Unity\Hub\Editor\{version}\Editor\Unity.exe` |
| 4 | Hub `editors.json` / glob `**/Editor/Unity.exe` under known roots |

```powershell
$ver = (Select-String -Path "ProjectSettings\ProjectVersion.txt" -Pattern "m_EditorVersion: (.+)").Matches.Groups[1].Value.Trim()
$candidates = @(
  "G:\UnityEditors\$ver\Editor\Unity.exe",  # optional custom root — adjust or remove
  "$env:ProgramFiles\Unity\Hub\Editor\$ver\Editor\Unity.exe",
  "$env:LOCALAPPDATA\Unity\Hub\Editor\$ver\Editor\Unity.exe"
)
$unity = $candidates | Where-Object { Test-Path $_ } | Select-Object -First 1
```

Do **not** rely on `unity.exe editors list` from Hub CLI — it can hang indefinitely. Prefer filesystem checks.

Windows paths: do not end `-projectPath` with a single trailing `\`.

---

## Script compile check (primary recipe)

Opening a project in batchmode imports assets and compiles scripts, then `-quit` exits. No custom `-executeMethod` required for a basic compile gate.

```powershell
$project = (Resolve-Path ".").Path   # or absolute project root
$log = Join-Path $project "Temp\cli-compile.log"
New-Item -ItemType Directory -Force -Path (Join-Path $project "Temp") | Out-Null
if (Test-Path $log) { Remove-Item $log -Force }

& $unity `
  -batchmode -nographics -quit `
  -accept-apiupdate `
  -projectPath $project `
  -logFile $log

Write-Host "EXIT=$LASTEXITCODE"
```

| Flag | Why |
|------|-----|
| `-batchmode` | Headless; no dialogs; exit 1 on hard failures |
| `-nographics` | No GPU; OK for compile-only. **Requires** `-logFile` (stdout logs are off) |
| `-quit` | Exit after work finishes |
| `-accept-apiupdate` | Run APIUpdater in batchmode (omit → possible CS errors) |
| `-logFile` | Full log path (always set under `Temp/`) |

**Timeouts:** first open / large projects can take many minutes. Use `block_until_ms` ≥ 600000 and poll the log / `Unity` process.

**Mutex:** cannot batchmode a project while the same project is open in the Editor. Kill stray `Unity` processes only if the user agrees or the process is clearly stuck from a prior CLI run.

---

## Parse the log

Success signals:
- Process exit code `0`
- Line like `Exiting batchmode successfully now!`
- `Csc ... Assembly-CSharp.dll` completed; DLLs copied under `Library/ScriptAssemblies/`

Failure signals:
- `error CS####`
- `Scripts have compiler errors`
- Exit code `1`
- Bee/Csc failure before assemblies copy

```powershell
Select-String -Path $log -Pattern "error CS|Scripts have compiler errors"
Select-String -Path $log -Pattern "warning CS"   # optional; not a fail
Select-String -Path $log -Pattern "Exiting batchmode"
```

Ignore licensing/token noise unless the run aborts with a license error. Warnings (`CS0618`, etc.) are not compile failures.

Report: exit code, whether any `error CS` matched, and the log path.

---

## executeMethod (builds, custom CI)

Static Editor method, script under an `Editor/` folder:

```text
-executeMethod Namespace.ClassName.MethodName
```

- On failure: throw, or `EditorApplication.Exit(nonZero)`.
- Args: `Environment.GetCommandLineArgs`.
- With `-activeBuildProfile`, profile scripting defines apply **before** the method runs.
- Async work + `-quit` can hang; prefer finishing synchronously or raising timeout via `-quitTimeout`.

Force recompile from an Editor method: `UnityEditor.Compilation.CompilationPipeline.RequestScriptCompilation(...)` — wait for compilation finished before exit (async).

---

## Other useful flags

| Flag | Use |
|------|-----|
| `-buildTarget win64` | Force platform on first import |
| `-buildWindows64Player <path>` | Player build (legacy CLI) |
| `-runTests` / `-testResults` | EditMode/PlayMode tests (do **not** pair `-quit` with in-progress tests) |
| `-username` / `-password` | License login (avoid; prefer already-activated Editor) |
| `-ignorecompilererrors` | Continue despite CS errors (usually wrong for a compile gate) |

Docs also cover Accelerator (`-EnableCacheServer`, `-cacheServerWaitForUploadCompletion` with `-quit`).

---

## MCP vs CLI

| Situation | Tool |
|-----------|------|
| Editor open + MCP on the **target** project | **unity-mcp-skill** (compile/console/smoke) |
| Project closed / MCP unavailable / MCP on a **different** project / “CLI compile” | **This skill** |
| Architecture / SerializeField rules | **unity-architecture** |

**Wrong-project MCP does not count as verification.** Confirm `Application.dataPath` before trusting console/compile results.

Do not invent a project-local shell wrapper; run `Unity.exe` directly with `-executeMethod` when headless. When the Editor is open on the target project, use MenuItems / MCP instead of fighting the project mutex with batchmode.

### Post-change gate (required)

After any Unity code/DLL/package change that should compile:

1. Try MCP on the target project → `scriptCompilationFailed`, console errors, smoke check.
2. If MCP cannot target that project → CLI compile recipe above + parse `error CS`.
3. If Editor holds the mutex and MCP is wrong → stop and say verification is blocked; do not declare success.
