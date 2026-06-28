# cxx-mutant

`cxx-mutant` is a standalone source-level mutation tester for C++/ObjC++/Metal.

This repository starts from the existing `marmorkrebs` embedded C++ source engine
(`engines/cxx-source/marmorkrebs-cxx.py`) and preserves token-level behavior for
PR-sized mutation gates.

It is intended to be a standalone project and can be imported by other mutation
workflows as a stable CLI.

## Quick start

```bash
pip install -e .

cxx-mutant run \
  --repo . \
  --files src/foo.cpp,src/bar.mm \
  --base origin/main \
  --build-command "ninja -C build target" \
  --test-command "./build/bin/target_test" \
  --report mutation.json
```

Useful options:

- `--max-mutants <n>`
- `--include-metal`
- `--mutators <names>`
- `--include <glob>` / `--exclude <glob>` to filter configured files
- `--base <ref>` for git-diff scoping
- `--lines <ranges>` for manual line targeting
- `--config <path>` for YAML/JSON defaults (`cxx-mutant.yml` / `.cxx-mutant.yml`)
- `--timeout <seconds>` per-mutant timeout
- `--threshold <0-1>` default is 1.0
- `--fail-on-empty` exits with code 3 when no mutants are generated
- `--format json|markdown|html|sarif`
- `--resume <report>` resumes from previous completed run
- `--artifact-dir <path>` for build/test logs
- `--config <path>` for YAML/JSON defaults
- `--quiet` for reduced output
- `--mode token|clang` (`--mode clang` requires libclang bindings and parse-backed discovery; use `--mode token` when unavailable)
- `--jobs <n>` (parallel mutant execution across isolated worktree workers)
- `--worktree-mode inplace|git-worktree|copy` (`git-worktree` and `copy` create isolated per-mutant workspaces)
- `--allow-dirty` to opt out of inplace clean-tree checks

## Repository notes

- Source layout: `src/cxx_mutant/engine.py` is execution + reporting; `src/cxx_mutant/cli.py` is the command surface.
- For migration from embedded Marmorkrebs, see [`docs/migration-from-marmorkrebs.md`](docs/migration-from-marmorkrebs.md).
- For Stryker-facing projection details, see [`docs/stryker-integration.md`](docs/stryker-integration.md).

## Compatibility with Marmorkrebs

`marmorkrebs` can invoke this tool via `--cxx-mutant-bin` while keeping the same
`--tool cxx-source` flags:

```bash
marmorkrebs --dir . --tool cxx-source --base origin/main \
  --build-command "ninja -C build target" \
  --test-command "./build/bin/target_test" \
  --cxx-mutant-bin cxx-mutant
```

## Reporting

`cxx-mutant` outputs both the legacy embedded `cxx-source` schema and a
`cxx-mutant.report.v1` projection in compatibility mode (used by Marmorkrebs
parser already).

## Commands

- `run` — discover mutants, execute build/test, and write report
- `list-mutants` — enumerate mutant payloads without executing commands
- `run-mutant` — run a single mutant by stable ID

### Config and status output

The parser emits `cxx-mutant.report.v1` by default; legacy compatibility remains
available through `--output-format legacy`.

```bash
cxx-mutant run \
  --repo . \
  --files src/foo.cpp \
  --build-command "ninja -C build target" \
  --test-command "./build/bin/target_test" \
  --report results/mutation.json \
  --format markdown
```

### Design notes

See [`docs/cxx-mutant-spec.md`](docs/cxx-mutant-spec.md) for acceptance criteria and
current M0–M2 status, including planned M3 clang mode and isolation model.
