# Intent

## Goal
Give a support coordinator a safe way to preview support tickets from CSV text before
any later import is approved. Add one pure function that parses the CSV, checks each
row and returns either the complete list of tickets in input order or a single error
code. The preview never changes data. It only shows what a future import would contain.

## Scope
A future implementation may create only two new files: `src/import-tickets.mjs`, which
exports the preview function, and `tests/import-tickets.test.mjs`, which holds its tests.
Do not change `src/tickets.mjs`, anything under `.github/`, `CLAUDE.md`, `package.json`
or any other existing file. A UI, database storage, network calls, file writes, real
imports and new package dependencies are out of scope. In this round only `intent.md`
changes. Do not write implementation code now.

## Constraints
- Reuse `src/tickets.mjs` as the stable ticket representation without modifying it.
- Use plain JavaScript (ES modules, Node 22 or later) with no new dependencies.
- The function must be side-effect free: no file system, network, global state or logging.
- Require a `title` header and allow an optional `priority` header. Any other column, a
  missing `title`, duplicate column names, a malformed row (wrong field count or broken
  quoting) or empty input with no header returns `{"error":"unknown-column"}`.
- Respect CSV quoting, so a quoted field may contain a comma.
- Trim surrounding whitespace from every value.
- An empty title after trimming returns `{"error":"empty-title"}`.
- Valid priorities are `low`, `normal` and `urgent`. A missing or blank priority becomes
  `normal`. Any other value returns `{"error":"invalid-priority"}`.
- Accept at most 100 data rows. Count the rows before validating any row values. More
  than 100 rows returns `{"error":"row-limit"}`.
- Success is `{"tickets":[{"title":"...","priority":"..."}]}` in input order. Never
  return partial success: the first error replaces the whole result.

## Plan
1. Stop and ask the user for explicit approval before writing any code. If a request
   would touch files or behaviour outside this scope, ask first instead of expanding it.
2. In `tests/import-tickets.test.mjs`, write tests that mirror every acceptance case below.
3. In `src/import-tickets.mjs`, write a small CSV tokenizer that handles quoted fields,
   escaped quotes and a trailing newline.
4. Validate the header (unknown, missing or duplicate columns), then check the
   number of data rows against the 100-row limit.
5. Validate each row in order (field count, trimmed title, priority default and allowed
   values) and build tickets through `ticket()` from `src/tickets.mjs`.
6. Return the success object or the first error object, then run the verification commands.

## Acceptance
The preview is complete when every case below produces exactly the expected result,
including error codes, ticket order and the default priority. The cases cover a valid
explicit priority, the default priority, a quoted title with a comma, an empty title,
an unknown column, an invalid priority and too many rows. The last case uses `repeat`
to mean 101 copies of the single data row after one header line.

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title\nReset password\n",
      "expected": {"tickets":[{"title":"Reset password","priority":"normal"}]}
    },
    {
      "csv": "title,priority\nVPN down,\n",
      "expected": {"tickets":[{"title":"VPN down","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n\"Question about billing, October\",low\n",
      "expected": {"tickets":[{"title":"Question about billing, October","priority":"low"}]}
    },
    {
      "csv": "title,priority\n,low\n",
      "expected": {"error":"empty-title"}
    },
    {
      "csv": "title,owner\nPrinter issue,Ana\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,critical\n",
      "expected": {"error":"invalid-priority"}
    },
    {
      "csv": "title\nPrinter issue\n",
      "repeat": 101,
      "expected": {"error":"row-limit"}
    }
  ]
}
```

## Verification
After implementation, run `npm test` and confirm that the existing ticket tests and the
new import tests all pass. Then run `git diff --check` and confirm that it reports no
whitespace problems. Check that `git status` lists only the two allowed new files. Report
the results to the user and ask for approval before committing. Do not push, open a PR
or merge on your own. If a check fails or a requirement is unclear, stop and ask rather
than widening the scope.
