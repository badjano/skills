# Commit scope — examples

## Example 0 — Auto-commit on feature branch

Agent finishes a small fix while on `feature/combat-hit-feel`.

1. Confirm branch is `feature/*` (not `develop`).
2. Run the hard gate; stage INCLUDE paths only.
3. Commit immediately (`fix: tighten hit stun window`) — do not wait for the user to ask.
4. Report short hash + what was left unstaged.

If the agent was on `develop`, create/switch to `feature/<name>` first, then commit.

## Example 0b — User rejects the change

User: “I didn’t like that at all” / “I don’t see any change.”

1. Identify the commit(s) for that rejected slice (e.g. last 1–N on this `feature/*` branch from this session).
2. `git reset --hard <sha-before-rejected-work>` — resets the commit **and** discards the changes.
3. Confirm clean status / expected `HEAD`. Do not reset on `develop`.

## Example A — Quality tiers after a perf session

**Dirty tree:** `QualitySettingsUI.cs`, `GraphicsQualityState.cs`, foliage cull scripts, `Runtime.asmdef`, plus hundreds of Synty/TreePack prefabs and mats from shadow A/B, SelectColor experiments, scene tweaks.

**Intent:** `perf: automate graphics quality tiers (low off / high on)`

| Path | Decision |
|------|----------|
| `Assets/Scripts/Core/GraphicsQualityState.cs` | INCLUDE — tier state |
| `Assets/Scripts/UI/QualitySettingsUI.cs` | INCLUDE — preset matrix |
| `Assets/Scripts/Systems/Generics/Props/Foliage*.cs` | INCLUDE — quality-driven cull/shadows |
| `Assets/Scripts/Runtime.asmdef` | INCLUDE — HTrace reference needed to compile |
| `Assets/Synty/**`, `Assets/TreePackVol.1/**`, experimental mats | EXCLUDE — measurement leftovers |
| `Assets/Scenes/Test_Map_Scene.unity` | EXCLUDE unless the tier feature requires a scene wire-up |

Stage only the INCLUDE list. Say so when finishing the commit.

## Example B — Diff-tab “commit all”

UI offers commit-and-push for the whole branch. Working tree still has unrelated WIP.

**Still run the hard gate.** Stage explicit paths for the named intent only; leave the rest unstaged; report exclusions.

## Example C — Two real intents in one tree

Files for a combat fix and an unrelated UI tweak.

**SPLIT:** two commits (or leave UI unstaged). Do not one-message both.
