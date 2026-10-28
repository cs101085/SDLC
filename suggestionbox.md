# Project suggestion box

This is a high-level project reference for proposed changes and questions to
raise in meetings with other departments. It is not owned by any one
repository.

## Design stage: identify new UI test elements

The Design implementation plan provides every locator, navigation path, and
element ID that a test of new UI needs. The Test Case Generation agent reads
the design and implementation plans and the source at `baseBranch`, where new
elements do not yet exist, so it cannot discover their identifiers from the
feature branch without violating the no-bias rule.

**Status:** Agreed. Design does not rate testability; a Confidence Agent at
the start of Test Case Generation checks that the requirements and Design
artifacts are accurate and complete.

**Detail provided (F15, input level L3).** For each new or changed screen, the
implementation plan gives the items below. The list was checked against the
Playwright documentation (locators, best practices, auto-waiting, frames,
dialogs, downloads, authentication) and the Compass test repository
conventions (frame accessors, `@DataSource` JSON, `DatabaseFixtureLoader`,
`WaitHelper`).

*Where and who*

1. **Route and navigation path.** The menu path or URL to the screen, the
   starting state, and every frame on the way. Name the frame by the
   `BasePage` accessor (`getViewFrame()`, `getFormFrame()`,
   `getForm1Frame()`, and so on) so the test never builds its own frame
   chain. Also say whether the action opens a dialog, a new tab, or a popup.
2. **Role and account.** The role required to sign in and the permissions that
   decide what the user sees. Name the `config.properties` key for the
   account; never put credentials in the plan. For permission rules, give a
   role that should be blocked and what it sees instead.

*What the test touches*

3. **A locator for each new or changed control.** Prefer a stable, unique `id`
   (about two-thirds of existing locators). Where there is no `id`, give the
   accessible role and name, the label, or the placeholder, which is the order
   Playwright recommends. Give the control type (button, text box, select,
   checkbox, grid, tab), the owning frame, and, for a repeated item such as a
   grid row, the key that identifies one row and the column headers. Do not
   rely on CSS classes or positions. Say which existing locators were renamed
   or removed, because existing page objects may break.
4. **Fields and data rules.** For each field: name and label, type, whether it
   is required, default value, whether it is read-only or enabled, the valid
   values, and the invalid and boundary values (length, range, format, date
   and number format). For a dropdown, the option list. Say which values must
   be unique per run so the test can use a unique-value placeholder.
5. **Exact text of user-visible messages.** The wording, where it appears
   (toast, inline error, dialog, banner), what triggers it, and any variable
   parts such as a name or count. The test compares the text exactly.
6. **Database rows for setup and checking.** Table, columns, values, parent
   rows before child rows, and the cleanup rule. Note differences between
   PostgreSQL and Oracle. Also name the rows and columns the action should
   change, so the test can confirm that the result was saved.

*How the screen behaves*

7. **State and rules.** The initial state when the screen loads, and when each
   control is shown, hidden, enabled, or disabled. A test can only assert a
   rule it is told about.
8. **Completion signal.** What the user sees when an asynchronous action
   finishes: a toast, a spinner that disappears, a row that appears, or a
   changed value. Playwright waits for elements, not for the application
   itself, so each slow action needs an observable end point. Mark long-running
   actions so the test uses the right timeout.
9. **Expected result after the action.** The observable outcome, such as the
   screen shown next, the changed grid row, or the value that remains after a
   reload. Express it in what a user can see, not in internal state.
10. **Browser side effects.** For a browser dialog (alert, confirm, prompt):
    its text and whether to accept or dismiss it. For a download: the file
    name pattern and format. For an upload: the accepted file types, the size
    limit, and the sample file to add under `src/test/resources/TestData/`.

*Running reliably*

11. **Test independence.** Anything the test needs that it cannot create
    itself: feature flags, configuration, licences, time-dependent values (the
    current date or a business period), and data shared with other tests. Each
    test must be able to run alone and in any order.
12. **External services.** Third-party or backend calls the screen makes that
    the test should not depend on, with the response a stub should return.
    This is needed only when the feature calls such a service.
13. **Traceability.** The requirement or acceptance-criteria ID that each
    control, message, and rule serves, so every test can cite its source.

If a locator or any other item is still missing, the test is generated with
the gap flagged (`UNRESOLVED:`) and its confidence lowered. Items 1 to 6 are
the minimum for input level L3; items 7 to 13 are required when they apply to
the feature.

**Sources:**
[Playwright best practices](https://playwright.dev/docs/best-practices),
[Locators](https://playwright.dev/java/docs/locators),
[Auto-waiting](https://playwright.dev/java/docs/actionability),
[Frames](https://playwright.dev/java/docs/frames),
[Dialogs](https://playwright.dev/java/docs/dialogs),
[Downloads](https://playwright.dev/java/docs/downloads),
[Authentication](https://playwright.dev/java/docs/auth),
and the Test Repository README (Compass iframe model, test data format,
conventions).

## Jira: require defect fields on bugs

For `defect` tickets, Test Case Generation ranks the observed defect second,
after the expected behavior (F12). The bug issue type requires reproduction
steps, an expected result, and an actual result, either as separate fields or
as a description template.

**Status:** Assumption. Test Case Generation relies on these fields; confirm
with the Jira administrator.

## Verification runner: prove defect tests fail before the fix

A regression test for a bug is only meaningful if it reproduces the bug.
Please make the future runner run each `defect` test twice: first against the
base branch, where it must fail, and then against the fix, where it must pass.
Record both results.

**Status:** Recorded as a requirement for the runner project (F4); not
enforceable until a runner exists.

## Requirements stage: no artifact for downstream stages

The Requirements stage is a stub that provides no artifact, and a bug skips it
and enters at Analysis. Downstream stages therefore treat the Jira ticket and
`analysis.md` as the requirements. If Requirements later produces an artifact,
Test Case Generation can rank it first for `new-feature` and `extension` work.

**Status:** For information; no change requested yet.

## Test Case Generation stage: improvements over the manual process

The new `test-case-generator` pack replaces the manual analyst handoff with
two validated artifacts: a traceable `testplan.md` and a complete
repository-relative `testcode.md`. It preserves the useful two-pass intent
from the former requirement and test-step prompts, including child-over-parent
conflict resolution, UI-only scope, positive/negative coverage, and a human
review point, while adding:

- Stable fully qualified TestNG identities for later Xray and JUnit binding.
- Explicit `NO_UI_SURFACE` handling instead of an ambiguous empty plan.
- Complete file blocks and suite entries that can be reviewed before they are
  copied into the test repository.
- A read-only base-branch source map that prevents generated implementation
  code from biasing requirements.
- Contract validation for traceability, duplicate identities, allowed paths,
  and status envelopes.

**Status:** Implemented in the `test-case-generator` pack; the stage wiring and
live runner remain subsequent project steps.
