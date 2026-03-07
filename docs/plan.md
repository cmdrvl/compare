# compare

This is the implementation-direction plan for the repository.

`docs/PLAN_COMPARE.md` captures the earlier feature-oriented framing inherited
from the broader spine plan. Where the two documents differ, this file should
govern repository structure, protocol boundaries, and the relationship to `rvl`
and factory/tournament workflows.

## Core definition

`compare` is the epistemic spine's exhaustive tabular delta primitive.

Given two bound relations, an alignment mode, and an equivalence policy,
`compare` deterministically emits the full change surface:

- rows added
- rows removed
- rows modified
- every changed field for every changed row

`compare` does not rank, compress, interpret, or score those changes.

## Why this repo exists

The spine needs a primitive that answers:

- exactly what changed
- where it changed
- under which alignment semantics
- under which equivalence semantics

This is the audit/debug substrate beneath higher-level interpretation.

`rvl` is not a replacement for `compare`.
`rvl` is the materiality analysis over a compatible change surface.

## Hard decisions

### 1. `compare` is a protocol primitive, not just a CSV tool

The concept is relation diff, not file diff.

Files are one way of binding relations at execution time. v0 may begin with CSV
bindings, but the protocol should not hard-code "CSV diff" as the concept.

### 2. `compare` defines the exhaustive change surface; `rvl` explains it

This is the core architectural relationship:

- `shape` decides whether comparison is coherent
- `compare` enumerates the full delta surface
- `rvl` explains the materially important subset of that surface

That does not mean `rvl` must literally execute by consuming a compare report.
It does mean both tools must share the same alignment and equivalence semantics,
or the spine will drift.

### 3. Exhaustive means exhaustive after declared equivalence normalization

The current spine framing says `compare` reports raw strings, but profiles also
carry equivalence rules. That is under-specified.

The correct model is:

- equivalence rules decide whether two values count as changed
- if a value counts as changed, the report may preserve raw values and
  canonicalized values

`compare` should not report formatting noise that the declared equivalence policy
explicitly says to ignore.

### 4. One pairwise primitive; longitudinal views are derived, not core

The core primitive is pairwise exhaustive diff between two relations.

Longitudinal lifecycle analysis across N snapshots may be valuable later, but it
is not the core identity of this repo and should not shape the base protocol.
If implemented, it should be a later layer built on repeated pairwise/temporal
delta semantics, not the center of v0.

### 5. `compare` is not a gate and not a scorer

`compare` answers "what changed?"

It does not answer:

- whether the datasets were structurally comparable in the first place
- whether the changes are material
- whether the new output is correct against ground truth
- whether the pipeline should proceed

Those belong to:

- `shape`
- `rvl`
- `benchmark`
- `assess`

## Non-goals

`compare` will not:

- perform structural compatibility checks beyond what its alignment mode requires
- determine materiality
- validate invariants
- rank candidates in tournaments
- become a general history analytics framework in v0

## Repo shape

Initial repository layout:

```text
compare/
├── docs/
│   ├── PLAN_COMPARE.md
│   └── plan.md
├── schemas/
│   ├── compare.report.v1.schema.json
│   └── compare.delta.v1.schema.json
├── fixtures/
│   ├── inputs/
│   └── reports/
├── crates/
│   ├── compare-core/
│   ├── compare-engine/
│   ├── compare-bindings/
│   └── compare-cli/
└── Cargo.toml
```

### `compare-core`

Owns protocol types:

- alignment mode
- equivalence policy surface used by the diff engine
- delta item types
- report artifact types
- refusal types

### `compare-engine`

Owns deterministic exhaustive diff:

- row alignment
- field comparison
- change classification
- stable ordering
- summary aggregation

### `compare-bindings`

Owns relation binding from real inputs:

- CSV bindings in v0
- later expansion to other relation sources if needed

### `compare-cli`

Owns command-line surface:

- `run`
- `validate`
- `--describe`
- `--schema`

It should stay thin.

## Core artifacts

### `compare.delta.v1`

This is the atomic change model.

It defines the units of exhaustive change independent of human presentation.

Minimum item shapes:

```json
{
  "type": "modified",
  "row_id": "LN-00421",
  "column": "balance",
  "old_raw": "1234567.89",
  "new_raw": "1234000.00",
  "old_canonical": "1234567.89",
  "new_canonical": "1234000.00"
}
```

```json
{
  "type": "added",
  "row_id": "LN-99001",
  "column": "balance",
  "old_raw": null,
  "new_raw": "500000.00",
  "old_canonical": null,
  "new_canonical": "500000.00"
}
```

This atomic model is important because:

- it keeps pairwise change semantics explicit
- it gives `rvl` a stable conceptual substrate
- it avoids coupling the protocol to a specific grouped JSON presentation

### `compare.report.v1`

This is the user-facing exhaustive diff artifact.

Minimum shape:

```json
{
  "version": "compare.report.v1",
  "outcome": "CHANGES",
  "profile_id": null,
  "profile_sha256": null,
  "input_verification": null,
  "bindings": {
    "old": { "source": "nov.csv", "content_hash": "sha256:..." },
    "new": { "source": "dec.csv", "content_hash": "sha256:..." }
  },
  "alignment": {
    "mode": "key",
    "key_columns": ["loan_id"]
  },
  "equivalence": {
    "trim_strings": true,
    "float_decimals": 2,
    "order": "order_invariant"
  },
  "summary": {
    "rows_added": 51,
    "rows_removed": 33,
    "rows_modified": 142,
    "rows_unchanged": 4008,
    "cells_changed": 387
  },
  "changes": [
    {
      "type": "modified",
      "row_id": "LN-00421",
      "fields": [
        {
          "column": "balance",
          "old_raw": "1234567.89",
          "new_raw": "1234000.00",
          "old_canonical": "1234567.89",
          "new_canonical": "1234000.00"
        }
      ]
    }
  ],
  "refusal": null
}
```

### Important report rules

1. The report records the actual equivalence policy used.
2. Change ordering is deterministic.
3. Rows are grouped for readability, but the underlying semantics are atomic
   delta items.
4. Raw values may be preserved for audit, but change detection is based on
   canonical equivalence, not display formatting.

## Alignment model

`compare` needs an explicit alignment model because that is where epistemic
meaning enters the diff.

### `key` alignment

Use when rows represent entities and a stable key exists.

This is the default mode for relational datasets and the mode that matters for
loan tapes, canonical outputs, and most factory artifacts.

### `row_order` alignment

Use when order itself is part of the artifact's meaning.

This mode is weaker epistemically for relational tables because row insertion,
deletion, or sorting can create large noisy diffs. It should remain available,
but the report must make the chosen alignment mode explicit so downstream tools
and humans understand what kind of difference they are seeing.

## Equivalence model

This is the most under-specified part of the current plan and must be fixed.

The diff engine needs one answer to:

`Do these two cells count as different under the declared comparison policy?`

At minimum, equivalence should support:

- string trimming
- null normalization
- order sensitivity for row-order mode
- decimal precision / canonical formatting for numeric-looking values

If the equivalence policy declares two cells equal, they do not appear in the
diff.

If they differ, the report may include both raw and canonical values.

## CLI shape

### Primary command

```text
compare run <OLD> <NEW> [OPTIONS]
```

Options:

- `--key <COLUMN>`
- `--profile <PATH>`
- `--profile-id <ID>`
- `--lock <LOCKFILE>` repeatable
- `--max-rows <N>`
- `--max-bytes <N>`
- `--delimiter <DELIM>`
- `--json`

The CLI may support a compatibility form without the explicit `run` subcommand
later, but the protocol plan should be subcommand-shaped.

### Validation and discovery

```text
compare validate
compare --schema
compare --describe
```

## Relationship to `rvl`

This relationship should remain fixed:

- `compare` owns exhaustive delta semantics
- `rvl` owns materiality semantics

`rvl` should conceptually operate on the same aligned and equivalence-normalized
change surface that `compare` defines.

That gives the spine a coherent stack:

1. `shape`: can these be compared?
2. `compare`: what changed?
3. `rvl`: what materially mattered?
4. `verify`: do the candidate values satisfy declared constraints?
5. `assess`: should we proceed?

## Factory role

`compare` is not central to the factory runtime the way `verify` is.

Its role in the factory is secondary but still real:

- inspect exact differences between twin snapshots
- debug regressions between candidate runs
- compare pre- and post-decode exported states
- attach exhaustive change evidence to packs when needed

It should not be the live constraint engine or the tournament scorer.

## Tournament role

In tournament terms, `compare` is a debugging microscope, not the judge.

Use it for:

- explaining why two candidate outputs differ
- drilling into regressions after benchmark or verify failures
- producing exhaustive evidence for humans reviewing close calls

Do not use it for:

- primary ranking
- correctness scoring
- policy gating by itself

Those belong to `benchmark`, `verify`, and `assess`.

## Build order

### Phase 1: lock the pairwise protocol

- write `compare.delta.v1.schema.json`
- write `compare.report.v1.schema.json`
- define alignment and equivalence types in `compare-core`

### Phase 2: deterministic engine

- implement key alignment
- implement row-order alignment
- implement atomic delta generation
- implement grouped report generation
- prove stable ordering

### Phase 3: profile and lock integration

- resolve profile-scoped columns
- record equivalence policy in output
- verify input membership against lockfiles

### Phase 4: CLI and human output

- implement `compare run`
- implement compact human output
- implement `--describe` and `--schema`

### Phase 5: later extensions

- consider non-CSV relation bindings
- consider timeline/lifecycle analysis only after pairwise protocol is stable

## Acceptance criteria for v0

`compare` is ready for first real use when all of this is true:

- the diff is defined in terms of relation alignment, not ad-hoc file text
- equivalence semantics are explicit and recorded in the report
- exhaustive changes are emitted deterministically
- `rvl` can be said to operate over the same conceptual delta surface
- profiles and locks integrate without ambiguity
- factory and tournament consumers can use compare reports as audit/debug
  evidence without inventing extra semantics

## The sentence to keep fixed

`compare` is the exhaustive delta protocol for the epistemic spine; `rvl` is
the materiality analysis over that same aligned change surface.
