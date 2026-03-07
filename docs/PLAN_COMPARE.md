# compare — Exhaustive Diff

This document preserves the earlier feature-oriented framing of `compare`.

The repository's current implementation-direction plan is `docs/plan.md`. Where
this file and `docs/plan.md` disagree, prefer `docs/plan.md` for repo
structure, protocol boundaries, and the relationship between `compare`, `rvl`,
and factory/tournament usage.

## One-line promise
**List every change between two datasets — every row, every field — without materiality compression or summarization.**

---

## Problem

`rvl` explains what matters — the smallest set of numeric changes that account for the total change. But sometimes you need to see *everything*: every cell that changed, every row added, every row removed. For debugging, audit, and compliance, you need exhaustive truth before intelligent summary.

`compare` is the audit layer. `rvl` is the intelligence layer. They complement each other.

---

## Non-goals

`compare` is NOT:
- Materiality-aware (that's `rvl`)
- A validator (that's `verify`)
- A structural gate (that's `shape`)
- Numeric-only (it diffs all cell types as strings)

It does not rank, compress, or filter. It shows everything.

---

## CLI

```
compare <OLD> <NEW> [OPTIONS]

Arguments:
  <OLD>                  Old CSV file
  <NEW>                  New CSV file

Options:
  --key <COLUMN>         Align rows by key column (default: row-order)
  --profile <PATH>       Scope diff to columns in this profile's include_columns
  --profile-id <ID>      Profile ID (resolved from search path)
  --lock <LOCKFILE>      Verify inputs are members of these lockfiles (repeatable)
  --max-rows <N>         Refuse if input exceeds N rows (default: unlimited)
  --max-bytes <N>        Refuse if input file exceeds N bytes (default: unlimited)
  --delimiter <DELIM>    Force CSV delimiter
  --json                 JSON output
```

When `--profile` or `--profile-id` is provided, `compare` only diffs columns specified in the profile's `include_columns`. Accepts both draft and frozen profiles. Without a profile, all columns are diffed.

Providing both `--profile` and `--profile-id` in one invocation is a refusal (`E_AMBIGUOUS_PROFILE`).

When one or more `--lock` files are provided, `compare` hashes `<OLD>` and `<NEW>` and verifies they are present as members of at least one provided lockfile. On success, JSON output includes an `input_verification` block.

### Exit codes

`0` NO_CHANGES | `1` CHANGES | `2` refusal

---

## Output (JSON)

```json
{
  "version": "compare.v0",
  "outcome": "CHANGES",
  "profile_id": null,
  "profile_sha256": null,
  "input_verification": null,
  "files": { "old": "nov.csv", "new": "dec.csv" },
  "alignment": { "mode": "key", "key_column": "u8:loan_id" },
  "dialect": {
    "old": { "delimiter": ",", "quote": "\"", "escape": null },
    "new": { "delimiter": ",", "quote": "\"", "escape": null }
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
      "row_id": "u8:LN-00421",
      "fields": [
        { "column": "u8:balance", "old": "1234567.89", "new": "1234000.00" },
        { "column": "u8:status", "old": "u8:current", "new": "u8:delinquent" }
      ]
    },
    {
      "type": "added",
      "row_id": "u8:LN-99001",
      "fields": [ { "column": "u8:balance", "old": null, "new": "500000.00" } ]
    },
    {
      "type": "removed",
      "row_id": "u8:LN-00100",
      "fields": [ { "column": "u8:balance", "old": "750000.00", "new": null } ]
    }
  ],
  "refusal": null
}
```

`compare` is exhaustive — ALL changes listed, no materiality compression. Values are **strings** (raw cell values, not parsed as numeric). This is the key distinction from `rvl`: `compare` shows everything, `rvl` explains what matters.

---

## Refusal codes

Same alignment/parsing codes as `rvl`:

| Code | Trigger | Next step |
|------|---------|-----------|
| `E_IO` | Can't read file | Check paths |
| `E_ENCODING` | Encoding detection failure | Check file encoding |
| `E_CSV_PARSE` | Can't parse as CSV | Check format |
| `E_HEADERS` | Header issues | Check CSV headers |
| `E_NO_KEY` | Key column not found | Check column name |
| `E_KEY_EMPTY` | Key column has empty values | Fix data or use row-order |
| `E_KEY_DUP` | Key column has duplicates | Fix data or use row-order |
| `E_KEY_MISMATCH` | Key columns differ between files | Check files are comparable |
| `E_ROWCOUNT` | Row count issue | Check data |
| `E_NEED_KEY` | Key required but not provided | Add `--key` |
| `E_DIALECT` | CSV dialect issues | Specify `--delimiter` |
| `E_AMBIGUOUS_PROFILE` | Both `--profile` and `--profile-id` provided | Use one |
| `E_INPUT_NOT_LOCKED` | Input not in provided lockfile | Re-run with correct `--lock` |
| `E_INPUT_DRIFT` | Input hash doesn't match lock member | Use the locked file |
| `E_TOO_LARGE` | Input exceeds limits | Increase limit or split input |

---

## Usage examples

```bash
# Full diff between two CSVs
compare old.csv new.csv --key loan_id --json

# Row-order diff (no key)
compare old.csv new.csv --json

# Human output for eyeballing
compare old.csv new.csv --key loan_id

# compare -> rvl: see everything, then see what matters
compare nov.csv dec.csv --key loan_id --json > compare.report.json
rvl nov.csv dec.csv --key loan_id --json > rvl.report.json
# compare shows 387 cells changed; rvl shows 3 cells explain 96% of it

# compare -> pack: exhaustive diff as audit evidence
compare nov.csv dec.csv --key loan_id --json > compare.report.json
pack seal compare.report.json nov.lock.json dec.lock.json --output evidence/audit/
```

---

## Relationship to `rvl`

| | `compare` | `rvl` |
|-|-----------|-------|
| **Shows** | Every change | Top-K material changes |
| **Values** | Strings (raw) | Numeric (parsed) |
| **Compression** | None | Materiality-ranked |
| **Use case** | Audit, debug | Intelligence, explanation |
| **Outcome** | CHANGES / NO_CHANGES | REAL_CHANGE / NO_REAL_CHANGE / REFUSAL |

They compose naturally: run `compare` for the full picture, run `rvl` for the explanation.

---

## `compare timeline` — Longitudinal Row Lifecycles

### One-line promise

**Track every row across N snapshots — when it appeared, when it changed, when it disappeared, and the full value trajectory of every field.**

### Problem

Pairwise diffs answer "what changed between these two files." Audit and compliance ask harder questions:

- "Show me the full lifecycle of loan LN-00421 across all twelve monthly tapes."
- "Which loans changed status more than once this year?"
- "When did this balance first diverge from the prior month?"
- "Which rows have been stable for 6 months and suddenly changed?"

Running N pairwise comparisons produces N independent reports. Stitching them together manually is error-prone and loses temporal structure. `compare timeline` does the stitching: it takes N ordered snapshots, aligns by key, and produces per-row lifecycles with per-field change histories.

### CLI

```
compare timeline <FILE>... --key <COLUMN> [OPTIONS]

Arguments:
  <FILE>...              Two or more CSV files in chronological order

Options:
  --key <COLUMN>         Align rows by key column (required)
  --labels <LABEL>...    Human-readable labels for each snapshot (e.g., "Jan" "Feb" "Mar")
  --profile <PATH>       Scope to columns in this profile's include_columns
  --profile-id <ID>      Profile ID (resolved from search path)
  --lock <LOCKFILE>      Verify all inputs are members of these lockfiles (repeatable)
  --max-rows <N>         Refuse if any input exceeds N rows
  --max-bytes <N>        Refuse if any input file exceeds N bytes
  --delimiter <DELIM>    Force CSV delimiter
  --json                 JSON output
```

`--key` is required (row-order alignment across N files is meaningless — insertion/deletion makes positional identity unstable). Minimum 2 files; maximum is bounded only by memory.

`--labels` is optional. When provided, must have exactly as many labels as files. Labels appear in output instead of filenames, making timelines human-readable ("Jan", "Feb" vs "tape_20250101_v3_final.csv").

### Exit codes

`0` NO_CHANGES (all rows identical across all snapshots) | `1` CHANGES | `2` refusal

### Output (JSON)

```json
{
  "version": "compare_timeline.v0",
  "outcome": "CHANGES",
  "snapshots": [
    { "index": 0, "file": "jan.csv", "label": "Jan", "rows": 4200 },
    { "index": 1, "file": "feb.csv", "label": "Feb", "rows": 4185 },
    { "index": 2, "file": "mar.csv", "label": "Mar", "rows": 4220 }
  ],
  "alignment": { "key_column": "u8:loan_id" },
  "summary": {
    "total_unique_rows": 4280,
    "present_in_all": 4100,
    "appeared": 120,
    "disappeared": 60,
    "always_stable": 3812,
    "ever_modified": 408,
    "multi_modified": 47
  },
  "lifecycles": [
    {
      "row_id": "u8:LN-00421",
      "first_seen": 0,
      "last_seen": 2,
      "present": [0, 1, 2],
      "change_count": 2,
      "fields": {
        "u8:balance": {
          "trajectory": ["1234567.89", "1234000.00", "1234000.00"],
          "change_count": 1,
          "first_change": 1,
          "last_change": 1,
          "behavior": "changed_once"
        },
        "u8:status": {
          "trajectory": ["current", "delinquent", "current"],
          "change_count": 2,
          "first_change": 1,
          "last_change": 2,
          "behavior": "oscillating"
        },
        "u8:rate": {
          "trajectory": ["3.25", "3.25", "3.25"],
          "change_count": 0,
          "first_change": null,
          "last_change": null,
          "behavior": "stable"
        }
      }
    },
    {
      "row_id": "u8:LN-99001",
      "first_seen": 1,
      "last_seen": 2,
      "present": [1, 2],
      "change_count": 0,
      "fields": {
        "u8:balance": {
          "trajectory": [null, "500000.00", "500000.00"],
          "change_count": 0,
          "first_change": null,
          "last_change": null,
          "behavior": "stable"
        }
      }
    }
  ],
  "refusal": null
}
```

### Field behavior classification

Each field on each row is classified into one of five behaviors based on its trajectory across all snapshots where the row is present:

| Behavior | Meaning |
|----------|---------|
| `stable` | Value never changed across all snapshots |
| `changed_once` | Value changed exactly once and stayed at new value |
| `monotonic` | Value changed more than once, always in the same direction (only meaningful for numeric-parseable strings; otherwise falls through to `trending`) |
| `trending` | Value changed more than once but never reverted to a prior value |
| `oscillating` | Value reverted to a previously-held value at least once |

Classification is purely mechanical — string equality for change detection, no numeric parsing. `monotonic` is a bonus classification: if all trajectory values happen to parse as numbers and are strictly increasing or decreasing, the field is `monotonic` instead of `trending`. This is a free signal, not a requirement.

`null` in the trajectory means the row was not present in that snapshot (before `first_seen` or after `last_seen`). Nulls are excluded from behavior classification.

### Summary fields

| Field | Meaning |
|-------|---------|
| `total_unique_rows` | Count of distinct key values across all snapshots |
| `present_in_all` | Rows present in every snapshot |
| `appeared` | Rows not in first snapshot but in at least one later snapshot |
| `disappeared` | Rows in at least one snapshot but not in the last snapshot |
| `always_stable` | Rows present in all snapshots with zero field changes |
| `ever_modified` | Rows with at least one field change across any adjacent pair |
| `multi_modified` | Rows with field changes in more than one adjacent pair (the volatile ones) |

### Human output

Without `--json`, `compare timeline` prints a compact report:

```
Timeline: 3 snapshots (Jan → Feb → Mar), 4280 unique rows

  present in all   4100  ████████████████████████████████████  96%
  appeared          120  ███                                    3%
  disappeared        60  ██                                     1%

  always stable    3812  ██████████████████████████████████    89%
  changed once      361  ████████                               8%
  multi-modified     47  ██                                     1%

Volatile rows (changed in 2+ periods):
  LN-00421   status: current → delinquent → current (oscillating)
  LN-00833   balance: 1.2M → 1.1M → 950K (monotonic ↓)
  LN-01205   status: current → default → liquidated (trending)
  ... and 44 more (use --json for full detail)
```

### Refusal codes (timeline-specific)

| Code | Trigger | Next step |
|------|---------|-----------|
| `E_TOO_FEW_FILES` | Fewer than 2 files provided | Provide at least 2 snapshots |
| `E_LABEL_MISMATCH` | Number of `--labels` doesn't match number of files | Fix label count |
| `E_SCHEMA_DRIFT` | Key column missing from one or more snapshots | Check files are comparable |

Plus all standard `compare` refusal codes (`E_IO`, `E_CSV_PARSE`, `E_NO_KEY`, etc.).

### Usage examples

```bash
# Track all loans across a year of monthly tapes
compare timeline jan.csv feb.csv mar.csv apr.csv may.csv jun.csv \
  jul.csv aug.csv sep.csv oct.csv nov.csv dec.csv \
  --key loan_id \
  --labels Jan Feb Mar Apr May Jun Jul Aug Sep Oct Nov Dec

# Find volatile rows in quarterly reports
compare timeline q1.csv q2.csv q3.csv q4.csv \
  --key position_id --labels Q1 Q2 Q3 Q4 --json \
  | jq '.lifecycles[] | select(.change_count > 2)'

# Scope to balance columns only
compare timeline jan.csv feb.csv mar.csv \
  --key loan_id --profile balances.profile.json --json

# With provenance: verify every snapshot against lockfiles
compare timeline jan.csv feb.csv mar.csv \
  --key loan_id \
  --lock jan.lock.json --lock feb.lock.json --lock mar.lock.json

# Timeline as audit evidence
compare timeline jan.csv feb.csv mar.csv \
  --key loan_id --json > timeline.report.json
pack seal timeline.report.json jan.lock.json feb.lock.json mar.lock.json \
  --note "Q1 longitudinal audit" --output evidence/q1-timeline/
```

### Relationship to pairwise `compare`

| | `compare` (pairwise) | `compare timeline` |
|-|----------------------|-------------------|
| **Inputs** | 2 files | N files (2+) |
| **Identity** | Per-pair | Across full series |
| **Alignment** | Key or row-order | Key only (required) |
| **Output grain** | Per-change | Per-row lifecycle |
| **Temporal questions** | No | Yes (stability, oscillation, monotonicity) |
| **Use case** | "What changed?" | "How has this row lived?" |

They compose naturally: use pairwise `compare` to investigate a specific transition flagged by `timeline`, or use `timeline` to contextualize a pairwise diff within the broader history.

### Why this matters

1. **Audit is temporal.** Regulators don't ask "what changed last month?" — they ask "show me the history." A tool that only diffs pairs forces humans to stitch the timeline manually.
2. **Volatility detection is free.** The `multi_modified` and `oscillating` classifications surface data quality problems (flip-flopping statuses, see-sawing balances) that are invisible in any single pairwise diff.
3. **Purely deterministic.** Same files in same order = same output. No heuristics, no ML — just exhaustive temporal tracking.
4. **Composes with every other tool.** Timeline report → `assess` (policy decision on longitudinal quality). Timeline report → `pack` (sealed audit evidence). Timeline volatility → `verify` (are oscillating rows violating a contract?).
5. **Agent-native.** An agent running a quarterly reconciliation can run `compare timeline` across the full year and surface rows that need attention — without being told which rows to look at.

---

## Implementation notes

The `csv-diff` crate (Rust, Martin Hafskjold Thoresen) implements the HashMap-join diff pattern that `compare` needs — key-column alignment, field-level change classification (added/removed/modified). Evaluate as a direct dependency or as a reference for building on `rvl`'s existing alignment code.

### Candidate crates

| Need | Crate | Notes |
|------|-------|-------|
| CSV parsing | `csv` (BurntSushi) | Same as `rvl` |
| Tabular diff reference | `csv-diff` | Evaluate as dependency or reference |
| Content hashing | `sha2` | Input verification |

---

## Determinism

Same inputs, same output, every time. No randomness, no side effects. Change ordering in the `changes` array follows key order (key-aligned) or row order (row-order mode).
