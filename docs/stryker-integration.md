# Stryker compatibility notes

`cxx-mutant` does not depend on the Stryker runtime. It emits a report projection
compatible with `mutation-testing-elements` (MTE) so that a downstream Stryker add-on
can consume deterministic C++ mutants without changing this engine.

## MTE contract in this release

Use:

```bash
cxx-mutant run \
  --repo . \
  --base origin/main \
  --files src/foo.cpp \
  --build-command "ninja -C build target" \
  --test-command "./build/bin/target_test" \
  --report results/mutation.json \
  --format mutation-testing-elements
```

Contract details:

- Top-level:
  - `mutationTestingElements.schemaVersion = "2.0"`
  - `mutationTestingElements.files`: map keyed by path.
  - `mutationTestingElements.testFiles`: emitted as `{}` in the engine today.
  - `mutationTestingElements.language = "cpp"`
  - `mutationTestingElements.projectRoot` is the repository root.
- Per-file:
  - `source`: full source text when discoverable.
  - `mutants`: ordered list of mutants.
- Per-mutant:
  - `id` (stable engine ID)
  - `mutatorName` (e.g., `EqualityOperator`)
  - `description` (operator transformation language)
  - `original`
  - `replacement`
  - `status` (`Killed`, `Survived`, `CompileError`, `TimedOut`, `Pending`, `RuntimeError`)
  - `statusReason`
  - `nodeKind` (clang token/AST kind when available)
  - `runCommand` (single command used to re-run this mutant)
  - `location.start` / `location.end` line+column spans

Status mapping from engine statuses:

- `KILLED` -> `Killed`
- `SURVIVED` -> `Survived`
- `BUILD_ERROR` -> `CompileError`
- `TIMEOUT` -> `TimedOut`
- `PENDING` -> `Pending`

All other statuses map to `RuntimeError`.

## Design goal

The engine intentionally stays parser-agnostic:

- `cxx-mutant` remains a stand-alone C++ mutation generator and runner.
- This projection keeps the door open for a dedicated `stryker-cxx` runner later.
- Stryker-specific orchestration (grouping, reporters, dashboard wiring) can be built
  without rewriting discovery or execution semantics in this engine.
  `stryker-cxx` is the preferred boundary for this layer so engine compatibility with
  Marmorkrebs and other callsites stays intact.
