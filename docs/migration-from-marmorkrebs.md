# Migration: moving C++ source mutation from Marmorkrebs

`cxx-mutant` is now an independent mutation testing CLI so the C++ source engine can
evolve without coupling to Marmorkrebs internals.

## Drop-in path from existing `cxx-source` usage

1. Keep your existing PR-gate flow in place.
2. Install or expose this binary (`pip install -e .` for local source or release package).
3. Point Marmorkrebs at it via `--cxx-mutant-bin`:

```bash
marmorkrebs --tool cxx-source --dir . --base origin/main \
  --build-command "ninja -C build tests" \
  --test-command "./build/bin/tests" \
  --cxx-mutant-bin cxx-mutant
```

This preserves the current behavior while allowing `cxx-mutant` to ship independently.

## Direct invocation (when not via Marmorkrebs)

```bash
cxx-mutant run \
  --repo . \
  --files src/foo.cpp \
  --base origin/main \
  --build-command "ninja -C build tests" \
  --test-command "./build/bin/tests" \
  --report mutation.json \
  --format markdown
```

## Output compatibility

`cxx-mutant` writes:

- legacy-compatible fields (`target_files`, `total`, `build_error`) for existing
  Marmorkrebs report consumers;
- explicit `cxx-mutant.report.v1` fields for tooling migration;
- a `mutationTestingElements` projection with Stryker-style statuses.

## Repo strategy

This repository is intentionally small and focused:

- `engine.py` owns execution and report formats;
- `cli.py` is a stable entrypoint for run/list/run-mutant;
- no dependency on Marmorkrebs runtime objects or command formats.

