---
name: unity-test-framework
description: >-
  Unity Test Framework (UTF): EditMode vs PlayMode tests, Arrange-Act-Assert,
  domain-reload-safe state, async yields, and log assertions. Use when writing
  or fixing Unity tests, NUnit tests under Assets, PlayMode tests, or validating
  editor tooling and runtime systems with UTF.
---

# Unity Test Framework

Source paradigm: Unity Editor Deep Research — EditMode/PlayMode UTF. Complements `unity-cli` (batch test runs) and `unity-awaitable` (async waits).

---

## Mode selection

| Mode | Runs in | Use for |
|------|---------|---------|
| **EditMode** | Editor, no Player loop required | Serialization, editor tools, pure logic, AssetDatabase |
| **PlayMode** | Enter Play Mode / player loop | Runtime behaviour, physics, scenes, Awaitable/frame waits |

Put tests in asmdefs with correct references (`UnityEngine.TestRunner`, `UnityEditor.TestRunner` for EditMode-only APIs).

---

## Structure

Always **Arrange → Act → Assert**.

- `[SetUp]` / `[TearDown]` for per-test fixtures
- `[OneTimeSetUp]` / `[OneTimeTearDown]` for expensive shared fixtures
- Prefer deterministic scene/prefab fixtures over hidden scene hierarchy dependencies

```csharp
[Test]
public void AppliesSerializedSpeed()
{
    // Arrange
    var go = new GameObject();
    var mover = go.AddComponent<Mover>();
    // Act
    mover.SetSpeed(3f);
    // Assert
    Assert.AreEqual(3f, mover.Speed);
}
```

---

## Domain reload & async

- Scene/domain-reload spanning tests must **persist state explicitly** (UTF domain-reload patterns) — do not assume static fields survive recompile mid-test.
- Prefer UTF yield / await instructions for editor async over spinning `Thread.Sleep`.
- For Unity 6 runtime async under test, follow `unity-awaitable` (CancellationToken; no double-await).
- Assert console expectations with UTF log assertion APIs when validating errors/warnings — don’t rely on manual console reading alone in CI.

---

## Hard rules

1. Tests must be **hermetic** — no dependence on the user’s open scene selection when avoidable.
2. Clean up created assets/GOs in TearDown (EditMode asset churn especially).
3. Do not ship tests that require disabling domain reload as a silent default — document if required.
4. CI: run via `unity-cli` batchmode test platform flags; parse logs for failures.

---

## Agent checklist

- [ ] Correct EditMode vs PlayMode choice
- [ ] AAA structure; SetUp/TearDown cleanup
- [ ] Domain-reload-safe if crossing reload
- [ ] Async uses UTF/Awaitable patterns with cancel
- [ ] Runnable headlessly via CLI when user asks for CI

## Related

- Batch compile/test: `unity-cli`
- Editor APIs under test: `unity-editor-extensibility`
