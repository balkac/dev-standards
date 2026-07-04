# Architecture Playbook — Unity / layered apps

How I structure a project: layers, assemblies, folders, and the one rule that keeps the core
testable (the async boundary). This is the structural companion to [`CONVENTIONS.md`](CONVENTIONS.md)
(which covers code style, naming, SOLID-in-the-small, and error handling). Written for Unity, but
the layering, package-by-feature, and async-boundary ideas apply to any layered .NET app.

For an empty skeleton to copy, see [`starter-tree.md`](starter-tree.md).

---

## 1. Clean architecture — dependencies point inward

```
Domain  ←  Application  ←  Infrastructure / Presentation  ←  Composition
                                     ↑
                         Common (cross-cutting, no deps)
```

- **Domain** — pure C#, **no framework** (no `UnityEngine`), **no exceptions**. Value objects,
  entities, geometry, the core model. Trusts its own internally-validated inputs.
- **Application** — pure & **synchronous**, **headless-testable**, **no framework**. Use cases and
  algorithms orchestrated from small collaborators behind interfaces. Depends only on Domain +
  Common.
- **Infrastructure** — framework-coupled adapters: loading, networking, serialization I/O,
  persistence, configuration `ScriptableObject`s. This is the first layer allowed to be async and
  to touch `UnityEngine`.
- **Presentation** — views, input, rendering, animation. Framework-coupled.
- **Composition** — the manual composition root: wires everything, owns the entry point. No DI
  framework needed; construct and inject by hand. Name it `Composition`, **not** `App` (too close
  to `Application`).
- **Common** — generic cross-cutting primitives with **no dependencies** (e.g. `Result` /
  `Result<T>`). Nothing project-specific goes here.

**One `.asmdef` per layer.** Assembly references encode the arrows above; the compiler then *stops*
you from referencing outward or pulling the framework into the core.

## 2. Assemblies & folders

- **Folder name = the assembly's last segment; one assembly spans the whole layer.** Each layer's
  top folder matches its `.asmdef`: `Domain/` → `MyGame.Domain`, `Presentation/` →
  `MyGame.Presentation`, …, `Composition/` → `MyGame.Composition`. The `.asmdef` file is named
  after the assembly (`MyGame.Composition.asmdef`).
- **Organizational sub-folders below a layer get no assembly and no namespace segment.** They only
  group files (see §4). A file's namespace is declared at the **feature/concern** level under its
  layer (`MyGame.Presentation.Input`, `MyGame.Domain.Levels`); the finer sub-folders inside it keep
  that namespace rather than deepening it — e.g. `Input/Snapping/GridSnapResolver.cs` is still
  `MyGame.Presentation.Input`. This relaxes a strict "folders = namespaces": a file's namespace
  need not include every folder segment. (Some namespaces may legitimately nest deeper where the
  concept genuinely does — don't churn those.)

## 3. Package by feature, not by technical layer

- **No global `Model` / `Services` / `Utils` split.** Keep a concern's data and logic together:
  `Generation/Cutting`, `Generation/Atoms`, `Generation/Partitioning` — not a top-level `Models/`
  bucket that scatters one feature across the tree.

## 4. One responsibility per folder

- **A folder holds a single concern:** an abstraction with its implementation(s) and the small
  types that exist only to support it — e.g. `Input/Snapping/{ISnapResolver, GridSnapResolver,
  SnapResult}`.
- **Different jobs never share a folder.** A view, a layout value-object, and an overlay effect
  each get their own sub-folder (`Board/Views`, `Board/Layout`, `Board/Highlighting`). Split finer
  rather than mixing.
- **Exceptions:**
  - a feature's **orchestrator / entry type may sit at the feature root** above its collaborators'
    sub-folders (e.g. the top-level generator in `Generation/`, the controller in `Gameplay/`);
  - a cohesive set of **value objects that together form one model is one responsibility** — don't
    fragment it into one-type folders (`Domain/Levels`, `Domain/Geometry` stay whole).
  - A sub-folder may start with a single file when it's a genuine standalone concern expected to
    gain siblings.
- These sub-folders organize files only — they do **not** add namespace segments (§2).

## 5. The async boundary — the rule people forget

- **Async lives only in framework-coupled layers** (Infrastructure and up). The pure **Domain** and
  **Application** layers stay **synchronous** so they remain headless-testable with `dotnet test`,
  without a Unity PlayerLoop or event loop.
- In Unity, use **UniTask** (`UniTask<Result<T>>`), which is allocation-free and integrates with the
  PlayerLoop — but *because* UniTask is Unity-coupled, an async port like `ILevelProvider` belongs in
  **Infrastructure**, not Application. The Application layer never returns a `Task`/`UniTask`.
- Same principle outside Unity: keep the algorithmic core synchronous and push `async`/`Task` out to
  the I/O adapters, so the core can be tested without an async host.

## 6. Composition root

- **Manual wiring, no DI container.** A single `Composition` assembly constructs the graph:
  build the services (provider chains, colour providers, …), inject them into the controllers, own
  the entry point (`GameBootstrapper` / `Program`). Teardown is explicit (`IDisposable` drops
  subscriptions).
- Keeping composition in one place means every other layer depends only on abstractions and never
  on the concrete wiring.

## 7. Fallback chains (a reusable I/O pattern)

When a resource can come from several sources, model it as a **chain that falls back**, each source
behind the same interface:

```
Primary (remote)  →  Bundled (local shipped copy)  →  Generated / default
```

Each provider implements the same port (`ILevelProvider`); a `FallbackLevelProvider` composes them
and tries the next on failure. Thin wrappers that differ only in a base URL or root path share one
underlying helper. Decide explicitly whether there's caching (often "one fallback, no cache" is
simplest and clearest).

## 8. `.meta` hygiene (Unity)

- When you move/rename/delete a `.cs`, move/remove its `.cs.meta` too (**preserve the GUID**) and
  clean orphan `.meta` files. A folder move that only relocates files (no code change) is fine
  precisely because namespaces don't track sub-folders (§2).

---

## Checklist for a new layer / feature

1. Does it belong in Domain (pure, no framework, no exceptions), Application (pure, synchronous),
   or a framework-coupled layer? Put it in the **innermost** layer that can hold it.
2. Is there **one assembly per layer**, with references pointing only inward?
3. Is the feature **packaged together** (by feature), and is each folder **one responsibility**?
4. Does any `async` accidentally sit in Domain/Application? Move it out to Infrastructure.
5. Do fallible operations return `Result`/`Result<T>` instead of throwing (see `CONVENTIONS.md` §4)?
6. Are the pure parts covered by `dotnet`-runnable tests?
