---
applyTo: "**"
---

# Test Case Generation exit assessor instructions

## What you judge

Test Case Generation has produced `testplan.md` and `testcode.md`. Rate whether
the plan covers what the sources require and whether the code carries out the
plan. The stage compares your rating with a threshold you are not told. A low
rating stops the stages after this one, so rate only what you can show.

## Inputs

All paths are in `/work/resources/inputs/artifacts/` unless stated. List one
bullet under `# Inputs` for each of: `testplan.md`, `testcode.md`, `input.json`,
`analysis.md`, `design.md`, `implementation.md`, `test-repository/`, and the
product source.

- `testplan.md` and `testcode.md`: the outputs under assessment.
- `input.json`, `analysis.md`, `design.md`, `implementation.md`: the sources.
- `test-repository/`: the Playwright repository snapshot, for existing page
  objects, tests, data files, and suite files.
- `/work/resources/inputs/source`: the product source at `baseBranch`.
- This pack's `instructions/`: the contracts that define both outputs.

If either output is missing, or its first envelope line is not `# Status`, write
the `FAILED:` envelope.

## Procedure

1. Read both envelopes. The plan status is `SUCCESS` or `NO_UI_SURFACE`; any
   other status means the stage should have stopped, so write `FAILED:`.
2. Read the ticket (description, `requirements`, comments), then the other
   sources. Build your own list of browser-observable requirements, in the
   plan's work kind order of precedence. Do this before you read Traceability, so
   the plan does not anchor you. Set aside backend-only behavior.
3. Read the plan. Match each of your requirements to the tests whose
   `Expected outcome` asserts it. A requirement that Traceability lists but no
   expected outcome asserts is not covered. A plan requirement you judge invalid
   is not counted.
4. Recompute the Coverage summary from the plan with the shell: requirements,
   covered, Coverage %, tests, coverage ratio, types, input levels, the BDD
   breakdown, and the unresolved locators. Use the formulas in
   `testplan-contract.md`.
5. Review test design (below).
6. Check the code against the plan (below).
7. Fill the rubric and write `confidence.md`. Keep the reasoning on STDOUT.

For `NO_UI_SURFACE`, confirm from the ticket and design that no browser-observable
behavior remains and that no `tests.` file exists. Score the justification.

## Test design review

- Each behavior has a positive test. Each stated rule, limit, invalid input, or
  alternate path has a negative or boundary test, when the sources give enough to
  write one.
- A change to an existing screen has a regression test for unchanged behavior at
  risk, with the risk stated.
- A defect has a `Type: defect` test that asserts the expected behavior, not the
  observed one, and traces a `DEF-` and a `REQ-` or `AC-` ID.
- A manual test has a valid reason: a person must judge it, or a browser cannot
  observe it. A missing locator or missing data is a gap on an automated test,
  not a reason for a manual one.
- No expected message, value, or outcome appears that the sources do not state.
- Every `Trace` and `Covered by` pair agrees, and each identity follows the
  naming rules.

## Plan and code consistency

- The identity map holds exactly the plan's automated identities, and each maps
  to a declared file and method.
- Each method has a Javadoc `Scenario:` and `Trace:` line and the plan's `Group`.
- Each generated class is in `testNg.xml`, and in `smoketest.xml` or
  `regressiontest.xml` for its groups, as a suite entry or already registered in
  `test-repository/`.
- Each file is in an allowed location. An `extend` file only adds lines. No new
  file already exists in `test-repository/`.
- Every `UNRESOLVED:` constant and every `DATA:` gap appears in the Verification
  notes. No concrete selector replaces a locator that the plan marks
  `UNRESOLVED`.
- No `test_key`, `test_id`, Xray annotation, credential, or hard-coded
  environment value is present.

## Rubric

Score each component from evidence and round to an integer. The `Total` is the
`Confidence`. The coverage ratio is informational: report it under
`# Findings`, and do not score it.

| Component | Max | Scoring |
|---|---|---|
| Coverage | 40 | `round(40 × covered ÷ total)`. `total` is the number of requirements in your list, plus any plan requirement you judge valid that your list lacks. `covered` is the number that a test (automated or manual) asserts. For a justified `NO_UI_SURFACE`, 40; otherwise 0. |
| Test design | 20 | Start at 20. Deduct 4 for each requirement with a stated rule or limit and no negative or boundary test; 4 when a changed existing screen has no regression test; 4 for each manual test whose reason does not hold; 4 for each test that states an outcome the sources do not; 4 for a defect with no valid `Type: defect` test. Floor 0. |
| Plan and code consistency | 20 | Start at 20. Deduct 5 for each kind of mismatch that you find in the list above (identity map, method and Javadoc and group, suite registration, unreported `UNRESOLVED:` or `DATA:`, an `extend` that is not additive or a new file that exists, forbidden content). Floor 0. For a plan of only manual tests, or `NO_UI_SURFACE`, 20 when there are no file blocks. |
| Input level | 15 | `round(15 × (L3 tests + 0.5 × L2 tests) ÷ automated tests)`. L1 tests count 0. 15 when there are no automated tests. |
| Metrics accuracy | 5 | Start at 5. Deduct 1 for each Coverage summary value that differs from your recomputed value. Floor 0. |

## Do not

- Do not fix, rewrite, or add tests, even as examples.
- Do not ask for a threshold or reason about what score lets the stage proceed.
- Do not score the quality of the product code or its design.
- Do not read a feature branch or generated implementation code.
