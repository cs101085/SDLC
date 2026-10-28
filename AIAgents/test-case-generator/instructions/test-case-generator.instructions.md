---
applyTo: "**"
---

# Test Case Generator Instructions

## Inputs

- Required ticket: `/work/resources/Jira/input.json`.
- Upstream artifacts: `analysis.md`, `design.md`, and `implementation.md` in
  `/work/resources/inputs/artifacts/`. If one is missing, continue from the
  remaining sources and record the gap in the coverage summary.
- Required read-only source baseline: `/work/resources/inputs/source`.
- Required contracts: `testplan-contract.md`, `testcode-contract.md`, and
  `testcode-naming-rules.md` in this directory.

## Workflow

1. Confirm the ticket exists and contains an identifier plus a description,
   acceptance criteria, or equivalent requirements. If not, write
   `INPUT_NOT_FOUND` or `INPUT_MALFORMED` to both outputs.
2. Read child tickets and attachments metadata when present. Treat a child
   requirement as the source of truth when it conflicts with its parent.
3. Separate browser-observable UI behavior from backend-only work. If no browser
   surface remains, write the successful `NO_UI_SURFACE` outcome; do not invent
   browser tests.
4. Classify the work kind and record it in `## Scope`. Classification never
   stops the run.
   - `defect` when `bugType` is `Bug`.
   - `extension` when the screens the ticket names already exist in the source
     baseline or in existing page objects.
   - `new-feature` otherwise.
5. Build a traceability table with stable requirement or acceptance-criteria IDs.
   For a `defect`, also add `DEF-` rows for the reproduction steps and actual
   result. Every proposed test must cite at least one `REQ-` or `AC-` ID.
6. Inspect the source baseline and its repository instructions. Search for
   matching pages, tests, assertion helpers, data fixtures, and suite entries.
   Existing code is pattern evidence only and must not change the requirements.
7. Write `testplan.md` with one stable fully qualified TestNG identity per test.
   Include positive behavior and supported negative or boundary behavior. Give
   each test a Gherkin scenario, numbered steps, an input level, and locator
   evidence.
8. Write `testcode.md` with complete file blocks, additive suite entries, and an
   identity map matching the plan exactly. Prefer existing helpers and fixtures.
   Never fabricate an API or locator; use the `UNRESOLVED:` convention instead.
9. Validate both files against their contracts, including counts, identities,
   allowed paths, duplicate detection, and status semantics.

## Source precedence

When sources conflict, the higher rank wins. Expected outcomes come only from
rank 1 or rank 2. Code supplies mechanics only: locators, navigation, data
shapes, and workflow order.

| Rank | `new-feature` | `extension` | `defect` |
|---|---|---|---|
| 1 | Ticket ACs and child tickets | Ticket ACs and child tickets | Expected behavior in the bug or its linked requirement |
| 2 | `analysis.md` | `analysis.md` | Observed defect: reproduction steps and actual result |
| 3 | `design.md`, `implementation.md` | `design.md`, `implementation.md` | `analysis.md` and design fix intent |
| 4 | Test repository, for mechanics | Existing page objects and tests, for mechanics, plus regression evidence for unchanged behavior | Source baseline and existing tests, for mechanics and workflow |
| 5 | Source baseline, for mechanics | Source baseline, for mechanics | — |

The observed defect defines how to reach the failure, not what is correct. A
`defect` test asserts the expected behavior and is expected to fail before the
fix. Do not claim it was run; the Verification runner checks this later.

## Scenarios and input levels

- Write one Gherkin `Scenario` per test in domain language: `Given` for the
  state, `When` for the user action, and `Then` for the observable outcome.
  Keep UI detail in `Steps`.
- Set each test's input level from the evidence actually found:
  - `L1`: acceptance criteria only.
  - `L2`: adds the screen, the navigation path, field labels, and expected
    messages.
  - `L3`: adds element IDs, data fields, and database rows.
- For new UI, take locators only from the design artifacts. For existing UI,
  take them from page objects or the source baseline. Prefer `#id`, then role
  and label locators, following the repository's own instructions.
- Lower the reported confidence when tests are below `L3` or locators are
  unresolved. Do not stop the run for vague requirements; Analysis and Design
  own that gate.

## F1 no-bias rule

The agent authors intent from the ticket. It uses the base-branch source to
understand what already exists and how tests are written, but it must never use
generated code, a feature branch, or post-implementation changes to discover or
rewrite requirements. The source checkout is read-only and must not receive
generated files.

## Output

Write only:

- `/work/resources/output/testplan.md`
- `/work/resources/output/testcode.md`

Overwrite existing outputs completely. Do not write reasoning transcripts,
credentials, or temporary files into the output directory.
