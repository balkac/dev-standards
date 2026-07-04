# Engineering Conventions — C# / .NET

The rules I follow in every C# project, and the reasoning behind them. The mechanical ones
(braces, `var`, spacing, import order) are enforced by [`.editorconfig`](.editorconfig), which
Rider and `dotnet format` both read — so they are tool-checked, not just documented here. For
project structure (layers, assemblies, folders, the async boundary) see
[`ARCHITECTURE.md`](ARCHITECTURE.md).

> When in doubt, take Rider's / ReSharper's suggestion — `.editorconfig` is the shared baseline
> for them.

These conventions are framework-agnostic: they hold in a Unity project, an ASP.NET service, or a
plain console library.

---

## 1. Code style (enforced by `.editorconfig`)

- **Braces everywhere.** Every `if`/`for`/`foreach`/`while` uses `{ }`, even for a single
  statement. No inline bodies, no brace-less stacked loops.
- **Allman braces.** The opening brace goes on its own next line; the body follows on the next
  line:
  ```csharp
  for (int x = 0; x < size; x++)
  {
      Step(x);
  }
  ```
- **`var` when the type is apparent** from the right-hand side (`new T(...)`, casts, array
  literals). Keep the **explicit type** otherwise — e.g. a method result
  (`Coverage coverage = fill.GetCoverage();`) — and for built-in types, because it reads better.
- **Readability over cleverness / micro-optimization.** Prefer a clear structure to a terse trick:
  - Use a real collection type for an edge/pair set (`HashSet<(int, int)>`), not a hand-packed
    `long` key.
  - Don't set an enum's underlying type (`: byte`) unless there's a real reason; the default `int`
    is clearer.
  - Spell out flag values (`1, 2, 4, 8`) instead of `1 << n` when the constants read plainly.

## 2. Naming

- **One public type per file**, file name = type name. (A single small file grouping a set of DTOs
  that only exist as one payload is the rare exception.)
- **Methods are verb + object** — they say what they *do*: `GetCoverage`, `BuildFillPolygon`,
  `FindRoot`, `ChooseAdjacentRegion`, `Merge`, `EnumerateShapes`. Avoid noun-only (`Union` →
  `Merge`) and vague (`Find` → `FindRoot`) names. This holds for **every** method, including
  `private` helpers and test data-builders — a method name is never a bare noun:
  - **Computed-value helpers** take a `Get`/`Compute` prefix: `Area()` → `GetArea()`,
    `AngleFromReverse()` → `ComputeAngleFromReverse()`.
  - **Factories** use `Create`/`Build`, including test builders: `Shape(...)` → `CreateShape(...)`.
  - **Predicates** use `Is`/`Are`/`Has`/`Can`: `RegionConnected(...)` → `IsRegionConnected(...)`.
  - *Idiomatic exceptions kept:* conversions (`ToDto`, `FromDto`, `CornerToWorld`), the RNG
    `Next*` family, and `Result.Success` / `Result.Failure` factories.
  - Test **method names** keep the `Scenario_Condition_Result` form (§5) — that convention
    overrides verb+object for the `[Test]` methods themselves, not for the helpers they call.
- **Type names what it *is*; the member/parameter name carries the *role*.** If the type is
  `Coordinate`, the role lives in the member name: `Offset`, `Solution`, `origin`. Don't bake the
  role into the type name (`CellOffset` → `Coordinate`, used everywhere with role-named members).
- **Names must reflect the real algorithm / scope.** Name the concrete algorithm, not a family
  (`SplitMix64RandomNumberGenerator`, not `XorShift`); name the real granularity (`CellGeometry`,
  not `GridGeometry`, when it's per-cell); drop redundant noise (`PolygonLoop`, not `Polygon2D` —
  it's a single loop and `2D` says nothing). Avoid `Type`/`Info`/`Data` suffixes that don't say
  what varies (`CellType` → `CellShape`).
- **Descriptive variable names — no cryptic abbreviations.** Spell the intent out: `boardSize`
  not `n`, `itemCount` not `k`, `coverage` not `cov`, `neighbour` not `nb`, `rootA`/`rootB` not
  `ra`/`rb`, `bottomLeft` not `bl`. *Allowed shorthands:* `i`/`j` for loop counters and `x`/`y`
  for coordinates (domain-standard); `left`/`right` for binary-operator operands; `other` for the
  equality/`Equals` parameter.
- **`[Flags]` enums:** values are powers of two; aggregate with OR (`Full = Bottom | Right | Top |
  Left`).

## 3. Design / SOLID

- **Thin orchestrators + single-responsibility collaborators behind interfaces.** An orchestrator
  wires a sequence of small injected steps (`IPlanner` → `IPartitioner` → `IColorer` → `IBuilder`
  → `IValidator`), each swappable (OCP/DIP) and unit-tested in isolation. Small, focused
  interfaces (ISP); depend on abstractions (DIP); one responsibility per type (SRP).
- **Separate the data structure from the policy.** A generic union-find (`DisjointSet`) is
  independent of any particular "grow the smallest region" merging *policy* that uses it. The data
  structure doesn't know why it's being merged.
- **No local (nested) functions — ever.** A method must not declare another named function inside
  its body. Promote it to a `private` method instead. When several steps share working state (and
  you'd be tempted to close over locals or thread them through parameters/delegates), encapsulate
  that state in a small, single-purpose (usually `private`, often nested) class whose methods
  operate on fields. *(Inline lambda **arguments** — a comparator passed to `Array.Sort`, a LINQ
  predicate — are fine; the rule targets named local functions, not expressions.)*
- **Avoid premature abstraction (YAGNI).** Extract an interface or a helper only on a real
  single-responsibility or readability win, not "in case we need it". Keep a type next to its only
  consumer and promote it (e.g. to a shared `Common`) only when a second consumer actually appears.

## 4. Error handling & determinism

- **No `throw` in domain / application logic.** Return `Result` / `Result<T>` (a small
  cross-cutting type) and handle it at the call site. Validate defensively (e.g. clamp sizes)
  instead of throwing. Value objects trust their own internally-validated inputs.
- **Catch third-party exceptions at the boundary** (e.g. JSON parsing in a serializer, a network
  call in a provider) and convert them to `Result.Failure`. Exceptions from libraries don't leak
  past the adapter that calls the library.
- **Deterministic where it matters.** Any generation/simulation uses a **seeded RNG** — the same
  seed yields the same output. Refactors are behaviour-neutral unless a change is intended, which
  makes them testable by golden output.

A minimal `Result` shape (put it in a dependency-free `Common`/shared assembly):

```csharp
public readonly struct Result
{
    public bool IsSuccess { get; }
    public string Error { get; }
    private Result(bool ok, string error) { IsSuccess = ok; Error = error; }
    public static Result Success() => new(true, null);
    public static Result Failure(string error) => new(false, error);
}

public readonly struct Result<T>
{
    public bool IsSuccess { get; }
    public T Value { get; }
    public string Error { get; }
    private Result(bool ok, T value, string error) { IsSuccess = ok; Value = value; Error = error; }
    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);
}
```

## 5. Testing & process

- **Every collaborator / data structure gets isolated tests.** Because collaborators are small and
  behind interfaces (§3), each is unit-tested on its own.
- **Validate headless first.** Run the pure tests with `dotnet test` before opening the editor /
  running the app, and keep the suite green after every change. This only works if the core stays
  framework-free and synchronous (see [`ARCHITECTURE.md`](ARCHITECTURE.md)).
- **Test method names use `Scenario_Condition_Result`** (e.g.
  `Generate_WithSeed_IsDeterministic`) — this is the one place a method name isn't verb+object.
- **Deliverable docs, code, and comments are in English** (day-to-day chat can be in any
  language). Keep the docs in sync: update the architecture doc after any structural change.

---

## Example transformations

These are the kinds of edits the conventions produce, kept as generic before → after pairs so the
*rule* is visible without tying it to any one codebase. Use them as a checklist during review.

| Rule (§) | Before | After | Why |
|---|---|---|---|
| Errors (§4) | `throw new InvalidOperationException(...)` in core logic | return `Result.Failure("...")` | Expected failure isn't exceptional; handle at the call site. |
| SOLID (§3) | one 400-line generator method | orchestrator + `IPlanner`/`IPartitioner`/`IColorer`/`IValidator` | Each step swappable and unit-tested in isolation. |
| SOLID (§3) | `partitioner` owns its own union-find internals | extract generic `DisjointSet`; partitioner holds the *policy* | Separate the data structure from the policy that drives it. |
| SOLID (§3) | a named local function inside a method | a `private` method, or a small state-holding nested class | No local functions; shared state lives in fields, not closures. |
| Naming (§2) | `Union(a, b)` | `Merge(a, b)` | Methods are verb+object, never a bare noun. |
| Naming (§2) | `BoardArea()`, `WorldRect()` | `GetBoardArea()`, `GetWorldRect()` | Computed-value helpers take `Get`/`Compute`. |
| Naming (§2) | `RegionConnected(r)` | `IsRegionConnected(r)` | Predicates take `Is`/`Are`/`Has`/`Can`. |
| Naming (§2) | `Shape(...)`, `FullCellPiece(...)` (test builders) | `CreateShape(...)`, `CreateFullCellPiece(...)` | Factories — including test builders — use `Create`/`Build`. |
| Naming (§2) | `XorShiftRng` (wrong algorithm) | `SplitMix64RandomNumberGenerator` | Name the concrete algorithm, not a vague family. |
| Naming (§2) | `Polygon2D` (one loop) | `PolygonLoop` | Name the real granularity; drop redundant `2D` noise. |
| Naming (§2) | `CellOffset` (role baked into type) | `Coordinate` with a member named `Offset` | Type says what it *is*; the member carries the *role*. |
| Naming (§2) | `n`, `k`, `cov`, `ra`/`rb`, `bl` | `boardSize`, `itemCount`, `coverage`, `rootA`/`rootB`, `bottomLeft` | Descriptive names; keep only `i`/`j`, `x`/`y`, `left`/`right`, `other`. |
| Readability (§1) | `enum Coverage : byte { A = 1 << 0, ... }` | `enum Coverage { A = 1, B = 2, C = 4, D = 8 }` | Default `int` and explicit flag values read more plainly. |
| Style (§1) | brace-less single-statement `if`/`for` | braces on every block, Allman | Uniform, diff-friendly, enforced by `.editorconfig`. |
