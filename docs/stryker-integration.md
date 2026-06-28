# Stryker compatibility notes

`cxx-mutant` does not depend on the Stryker runtime. It is a Python mutation
runner for C++ sources that emits a projection matching the same shape as
`mutation-testing-elements`, which keeps future interoperability realistic.

## What is projected today

- `mutationTestingElements.schemaVersion = 2.0`
- per-file sources and mutant entries in `files[file]["mutants"]`
- status mapping:
  - `KILLED` -> `Killed`
  - `SURVIVED` -> `Survived`
  - `BUILD_ERROR` -> `CompileError`
  - `TIMEOUT` -> `TimedOut`
  - `PENDING` -> `Pending`

## Why this is enough for a stryker add-on

This gives a stable bridge for:

- downstream status dashboards that already read `mutation-testing-elements`;
- deterministic mutant IDs and ranges in reports;
- non-breaking extension path to richer AST metadata.

The Stryker add-on question for this repo is therefore mainly packaging and
viewer integration, not proof-of-concept parser rewrites.

