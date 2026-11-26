# Test plan contract

`testplan.md` is the reviewable test-design artifact that `TestCaseGeneration`
produces. It describes proposed automated and manual tests. It is not
executable source, and it does not register tests in Xray.

## Status envelope

The first three non-empty lines of every output, including a failure, are:

```markdown
# Status
STATUS_CODE:MESSAGE
Confidence: <integer>%
```

The heading is level one and is the first heading in the file. `MESSAGE` is one
short sentence without stack traces. `Confidence` is an integer from `0`
through `100`: the generator's self-rating, lowered for input levels below L3
and for unresolved locators. It is advisory. The entry and exit assessments
gate the stage. The stage reads the first envelope only and never infers status
from later prose.

| Code | Meaning | Required body |
|---|---|---|
| `SUCCESS` | One or more tests were produced. | At least one `## Test:` section |
| `NO_UI_SURFACE` | The ticket has no UI surface to drive. This is a successful, intentional empty plan. | A `## No UI surface` section and no test sections |
| `INPUT_NOT_FOUND` | A required input artifact was unavailable. | Envelope only |
| `INPUT_MALFORMED` | A required input artifact could not be parsed. | Envelope only |
| `GENERATION_FAILED` | Test design could not be completed. | Envelope only |

`NO_UI_SURFACE` is not `SUCCESS` with an empty body, and an empty plan with any
other successful code is invalid.

## Document structure

After the envelope, the sections occur in this order:

1. `## Scope`
2. `## Traceability`
3. One `## Test: <identity>` section per test, or `## No UI surface`
4. `## Coverage summary`

Headings and field labels are required so that a parser can extract records
without interpreting prose. The examples in this contract are excerpts, and
their values do not reconcile across sections.

### Scope

```markdown
## Scope

- Ticket: <ticket key>
- Ticket type: <Jira issue type; `bugType` in input.json>
- Work kind: <new-feature | extension | defect>
- Classification evidence: <why this kind, such as the issue type or the screens found>
- Source precedence: <the F12 ranked list applied for this work kind>
- Component: <Jira Component names, comma separated, or none>
- Folder: </first Component, or / when there is none>
- Design: <artifact name, or missing>
- Implementation: <artifact name, or missing>
- UI surface: <brief description, or none>
- Not UI-observable: <requirements left out because a browser cannot observe them, or none>
```

`Work kind` selects the source precedence in F12. Classification never stops
the run. For a `defect`, the expected result becomes a `REQ-` row, and the
reproduction steps and actual result become `DEF-` rows. If the ticket omits
them, record the gap under Gaps.

`input.json` carries no Component today (R6). Until it does, write `none`,
`Folder: /`, and record the gap. `Folder` is the Xray folder for manual tests.
It uses the first Component only.

`Not UI-observable` lets the exit assessor challenge the scope decision. It
does not count toward the metrics.

### Traceability

```markdown
## Traceability

| ID | Requirement or acceptance criterion | Source | Jira issue | Covered by |
|---|---|---|---|---|
| AC-1 | <verbatim or faithfully quoted requirement> | ticket | PROJ-12 | tests.LoginTest.testValidLogin |
| REQ-2 | <requirement that a later comment changed> | comment | PROJ-12 | manual.Login.verifyBrandColors |
| DEF-1 | <observed defect: steps and actual result> | ticket | PROJ-12 | tests.LoginTest.testLockedUserMessage |
```

- **ID.** `AC-n` for acceptance criteria stated as such, `REQ-n` for other
  intended behavior, and `DEF-n` for observed defect behavior. A `DEF-` row
  defines how to reach the failure. It never defines the expected outcome.
- **Source.** One of `ticket`, `comment`, `child`, `analysis`, `design`, or
  `implementation`. The source makes the F12 ranking auditable. A later comment
  that changes the description wins; cite the author and date in the
  requirement text.
- **Jira issue.** The issue key that Step 6 links the test to: the ticket key,
  or the child ticket key for a `child` source.
- **Covered by.** Comma-separated test identities, or `UNCOVERED` with the
  reason in the coverage summary.

Every test references at least one ID.

### Test record

Each test has one section. The heading value equals the `Identity` field.
The identity prefix decides the mode: `tests.` is automated and `manual.` is
manual. [Naming rules](./testcode-naming-rules.md) define both forms.

````markdown
## Test: tests.LoginTest.testValidLogin

- Identity: `tests.LoginTest.testValidLogin`
- Mode: automated
- Feature: Login
- Title: Valid users can sign in
- Type: positive
- Trace: AC-1
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
- Coverage rationale: Covers the successful sign-in path required by AC-1.
- Input level: L3
- Locator evidence:
  - `#user` in `getViewFrame()` — page object `pages.Login`
- Data: `src/test/resources/TestData/LoginData.json` (`validLogin`)
- Group: smoke, regression
- Gaps: none
````

A manual test replaces `Input level`, `Locator evidence`, `Data`, and `Group`
with `Manual reason`, and writes `Steps` as a table that maps to Xray manual
steps:

````markdown
## Test: manual.Login.verifyBrandColors

- Identity: `manual.Login.verifyBrandColors`
- Mode: manual
- Feature: Login
- Title: The login button uses the brand color
- Type: positive
- Trace: REQ-2
- Scenario:

  ```gherkin
  Scenario: The login button uses the brand color
    Given the login page is displayed
    When a reviewer compares the button with the approved mockup
    Then the button color matches the mockup
  ```

- Steps:

  | Action | Data | Expected result |
  |---|---|---|
  | Open the login page. | (None) | The login page is displayed. |
  | Compare the login button with the mockup. | `login-mockup.png` | The colors match. |

- Expected outcome:
  - The button color matches the approved mockup.
- Coverage rationale: Color accuracy needs a human comparison.
- Manual reason: VISUAL: A DOM assertion cannot judge the color against a mockup.
- Gaps: none
````

| Field | Automated | Manual | Rule |
|---|---|---|---|
| `Identity` | Required | Required | Fully qualified identity |
| `Mode` | Required | Required | `automated` or `manual` |
| `Feature` | Required | Required | Cucumber `Feature:` name; groups the BDD breakdown |
| `Title` | Required | Required | One line; becomes the Xray summary |
| `Type` | Required | Required | `positive`, `negative`, `boundary`, `regression`, or `defect` |
| `Trace` | Required | Required | IDs from Traceability |
| `Scenario` | Required | Required | One `gherkin` block with `Given`, `When`, and `Then` lines |
| `Preconditions` | Optional | Optional | Bulleted list |
| `Steps` | Numbered list | Table | At least one step or row |
| `Expected outcome` | Required | Required | Observable result |
| `Coverage rationale` | Required | Required | Why the test exists |
| `Input level` | Required | Omit | `L1`, `L2`, or `L3` (F15) |
| `Locator evidence` | Required | Omit | Each locator and where it was found |
| `Data` | Optional | Omit | File, and array name for `@DataSource` |
| `Group` | Required | Omit | `smoke`, `regression`, or both |
| `Manual reason` | Omit | Required | `<CATEGORY>: <sentence>` |
| `Gaps` | Required | Required | `none`, or each flagged gap on its own bullet |

- **Scenario and steps.** `Scenario` states behavior in domain terms. `Steps`
  states the concrete UI actions. Step 6 copies both into the Xray test
  description so an analyst can replay the test by hand.
- **Type.** `regression` checks unchanged behavior that the change could
  affect. `defect` marks the test that reproduces the reported defect: it
  asserts the expected behavior and is expected to fail on the base branch. The
  Verification runner uses it for the red-then-green check. `defect` is valid
  only when `Work kind` is `defect`, and its `Trace` cites at least one `DEF-`
  ID and one `REQ-` or `AC-` ID.
- **Locator evidence.** Write each locator as `<locator> — <where found>`: a
  page object, base-branch source, or design artifact. A locator without
  evidence is `UNRESOLVED:<element name>`, and the test's input level is below
  L3.
- **Manual reason.** The category is `VISUAL`, `EXTERNAL`, `DATA`,
  `ENVIRONMENT`, or `OTHER`. Manual is the last resort (F2): the sentence says
  why automation cannot cover the requirement.
- **Gaps.** Flag what the sources did not state, such as an exact message or a
  missing Component. Never invent the missing value.

### No UI surface

```markdown
## No UI surface

Reason: The ticket changes a scheduled service and introduces no page,
component, user interaction, or browser-visible outcome.
```

This outcome is complete and reviewable. It contains no invented browser tests.
The Traceability table has no rows, and the stage may still report a confidence
value and pass its gate.

### Coverage summary

```markdown
## Coverage summary

### Metrics

- Requirements: 4
- Requirements covered: 3
- Requirements not covered: AC-4 (the ticket gives no exact message)
- Coverage %: 75
- Tests: 5 (automated 4, manual 1)
- Coverage ratio: 1.67
- Test types: positive 2, negative 1, boundary 1, regression 1, defect 0
- Input levels: L3: 2, L2: 1, L1: 1

### BDD breakdown

| Feature | Scenarios | Automated | Manual |
|---|---|---|---|
| Login | 5 | 4 | 1 |

### Gaps

- Unresolved locators: none
- Gaps and assumptions: none
```

The generator computes every value from the document, so a parser can recompute
and compare them:

- `Requirements` counts the `AC-` and `REQ-` rows. `DEF-` rows are excluded.
- `Requirements covered` counts those rows whose `Covered by` is not `UNCOVERED`.
- `Coverage %` is `round(100 × covered ÷ requirements)`. It is `n/a` when there
  are no requirements.
- `Coverage ratio` is `tests ÷ covered requirements`, rounded to two decimals. It
  is `n/a` when nothing is covered.
- `Input levels` counts automated tests only.
- `BDD breakdown` has one row per distinct `Feature`.
- `Unresolved locators` lists each as `<identity>: <element name>`.

For `NO_UI_SURFACE`, write `Tests: 0 (automated 0, manual 0)`, mark the other
metrics `n/a`, and keep Gaps.

## Parsing and validation rules

- Read the envelope before any other section.
- Require a valid `Work kind` for `SUCCESS`.
- Require the heading to equal `Identity`, the `tests.` prefix to match
  `automated`, and the `manual.` prefix to match `manual`.
- Treat a duplicate identity as an error.
- Require the fields that the table lists for the mode. Treat a missing field,
  an empty `Steps`, or a `Scenario` without `Given`, `When`, and `Then` as
  malformed.
- Require every `Trace` ID to exist in Traceability, and require at least one
  `REQ-` or `AC-` ID. A `DEF-` ID alone cannot define an expected outcome.
- Require Traceability and `Trace` to agree: each test appears in `Covered by`
  for every ID it traces, and every identity in `Covered by` exists.
- Require `Jira issue` to match `[A-Z][A-Z0-9]+-[0-9]+`.
- Require a test with an `UNRESOLVED:` locator to have an input level below L3,
  and require `Unresolved locators` to list every such locator.
- Require every Coverage summary value to equal its recomputed value.
- Require `Tests: 0` and no `## Test:` heading for `NO_UI_SURFACE`.
- Preserve identities exactly. Xray result binding keys on them.
- Do not treat prose outside the defined sections as structured fields.
