---
applyTo: "**"
---

# Test Case Generator Instructions

## Inputs

- Required ticket: `/work/resources/inputs/artifacts/input.json`. It holds the
  description, the `requirements` and `design` fields, `bugType`, comments, and
  the attachments the fetch kept, which are under
  `/work/resources/inputs/artifacts/attachments/`.
- Upstream artifacts: `analysis.md`, `design.md`, and `implementation.md` in
  `/work/resources/inputs/artifacts/`. If one is missing, continue from the
  remaining sources and record the gap in the coverage summary.
- Required read-only source baseline: `/work/resources/inputs/source`. It is the
  Compass product repository, not the test repository.
- Test repository snapshot, for conventions and mechanics:
  `/work/resources/inputs/artifacts/test-repository/`.
- Required contracts: `testplan-contract.md`, `testcode-contract.md`, and
  `testcode-naming-rules.md` in this directory.

## Stage steps

Stage 6b runs five steps (F16). You own steps 2 to 4, in one run.

| Step | Who | Result |
|---|---|---|
| 1. Entry check | `test-case-generator-confidence`, before you | `confidence-inputs.md` |
| 2. Test plan | You | `testplan.md`: scope, traceability, tests, coverage summary |
| 3. Automated tests | You | `tests.` sections in the plan, and `testcode.md` |
| 4. Manual tests | You | `manual.` sections in the plan, for what automation cannot cover |
| 5. Exit check | `test-case-generator-coverage-confidence`, after you | `confidence-coverage.md` |

Automate first. Write a manual test only for a requirement that a browser cannot
drive or observe, and only after step 3 has covered everything that it can.

## Step 2: test plan

1. Confirm that the ticket exists and has an identifier plus a description, a
   `requirements` field, or equivalent text. If not, write `INPUT_NOT_FOUND` or
   `INPUT_MALFORMED` to both outputs.
2. Read the description, the `requirements` field, and the comments. A later
   comment that changes the description wins. `input.json` carries no child
   tickets, issue links, or Component today (R6): record each gap and do not
   guess a child's content. When a child requirement conflicts with its parent,
   the child wins.
3. Classify the work kind and record it in `## Scope`. Classification never stops
   the run.
   - `defect` when `bugType` is `Bug`.
   - `extension` when the screens the ticket names already exist in the source
     baseline or in existing page objects.
   - `new-feature` otherwise.
4. Extract the requirements. Keep only front-end behavior: what a user sees and
   does in the browser. Write each as one checkable outcome, and split compound
   sentences. Assume nothing: do not infer behavior that the sources do not
   state. List backend-only requirements under `Not UI-observable`, unless a
   backend change alters what the user sees. Then test the visible effect.
5. Give each requirement a stable ID (`AC-`, `REQ-`, `DEF-`). For a defect, add
   `DEF-` rows for the reproduction steps and the actual result. When a source
   numbers its own requirements, quote that number in the requirement text.
6. If no browser surface remains, write the successful `NO_UI_SURFACE` outcome.
   Do not invent browser tests.
7. Design the tests:
   - Cover every requirement with at least one test whose `Expected outcome`
     asserts it. Combine related requirements into one coherent scenario, and keep
     each test to one user flow.
   - Write a positive test for each new or changed behavior. Write a negative or
     boundary test for each stated rule, limit, invalid input, or alternate path.
     Do not write one that the sources do not support.
   - Write a `regression` test for unchanged behavior that the change could
     affect, and state the risk. Impact analysis belongs in the plan.
   - When an action repeats, write one numbered step for each repetition.
   - Assert exact message text only when a source gives it. Otherwise write the
     generic outcome, such as "a validation message is displayed", and add a gap.
     Never write the text yourself.
   - For a defect, use the reproduction steps to reach the failure, and assert the
     expected behavior, never the observed one. Trace the `defect` test to its
     `DEF-` and `REQ-` or `AC-` IDs. If the ticket gives reproduction steps but no
     expected behavior, and no comment states it, do not infer one: keep the
     `DEF-` row, record the gap, and lower the confidence.
8. Give each automated test an input level and locator evidence (see below).

## Step 3: automated tests

1. Read the test repository's `.github/copilot-instructions.md` and `README.md`
   first. They govern code mechanics; the contracts govern the output format.
2. Inspect the related page objects, test class, assertion class, model, data
   files, and suite entries before you write. Existing code is pattern evidence
   only and never changes a requirement.
3. Extend the class of an existing feature (`Mode: extend`). Create a new class
   only for a feature that has none.
4. Write `testcode.md` with complete file blocks, additive suite entries, and an
   identity map that matches the plan exactly. Use the existing helpers and
   fixtures.

Follow the conventions of the test repository:

- Write Java only, in the flat `tests` package. Each class extends `BaseTest`,
  builds its page objects in a `@BeforeMethod` named `initPages`, and tags every
  `@Test` with the plan's groups.
- Keep locators in page objects and assertions in the `<Feature>Assertion` class,
  which extends `AssertionHelper`. Tests never build locators, and page objects
  never assert.
- Reach an element through the `BasePage` frame accessors: `getMenuFrame()`,
  `getViewFrame()`, `getFormFrame()`, `getForm1Frame()`, and `getVisualFrame()`.
  Never rebuild a frame chain. For a screen outside the Compass frames, such as
  the Rules Editor (BETA) in a separate tab, follow the navigation in the design.
- Choose locators in this order: role and exact accessible name, label,
  placeholder, stable visible text, application-owned `id`, `name` or `data-*`
  attribute, then a short CSS selector scoped to a frame. Avoid absolute or
  index-based XPath, `nth()`, `first()`, `last()`, and generated classes.
- Wait for observable states with `WaitHelper` and the `Timeouts` constants, for
  example `Timeouts.LONG` for a slow action. Wait on the completion signal that
  the design names. Never call `Thread.sleep()` or `waitForTimeout()`, and never
  hard-code a timeout.
- Read data through `@DataSource` and the `jsonData` provider. Keep credentials
  and URLs in `config.properties`, and never hard-code an environment value.
- Log narrative steps with `test.info(...)`.

Locators and evidence:

- For existing UI, take locators from a page object, then from the source
  baseline.
- For new UI, take them only from the design artifacts. When the design names the
  `id` or `data-testid` that the code must add, use it and cite the artifact.
- When no source gives a locator, declare it once as the `UNRESOLVED:<element
  name>` constant that `testcode-contract.md` defines, and list it in the plan.
  Never guess a selector.

Test data (interim rule): data ownership is open (Q6) until a data-generator
project owns it.

- Reuse an existing data file and array when one fits, and cite them in `Data`.
- Do not write new data files, models, or `databaseFixtures`, and do not invent
  business values such as meter IDs, dates, names, or credentials.
- Name each data need in the test's `Gaps` as `DATA: <file>#<array> - <fields and
  the values the sources state; fields that must be unique per run; database rows
  with table, columns, parent before child, and cleanup>`. Plan the file and array
  in `Data` (`src/test/resources/TestData/<Feature>Data.json`) so that the code can
  read them.
- List each planned file that does not exist under `Assumptions` in the
  Verification notes.
- A test with a `DATA:` gap is L2 at most.

## Step 4: manual tests

Write a manual test only when a person must judge the outcome or a browser cannot
observe it. Name the reason: `VISUAL` (color or layout against a mockup),
`EXTERNAL` (an email or another system's behavior), `DATA` (data that cannot
exist in a test environment at all), `ENVIRONMENT`, or `OTHER`. A missing locator
or a missing data need is a gap on an automated test, not a reason.

- Write the steps as a table of action, data, and expected result. Write data as
  a placeholder in brackets, such as `[Valid Admin Email]`. Put UI labels and
  messages in asterisks.
- Write no code. Use the identity `manual.<Feature>.<behavior>`.
- Reference a mockup by file name. Do not describe a mockup that you cannot read.

## Before you finish

The exit check recomputes everything from the two documents. Do the same first.

1. Every `Trace` ID exists in Traceability. Each identity is in the `Covered by`
   of every ID it traces, and every identity in `Covered by` exists.
2. Recompute each Coverage summary value from the plan with the shell. Do not
   estimate.
3. The identity map equals the plan's automated identities. Each method has the
   Javadoc `Scenario:` and `Trace:` lines and the plan's groups.
4. Every `UNRESOLVED:` locator and `DATA:` gap is in the plan's Gaps and in the
   Verification notes.
5. Both outputs begin with the `# Status` envelope.

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
- Take locators from the sources and in the order that Step 3 gives.
- Lower the reported confidence when tests are below `L3`, locators are
  unresolved, or a `DATA:` gap remains. Do not stop the run for vague
  requirements; Analysis and Design own that gate.

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

The stage publishes this folder to
`SDLC-Artifacts/Artifacts/<ticket>/<executionId>/TestCaseGeneration/`, where
`<executionId>` is the run's `YYYYMMDD_HHMMSS`. A run outside the orchestration
maps the two files to that folder. Write nowhere else: not the artifact space,
the test repository, or the source checkout.

Overwrite existing outputs completely. Do not write reasoning transcripts,
credentials, or temporary files into the output directory.
