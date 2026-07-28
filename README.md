# Engineering Standards

A portable, project-agnostic set of engineering conventions I carry from one project to the
next, so every codebase reads and behaves the same way. Nothing here is tied to a specific
project — copy the whole folder into a new repo and start.

## What's in here

| File | Scope | What it is |
|---|---|---|
| [`CONVENTIONS.md`](CONVENTIONS.md) | **Any C# / .NET project** | Code style, naming, SOLID-in-the-small, error handling, testing & process. Applies with or without Unity. |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | **Unity (and layered apps)** | Clean-architecture playbook: layers as assemblies, package-by-feature, one-responsibility-per-folder, the async boundary, `Result` instead of `throw`, command→outcome boundary, behavioural seams. |
| [`PERFORMANCE.md`](PERFORMANCE.md) | **Unity** | Keeping it smooth and low-garbage: light-by-construction, pooling contracts, budgets from a config asset, C# allocation mechanics, Boehm GC, API traps, device measurement, mobile rendering/build, procedural meshes, Addressables, canvas/TMP, memory verification. |
| [`UNITY.md`](UNITY.md) | **Unity** | Engine correctness: lifecycle/initialisation order, frame & time semantics, enable/disable/destroy, mobile app lifecycle, events & teardown, async/threading, persistence traps. |
| [`GAME-FEEL.md`](GAME-FEEL.md) | **Unity / casual games** | The design-side standards: reference-pattern research, FTUE engineering, authored motion, input feel, budgeted juice — design defaults to validate by playtest. |
| [`starter-tree.md`](starter-tree.md) | **Unity** | An empty layered skeleton (assemblies + folders) to copy and start from, plus the `dotnet test` bridge for the pure layers. |
| [`COMMIT-CONVENTIONS.md`](COMMIT-CONVENTIONS.md) | **Any repo** | Conventional Commits adapted: required types, tree-derived scopes, imperative subjects, why-bodies, breaking-change footers. |
| [`.editorconfig`](.editorconfig) | **Any C#** | The mechanical rules (braces, `var`, Allman, import sorting), **tool-enforced** — Rider and `dotnet format` both read it. |
| [`.gitattributes`](.gitattributes) | **Unity** | Unity line-ending/merge template + the "store binaries inline, not in LFS" decision (see the note in the file). |

The split is deliberate: `CONVENTIONS.md` is pure code discipline and holds even in a plain
console library; everything Unity- or structure-specific lives in `ARCHITECTURE.md` and the
skeleton.

## Using it in a new project

1. **Copy the mechanical enforcers to the repo root** so the IDE picks them up:
   ```
   cp engineering-standards/.editorconfig   <new-project>/.editorconfig
   cp engineering-standards/.gitattributes  <new-project>/.gitattributes    # Unity projects
   ```
   `.editorconfig` starts working immediately in Rider / VS / `dotnet format` — the style rules
   below are then enforced, not just documented.

2. **Keep the docs where the team reads them.** Either copy `CONVENTIONS.md` (+ the Unity set:
   `ARCHITECTURE.md`, `PERFORMANCE.md`, `UNITY.md`, `GAME-FEEL.md`, `starter-tree.md`) into the
   project's `Docs/` folder, or link to this repo from the project README.
   If you copy them, fix the relative link at the top of `.editorconfig`'s comment to point at
   wherever `CONVENTIONS.md` landed.

3. **Scaffold from `starter-tree.md`** (Unity) — create the layer folders and `.asmdef`s first,
   with dependencies pointing inward, before writing any feature code.

4. **Read `ARCHITECTURE.md` §"The async boundary" before adding async** — it's the rule people
   forget, and it's what keeps the core headless-testable.

## The one-paragraph version

Braces everywhere, Allman style, `var` only when the type is already apparent. One public type
per file, methods are **a verb or a verb phrase**, names say what a thing *is* (type) vs the
*role* it plays (member). Clean architecture with dependencies pointing inward, one assembly per
layer, package **by feature not by technical layer**, one responsibility per folder. The pure
core (Domain + Application) has **no framework dependency and no async** — expected failure
returns `Result` / `Result<T>` (fail-fast `throw` stays for broken invariants); async lives only
in the framework-coupled layers so the core stays headless-testable. Every collaborator gets
isolated tests; validate headless before opening the editor. Deliverable docs and code comments
are in English.
