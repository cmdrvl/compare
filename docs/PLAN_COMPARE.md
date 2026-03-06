# compare — Exhaustive Diff

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
