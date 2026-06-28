# cxx-mutant standalone tool spec

## Purpose

`cxx-mutant` is the independent C/C++/ObjC++/Metal mutation engine for PR-sized mutation
quality gates. It must remain lightweight and callable from existing CI while exposing a
Stryker-style reporting seam (`mutation-testing-elements` v2.0) so downstream tooling
can build full mutation dashboards without engine rewrites.

The end goal is a **Stryker-grade external provider**: deterministic mutant discovery,
reliable per-mutant execution, stable IDs/reproducers, and stable contracts for humans,
CI, and host integrations.

## Definition of completion (Stryker-level target)

The spec is complete when all statements below are true in docs, implementation, and consumer
surfaces:

1. The engine is stable as a CLI tool independent from Marmorkrebs internals.
2. Mutant discovery and execution are deterministic and reproducible from explicit inputs.
3. Machine contracts are versioned, complete, and schema-stable.
4. Mutation payloads can be consumed by a `stryker-cxx`-style host without format shims.
5. The repo has a production-quality command contract for run/listing/replay.
6. CI-friendly failure handling exists for build failures, timeouts, and infra errors.
7. A migration path from embedded Marmorkrebs output remains intact.

## Current implementation status (2026-06-29)

- [x] Extracted engine into standalone package with `run`, `list-mutants`, `run-mutant`.
- [x] Stable mutant IDs and deterministic token-mode discovery.
- [x] Config files (`cxx-mutant.yml` / `.cxx-mutant.yml`) for paths, mutators, execution, and report settings.
- [x] Resume (`--resume`), `--fail-on-empty`, and machine-readable output formats.
- [x] Inplace dirty-tree guard with `--allow-dirty`.
- [x] `--mode token|clang` with parse-backed discovery and `compile_commands.json` support.
- [x] Isolated `git-worktree` and `copy` execution modes.
- [x] Parallel execution (`--jobs` > 1) in isolated modes.
- [x] Repro-command capture on every executed mutant (`reproCommand`).
- [x] `mutation-testing-elements` projection (`schemaVersion: 2.0`) with:
  - stable `location.start`/`location.end`
  - Stryker-like statuses and `TimedOut`
  - mutator `description` and `runCommand`
  - `projectRoot`, `language` fields.
- [x] Basic Stryker migration docs and Marmorkrebs compatibility.
- [ ] Full Stryker parity: test-runner integration, selector/runner orchestration, mutation operators parity,
  equivalent-mutant filtering, richer skip/ignore controls.

## Scope and non-goals

`cxx-mutant` is an operator, not a full compiler integration platform. Initial
non-goals stay in place so Stryker parity lands incrementally without overcommitting:

- Not a full LLVM-IR mutation engine on day one.
- No built-in test-sharding/scheduler beyond simple file/mutant partitioning.
- No implicit claims of semantic equivalence from survivors.
- No mutation of macros/generated files/vendored trees unless explicitly enabled.
- No hidden coupling with Marmorkrebs command objects or runtime internals.

## Baseline behavior inheritance from Marmorkrebs

The original embedded behavior is still the compatibility floor:

- Diff-scoped discovery against `git diff --unified=0 <base>`.
- Caller-controlled source paths and line scoping.
- One-mutant-at-a-time execution.
- Build command and test command run on each mutant.
- Native statuses: `KILLED`, `SURVIVED`, `BUILD_ERROR`, `TIMEOUT`, `PENDING`.

These behaviors must stay stable while higher-level contracts harden.

## CLI contract (authoritative)

The public contract for each command is below. All flags in examples can be overridden via
config, and CLI options always win.

### `run`

```bash
cxx-mutant run \
  --repo . \
  --files src/foo.cpp,src/bar.mm \
  --base origin/main \
  --build-command "ninja -C build target" \
  --test-command "./build/bin/target_test" \
  --report results/mutation.json
```

Required:

- `--repo <path>`: repo root to mutate and execute commands.
- `--files <paths>`: comma-separated repo-relative files to target.
- `--build-command <cmd>` and `--test-command <cmd>`: executable commands for build and test.

Scoping:

- `--base <ref>`: only changed lines versus a git ref.
- `--lines <ranges>`: manual line filters (`10-20,40`).
- `--include <glob>`, `--exclude <glob>`: include/exclude file patterns.

Mutation and mode:

- `--mutators <names>`: comma-separated mutator names.
- `--max-mutants <n>`: cap total run candidate count.
- `--include-metal`: include `.metal` when token-mode is selected.
- `--mode token|clang`: discovery path (`token` default).

Execution controls:

- `--timeout <seconds>`: per-mutant timeout (applies to build and test commands).
- `--jobs <n>`: enable parallel execution in `copy`/`git-worktree` modes.
- `--worktree-mode inplace|git-worktree|copy`:
  - `inplace` for compatibility,
  - isolated modes for parallelism.
- `--shard-index <i> --shard-total <n>`: partition mutant set deterministically.
- `--resume <report>`: continue from completed mutants in prior report.
- `--artifact-dir <path>`: custom artifact root for build/test logs.
- `--allow-dirty`: explicit allow-override of dirty-file guard.

Output:

- `--report <path>`: primary report path (JSON payload).
- `--format json|markdown|html|sarif|mutation-testing-elements`:
  - `json` writes `cxx-mutant.report.v1`.
  - `mutation-testing-elements` writes MTE-only payload in `mutationTestingElements` shape.
- `--output-format legacy|cxx-mutant`: legacy compatibility projection vs native schema.
- `--quiet`: suppress progress chatter.

Behavioral invariants:

- `--jobs > 1` is invalid in `inplace` mode.
- Non-zero build/test outcomes are recorded as mutant-level infrastructure statuses and do not hide
  build errors from score calculations.

### `list-mutants`

```bash
cxx-mutant list-mutants \
  --repo . \
  --files src/foo.cpp \
  --base origin/main \
  --format json
```

This returns a deterministic mutant inventory with stable IDs but does not run build/test.

List payload requirements:

- JSON array of objects with at least:
  - `id`, `file`, `line`, `column`, `mutator`, `original`, `mutated`.
- `nodeKind` and `mode` may be included for implementation detail.

### `run-mutant`

```bash
cxx-mutant run-mutant \
  --repo . \
  --id src/foo.cpp:42:17:EqualityOperator:abc123 \
  --build-command "ninja -C build target" \
  --test-command "./build/bin/target_test" \
  --report results/one_mutant.json
```

Single-mutant execution must resolve the mutant by stable ID from the same discovery rules
as `run` with the same inputs.

## Exit codes (required)

- `0`: run completed and met threshold.
- `1`: configuration/runtime fault.
- `2`: run completed, but quality threshold unmet.
- `3`: zero mutants and `--fail-on-empty` enabled.

All commands should preserve non-zero codes and print machine-parseable context in debug logs.

## Contracts and schemas

### `cxx-mutant.report.v1` (canonical)

```json
{
  "schemaVersion": "cxx-mutant.report.v1",
  "tool": "cxx-mutant",
  "repo": "/path/to/repo",
  "base": "origin/main",
  "startedAt": "2026-06-28T12:00:00Z",
  "completedAt": "2026-06-28T12:05:00Z",
  "score": 0.83,
  "threshold": 0.8,
  "totalMutants": 6,
  "killed": 5,
  "survived": 1,
  "buildErrors": 0,
  "timeouts": 0,
  "execution": { "mode": "token", "worktreeMode": "copy", "jobs": 2 },
  "commands": { "build": "ninja -C build target", "test": "./build/bin/target_test" },
  "mutants": [
    {
      "id": "src/foo.cpp:42:17:EqualityOperator:abc123",
      "file": "src/foo.cpp",
      "line": 42,
      "column": 17,
      "mutator": "EqualityOperator",
      "original": "==",
      "mutated": "!=",
      "status": "SURVIVED",
      "durationMs": 1200,
      "buildLog": "agent_space/cxx-mutant/build_1.log",
      "testLog": "agent_space/cxx-mutant/test_1.log",
      "detail": "all targeted tests passed",
      "nodeKind": "",
      "run": {
        "reproCommand": "cxx-mutant run-mutant ..."
      }
    }
  ]
}
```

Compatibility notes:

- Scores are `0.0..1.0`.
- `totalMutants: 0` must always be emitted explicitly.
- `buildErrors` and `timeouts` are separate counters and do not raise score implicitly.
- `legacy` compatibility fields may exist for backwards compatibility but are secondary.

### Stryker-compatible projection (`mutation-testing-elements`)

`cxx-mutant` must emit this projection when `--format mutation-testing-elements` is requested:

- Top-level keys:
  - `schemaVersion = "2.0"`
  - `files`
  - `testFiles`
  - `language = "cpp"`
  - `projectRoot`
- Per mutant payload:
  - `id`, `mutatorName`, `description`, `original`, `replacement`
  - `status` in Stryker domain (`Killed|Survived|CompileError|TimedOut|Pending|RuntimeError`)
  - `statusReason`, `nodeKind`, `runCommand`
  - `location.start` / `location.end`

Status mapping from engine statuses:

- `KILLED` -> `Killed`
- `SURVIVED` -> `Survived`
- `BUILD_ERROR` -> `CompileError`
- `TIMEOUT` -> `TimedOut`
- `PENDING` -> `Pending`
- all other statuses -> `RuntimeError`

This is the contract that a dedicated host such as a future `stryker-cxx` package should
consume directly.

## Core feature requirements for Stryker-level acceptance

To justify “Stryker-level” claims, the project must satisfy these core requirements:

- Determinism
  - Stable mutant ordering.
  - Stable mutant IDs across identical inputs.
  - Stable output ordering for humans and consumers.
- Reproducibility
  - `reproCommand` must exist for each executed mutant.
  - Report payload must include build/test command context.
- Report compatibility
  - v1 payload and MTE projection both present and parseable.
  - `language`, `projectRoot`, and source spans are always populated when possible.
- Isolation controls
  - Safe isolated execution modes and explicit no-destructive cleanup.
  - Resume semantics for interrupted runs.
- Governance
  - Config-driven defaults with CLI overrides.
  - Documented non-goals and supported mutator set.

## Mutator set and semantics

Built-in mutators:

- `ConditionalBoundary`: boundary flips (`<` <-> `<=`, `>` <-> `>=`).
- `EqualityOperator`: `==` <-> `!=`.
- `LogicalOperator`: `&&` <-> `||`.
- `BooleanLiteral`: `true` <-> `false`.
- `ArithmeticOperator`: `+ - * /` swaps.
- `AssignmentOperator`: `+= -= *= /=` swaps.
- `BitwiseOperator`: `& | ^` swaps.
- `UnaryOperator`: `!` removal/addition.
- `ReturnValue`: `return true` <-> `return false`.

Default production profile:

- `ConditionalBoundary,EqualityOperator,LogicalOperator,BooleanLiteral`

Extension policy:

- No opt-in mutator is allowed without explicit default/opt-in documentation.
- AST-derived mutator additions should include equivalent risk/benefit notes and
  acceptance tests.

## Discovery modes and AST trust boundary

### Token mode (baseline)

Requirements:

- Ignore comments and string/character literals.
- Keep deterministic traversal.
- Default include list handles C-family extensions (`.cpp`, `.cc`, `.cxx`, `.c`, `.mm`, `.m`, `.h`, `.hpp`, `.hh`, `.hxx`).
- `.metal` gated by `--include-metal`.
- No template or punctuation false-positives from common token edge cases.
- Exclusion of include/preprocessor lines by default where feasible.

### Clang mode (roadmap target)

Requirements:

- Resolve compile options from `compile_commands.json` when available.
- Prefer AST-token confirmation for discovered mutations.
- Record token/AST context (`nodeKind`) in mutant records.
- Avoid macro expansion by default.

Current status: operator token discovery in clang mode is implemented with libclang-backed discovery.

## Execution model, status mapping, and safety

Per mutant:

1. Apply one mutation.
2. Run build command.
3. If build succeeds, run test command.
4. Restore original source.

Status resolution:

- build command timeout -> `TIMEOUT`.
- build command non-zero exit -> `BUILD_ERROR`.
- test command timeout -> `TIMEOUT`.
- test command non-zero -> `KILLED`.
- test command zero -> `SURVIVED`.

Workspace behavior:

- `inplace`: mutates in place and restores with tracked-file cleanup guard.
- `git-worktree`/`copy`: per-mutant isolated work directories.
- Logs are written under `artifact-dir` or `agent_space/cxx-mutant`.
- `--resume` skips completed mutant IDs from a prior report.

## Configuration schema

`cxx-mutant.yml` / `.cxx-mutant.yml` supports:

```yaml
schemaVersion: cxx-mutant.config.v1
base: origin/main
files:
  include:
    - "aten/src/**/*.cpp"
    - "aten/src/**/*.mm"
  exclude:
    - "**/generated/**"
mutators:
  enabled:
    - ConditionalBoundary
    - EqualityOperator
    - LogicalOperator
    - BooleanLiteral
execution:
  buildCommand: "ninja -C build target"
  testCommand: "python test/run_test.py test_mps --keep-going"
  timeoutSeconds: 300
  jobs: 2
  worktreeMode: copy
  mode: clang
  sharded:
    index: 1
    total: 4
report:
  threshold: 0.6
  failOnEmpty: false
  artifactDir: agent_space/cxx-mutant
```

Rules:

- CLI args always override config.
- `files` string shorthand and `files.include`/`files.exclude` are both accepted.
- Unknown configuration keys should be ignored with warning, not hard-fail.

## Human-readable reporting

JSON is canonical; markdown/html/sarif are secondary views:

- Markdown summary includes score, thresholds, mutators, survivors, build/error counts, commands.
- HTML/SARIF are useful transport contracts and can be added/extended incrementally.

## Marmorkrebs contract

Keep this adapter contract stable:

- `marmorkrebs` continues to invoke external provider when configured.
- Legacy behavior remains functional via `--output-format legacy`.
- Mapping is currently:
  - `totalMutants` -> `MutationResult.totalMutants`
  - `killed` -> `MutationResult.killed`
  - `survived` -> `MutationResult.survived`
  - `buildErrors` -> `MutationResult.noCoverage`
  - `timeouts` -> `MutationResult.timeout`
  - `score` -> `MutationResult.score`
  - `surviving mutants` -> `MutationResult.survivingMutants`

## Repository layout (current + planned)

```text
cxx-mutant/
  README.md
  LICENSE
  pyproject.toml
  src/cxx_mutant/
    __init__.py
    cli.py
    engine.py
  docs/
    cxx-mutant-spec.md
    migration-from-marmorkrebs.md
    stryker-integration.md
    mutators.md
```

Planned additions for full Stryker-level parity:

- Dedicated acceptance tests for discovery/restore/report contracts.
- Negative/edge fixtures for parser and timeout behavior.
- CI workflow, package publication metadata, and plugin compatibility examples.

## Milestones

### M0: extraction without behavior regression

- External packaging + stable entrypoint.
- Baseline token-mode parity with historical behavior.
- Marmorkrebs compatibility path preserved.

### M1: production CLI

- `list-mutants`, `run-mutant` and resume.
- Deterministic IDs and command-level reproducibility.
- `--format mutation-testing-elements` projection available.

### M2: isolation and safety

- `git-worktree`/`copy` support.
- Artifact-root logging and dirty-tree safety.
- Sharding and thresholded exits.

### M3: structural trust

- Hardened clang mode with compile-database fallbacks.
- AST node kind metadata.
- Stable location precision suitable for MTE consumers.

### M4: stryker ecosystem parity

- Full host-ready payload stability for a future `stryker-cxx` package.
- Repro-command and source mapping completeness.
- Clear equivalent-mutant policy and reporting of infra states.

### M5: CI-grade hardening

- Repository-level CI and release pipeline.
- Automated schema validation for `cxx-mutant.report.v1` and MTE payload.
- Plugin/adapter integration proof across at least one external repo.

## Acceptance criteria for independent release

The repo is treated as independent when all checks are demonstrably true:

- Executes standalone from CLI + config on a clean checkout.
- Emits both canonical report and MTE projection.
- Deterministic replay via `run-mutant`.
- Resume and shard semantics are deterministic and documented.
- `timeout`, `build_error`, `killed`, `survived`, and `time out` are observable in all reports.
- A future `stryker-cxx` host can consume projection without custom schema transforms.

## Open questions

- Do we add a dedicated shader-specific mode (`--mode metal`) or keep token-mode default.
- How much AST-context filtering is needed before enabling heavier mutator families.
- Whether artifact upload/report formats should include a per-mutant git patch diff.
- Where to place threshold policy when both adapter and tool are in play.
