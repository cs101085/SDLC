# Test plan contract

`testplan.md` is the reviewable test-design artifact produced by
`TestCaseGeneration`. It describes proposed tests; it is not the executable
source and it does not register tests in Xray.

## Required status envelope

The first three non-empty lines of every result, including a failure result,
are:

```markdown
# Status
STATUS_CODE:MESSAGE
Confidence: <integer>%
```

The status heading must be a level-one heading and must be the first heading
in the file. `Confidence` is an integer from `0` through `100`. The stage
gate reads the first matching status envelope and does not infer status from
later prose.

The generator may use these status codes:

| Code | Meaning | Required body |
|---|---|---|
| `SUCCESS` | One or more reviewable tests were produced. | At least one `## Test: ...` section |
| `NO_UI_SURFACE` | The ticket has no UI surface to drive. This is a successful, intentional empty plan. | A `## No UI surface` section and no test sections |
| `INPUT_NOT_FOUND` | A required input artifact was unavailable. | No test sections |
| `INPUT_MALFORMED` | A required input artifact could not be parsed. | No test sections |
| `GENERATION_FAILED` | Test design could not be completed. | No test sections |

`NO_UI_SURFACE` must not be represented as `SUCCESS` with an empty body.
Conversely, an empty plan with any other successful code is invalid.

## Document structure

After the status envelope, the document contains these sections in order:

1. `## Scope`
2. `## Traceability`
3. One `## Test: <test identity>` section for each proposed test, or
   `## No UI surface` when the status is `NO_UI_SURFACE`
4. `## Coverage summary`

The body is Markdown, but the headings and field labels below are required so
that a parser can extract records without interpreting prose.

### Scope

```markdown
## Scope

- Ticket: <ticket key>
- Ticket type: <`bugType` from input.json>
- Work kind: <new-feature | extension | defect>
- Classification evidence: <why this kind, such as the bug type or the existing screens found>
- Source precedence: <the F12 ranked list applied for this work kind>
- Design: <source artifact name>
- Implementation: <source artifact name>
- UI surface: <brief description or `none`>
```

`Work kind` selects the source precedence defined in F12 of the project document. The
classification never stops the run. When `Work kind` is `defect`, the expected result becomes a
`REQ-` row, and the reproduction steps and actual result become `DEF-` rows. If the ticket does not
supply them, record the gap in the coverage summary.

### Traceability

```markdown
## Traceability

| ID | Requirement or acceptance criterion | Covered by |
|---|---|---|
| REQ-1 | <verbatim or faithfully quoted requirement> | tests.LoginTest.testValidLogin |
| DEF-1 | <observed defect: steps and actual result> | tests.LoginTest.testLockedUserMessage |
```

Use `REQ-`, `AC-`, or the ticket's own IDs for intended behavior, and `DEF-` for observed defect
behavior. A `DEF-` row defines how to reach the failure. It never defines the expected outcome:
that comes from a `REQ-` or `AC-` row.

Every proposed test must reference at least one traceability `ID`. A
requirement or acceptance criterion that is intentionally not covered must be
listed with `Covered by` set to `UNCOVERED` and explained in the coverage
summary.

### Test record

Each test has exactly one section. The heading value is the fully qualified
TestNG method and must equal the `Identity` field:

````markdown
## Test: tests.LoginTest.testValidLogin

- Identity: `tests.LoginTest.testValidLogin`
- Title: Valid users can sign in
- Type: positive
- Trace: REQ-1, AC-1
- Scenario:

  ```gherkin
  Scenario: Valid users can sign in
    Given a registered user on the login page
    When the user submits valid credentials
    Then the dashboard is displayed
  ```

- Preconditions:
  - The user exists in the selected test environment.
- Steps:
  1. Open the login page.
  2. Enter valid credentials.
  3. Submit the form.
- Expected outcome:
  - The user is authenticated and redirected to the dashboard.
- Coverage rationale: Covers the successful authentication path and the
  redirect required by AC-1.
- Input level: L3
- Locator evidence:
  - `input[name='user']` — existing page object `pages.Login`
- Data: `src/test/resources/TestData/login.json`
- Group: smoke
````

The required fields are `Identity`, `Title`, `Type`, `Trace`, `Scenario`,
`Steps`, `Expected outcome`, `Coverage rationale`, `Input level`, and `Locator evidence`.
`Preconditions`, `Data`, and `Group` are optional. `Steps` must contain at least one numbered
step. `Trace` must contain at least one requirement or acceptance-criteria ID from the
Traceability table.

`Scenario` is one Gherkin `Scenario` block with at least one `Given`, `When`, and `Then` line. It
states behavior in domain terms. `Steps` states the concrete UI actions a person or the generated
code performs. Step 6 copies both into the Xray test description so a QA analyst can replay the
test by hand.

`Input level` is `L1`, `L2`, or `L3`, as defined in F15 of the project document. `Locator
evidence` lists each locator the test needs and where it was found: a page object, base-branch
source, or a design artifact. A locator with no evidence is listed as `UNRESOLVED`, and the test's
input level is below `L3`.

### No UI surface

For a ticket with nothing a browser can drive:

```markdown
## No UI surface

Reason: The ticket changes a scheduled service and introduces no page,
component, user interaction, or browser-visible outcome.
```

This outcome is reviewable and complete. It must not include invented browser
tests. The stage may still report a confidence value and a successful exit
gate for this outcome.

### Coverage summary

```markdown
## Coverage summary

- Requirements covered: REQ-1, AC-1
- Requirements not covered: none
- Test count: 1
- Input levels: L3: 1, L2: 0, L1: 0
- Unresolved locators: none
- Gaps and assumptions: none
```

## Parsing and validation rules

- Read the status envelope before parsing any other section.
- Require `Work kind` to be `new-feature`, `extension`, or `defect` for `SUCCESS`.
- Treat duplicate test identities as an error.
- Treat a test with missing traceability, steps, expected outcome, or rationale
  as malformed.
- Treat a test whose `Scenario` lacks a `Given`, `When`, or `Then` line as malformed.
- Treat a test whose `Trace` cites only `DEF-` IDs as malformed. An expected outcome
  needs an intended-behavior source.
- Require `Unresolved locators` to list every `UNRESOLVED` locator evidence entry.
- Require `Test count: 0` and no `## Test:` headings for `NO_UI_SURFACE`.
- Require `Test count` to equal the number of test sections for `SUCCESS`.
- Preserve the exact identity string; downstream Xray mapping is keyed by it.
- Do not treat prose outside the defined sections as structured fields.
