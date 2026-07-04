# Starter Tree — layered Unity skeleton

An empty skeleton to copy when starting a new Unity project. Create the layer folders and their
`.asmdef`s **first**, with references pointing inward, before writing any feature code. Replace
`MyGame` with your project name throughout. See [`ARCHITECTURE.md`](ARCHITECTURE.md) for the rules
this encodes.

```
Assets/_Project/Scripts/
  Common/            MyGame.Common          (no deps)              — Result, Result<T>, generic primitives
  Domain/            MyGame.Domain          → Common               — pure C#, no UnityEngine, no exceptions
    <Model>/         (value objects that form one model = one folder)
    <Geometry>/
  Application/       MyGame.Application     → Domain, Common       — pure & synchronous, headless-testable
    <Feature>/                              — orchestrator at the feature root…
    <Feature>/<Step>/                       — …collaborators behind interfaces, one concern per folder
    Serialization/                          — ISerializer + Json impl + DTOs
  Infrastructure/    MyGame.Infrastructure  → Application, Domain, Common, UniTask, UnityEngine
    Loading/                                — async ports (ILevelProvider : UniTask<Result<T>>), providers, fallback chain
    Configuration/                          — ScriptableObject configs
  Presentation/      MyGame.Presentation    → Application, Domain, UnityEngine
    <Feature>/Views/                        — MonoBehaviour views
    <Feature>/Layout/                       — pure layout value objects
    Input/Pointer/   Input/Picking/  Input/Snapping/
    Configuration/                          — feel/tuning ScriptableObjects
  Composition/       MyGame.Composition     → all above            — manual composition root, no DI framework
    GameBootstrapper (entry + serialized assets), GameServices (wiring), <Flow>Controller
  Editor/            MyGame.Editor          → Common, Domain, Application, Presentation
  Tests/
    EditMode/        MyGame.Tests           → Domain, Application, Common   — pure, run headless via dotnet
    EditModeUnity/   MyGame.Tests.Unity     → + UnityEngine                 — Unity Test Runner only
```

## `.asmdef` reference rules

The reference list in each `.asmdef` is what enforces the inward-pointing arrows — get these right
and the compiler prevents architecture drift:

| Assembly | References (and nothing outward) |
|---|---|
| `MyGame.Common` | — |
| `MyGame.Domain` | `MyGame.Common` |
| `MyGame.Application` | `MyGame.Domain`, `MyGame.Common` |
| `MyGame.Infrastructure` | `MyGame.Application`, `MyGame.Domain`, `MyGame.Common`, `UniTask` |
| `MyGame.Presentation` | `MyGame.Application`, `MyGame.Domain` (+ `MyGame.Common`) |
| `MyGame.Composition` | all of the above |
| `MyGame.Editor` | `MyGame.Common`, `MyGame.Domain`, `MyGame.Application`, `MyGame.Presentation` |
| `MyGame.Tests` | `MyGame.Domain`, `MyGame.Application`, `MyGame.Common` |

Keep `MyGame.Domain` and `MyGame.Application` **without any UnityEngine reference** — that's what
lets their tests run under plain `dotnet`. Mark them "no engine references" in the `.asmdef` where
the option exists.

## Example `.asmdef`

```json
{
  "name": "MyGame.Application",
  "references": ["MyGame.Domain", "MyGame.Common"],
  "includePlatforms": [],
  "excludePlatforms": [],
  "noEngineReferences": true
}
```

## First steps in a new repo

1. Copy `.editorconfig` and `.gitattributes` to the repo root.
2. Create the folder + `.asmdef` skeleton above (empty layers are fine).
3. Add `Result` / `Result<T>` to `Common` first (everything else returns them).
4. Write one Domain value object + its `dotnet` test to confirm the headless path works before
   opening the Unity editor.
