# Project suggestion box

This is a high-level project reference for proposed changes and questions to
raise in meetings with other departments. It is not owned by any one
repository.

## Design stage: identify new UI test elements

Test Case Generation reads the design and implementation plans and the source
at `baseBranch`, where new elements do not yet exist. It cannot take their
identifiers from the feature branch without violating the no-bias rule, so the
implementation plan must supply every locator, navigation path, and element ID
that a test of new UI needs.

Design owns the mechanics: where the screen is, how to find each control, and
how to know an action has finished. The requirements own expected outcomes
(message text, rules, results) and rank above Design (F12). The plan cites the
requirement for each outcome and flags any difference; it does not restate or
add outcomes.

**Status:** Agreed. Design does not rate testability; a Confidence Agent at the
start of Test Case Generation checks that the requirements and Design artifacts
are accurate and complete.

**Detail provided (F15, input level L3).** For each new or changed screen, the
implementation plan gives the items below. They were checked against the
Playwright documentation and the Compass test repository conventions (frame
accessors, `@DataSource` JSON, `DatabaseFixtureLoader`, `WaitHelper`).

*Where and who*

1. **Route and navigation.** The menu path or URL, the starting state, and each
   frame on the way, named by its `BasePage` accessor (`getViewFrame()`,
   `getFormFrame()`, `getForm1Frame()`) so the test never builds its own frame
   chain. For a screen outside the Compass frames, say how the test finds it
   (the Rules Editor (BETA) opens in a separate tab, matched by the
   `rule_editor` URL fragment). Say whether the action opens a dialog, tab, or
   popup. For a canvas or graph screen, give the starting graph (for example, a
   rule with two Decision Makers of two Action Takers each) and how to create
   it: canned file, fixture, or drag and drop.
2. **Role and account.** The role needed to sign in, the permissions that change
   what the user sees, and the `config.properties` key for the account (never
   credentials). For a permission rule, a role that should be blocked and what
   it sees instead.

*What the test touches*

3. **A locator for each new or changed control.** In this order: accessible
   role and name, label, or placeholder (the order Playwright and the Test
   Repository README recommend), then a stable, unique `id` or `data-testid`.
   Where the screen has no such hook (the Rules Editor (BETA) has none today),
   the plan names the `id` or `data-testid` the code must add, and Code
   Generation implements it as written (see open points). A stable
   library-owned class such as `.react-flow__node-actionTaker` is allowed when
   nothing user-facing exists; application styling classes and positions are
   not. Also give:
   - the control type (button, text box, select, checkbox, grid, tab, menu
     item) and its frame or tab;
   - for a repeated item (grid row, canvas node), the key that identifies one
     item, such as its label, and the column headers;
   - any existing locator that was renamed or removed, since page objects may
     break.
4. **Fields and data rules** (if the feature has fields). Per field: name and
   label, type, required, default, read-only or enabled, valid values, and
   invalid and boundary values (length, range, format, date and number format).
   For a dropdown, the options. Say which values must be unique per run, so the
   test can use a unique-value placeholder.
5. **Exact message text.** The wording, where it appears (toast, inline error,
   dialog, banner), its trigger, and any variable parts such as a name or
   count. The test compares the text exactly. The requirements own the wording;
   the plan cites them and lists any message they leave open or the design
   changes.
6. **Database rows** (if the feature reads or writes the database). Table,
   columns, values, parent rows before child rows, the cleanup rule, and any
   PostgreSQL or Oracle difference. Also name the rows and columns the action
   should change, so the test can confirm the save.

*How the screen behaves*

7. **State and rules.** The initial state on load, and when each control is
   shown, hidden, enabled, or disabled. A test can only assert a rule it is
   told. Cite the requirements and flag any difference.
8. **Completion signal.** What the user sees when an asynchronous action
   finishes: a toast, a spinner that disappears, a new row, or a changed value.
   Playwright waits for elements, not for the application, so each slow action
   needs an observable end point. Mark long-running actions for a longer
   timeout.
9. **Expected result.** What a user can see afterward, such as the next screen,
   the changed grid row, or the value after a reload; not internal state. Cite
   the requirement that states it and add no outcomes of your own.
10. **Browser side effects.** Dialog (alert, confirm, prompt): its text and
    whether to accept or dismiss. Download: file name pattern and format.
    Upload: accepted file types, size limit, and the sample file to add under
    `src/test/resources/TestData/`.

*Running reliably*

11. **Test independence.** Anything the test cannot create itself: feature
    flags, configuration, licences, time-dependent values (current date,
    business period), and data shared with other tests. Each test must run
    alone and in any order.
12. **External services** (only if the screen calls one). Calls the test should
    not depend on, and the response a stub should return.
13. **Traceability.** The requirement or acceptance-criteria ID behind each
    control, message, and rule, using IDs that exist in the requirements
    artifact (if there are none, cite section, item, and row, and say so).
    State the requirements version (execution ID or date) the plan was built
    from, so a later revision is detected.

If an item is missing, the test is generated with the gap flagged
(`UNRESOLVED:`) and lower confidence. Items 1 to 6 are the minimum for L3 where
they apply (4 and 6 only for fields or database rows); items 7 to 13 are
required when they apply.

**Sources:**
[Playwright best practices](https://playwright.dev/docs/best-practices),
[Locators](https://playwright.dev/java/docs/locators),
[Auto-waiting](https://playwright.dev/java/docs/actionability),
[Frames](https://playwright.dev/java/docs/frames),
[Dialogs](https://playwright.dev/java/docs/dialogs),
[Downloads](https://playwright.dev/java/docs/downloads),
[Authentication](https://playwright.dev/java/docs/auth),
the Test Repository README (Compass iframe model, test data format,
conventions), and the Rules Editor page object (`RuleEditor.java`).

## Jira: require defect fields on bugs

For `defect` tickets, Test Case Generation ranks the observed defect second,
after the expected behavior (F12). The bug issue type requires reproduction
steps, an expected result, and an actual result, either as separate fields or
as a description template.

**Status:** Assumption. Test Case Generation relies on these fields; confirm
with the Jira administrator.

**Evidence against the assumption.** METERSENSE-11680, a bug, has numbered
reproduction steps inside a prose description and no expected or actual result.
A later comment changes the expected behavior ("Only display the comments line
if the report is showing data"). Test Case Generation reads comments for that
reason, but a template would make the expected result explicit and raise
confidence.

## Verification runner: prove defect tests fail before the fix

A regression test for a bug is only meaningful if it reproduces the bug.
Please make the future runner run each `defect` test twice: first against the
base branch, where it must fail, and then against the fix, where it must pass.
Record both results.

**Status:** Recorded as a requirement for the runner project (F4); not
enforceable until a runner exists. The test plan marks the reproducing test with
`Type: defect`, so the runner can tell which tests to run twice.

## Requirements stage: the reference copy is behind the live pipeline

`References/SDLC-Orchestration/` has Requirements as a stub that provides no
artifact. The IDEA-30 run manifest
(`References/SDLC-Artifacts/Artifacts/IDEA-30/20261120_193043/manifest.json`)
shows a live stage that provides `requirements.md`, `open-questions.md`, and
`answers.md`. It also raises a delivery Epic whose requirements field
(`customfield_10047`) the ticket snapshot reads as `requirements` (see
`fetch-ticket.sh`). Test Case Generation reads that field, so it needs no new
input. A bug skips Requirements and enters at Analysis, so its `requirements`
field is empty.

The Step 1 compile check ran against the reference copy, and Step 4 was wired
against it too. Please share a current copy, or confirm the stage list, so that
the compile check can be repeated against the live stages.

**Status:** Requested: a current copy of the orchestration project. Nothing
blocks on it.

## Test Case Generation stage: improvements over the manual process

The new `test-case-generator` pack replaces the manual analyst handoff with
two validated artifacts: a traceable `testplan.md` and a complete
repository-relative `testcode.md`. It preserves the useful two-pass intent
from the former requirement and test-step prompts, including child-over-parent
conflict resolution, UI-only scope, positive/negative coverage, and a human
review point, while adding:

- Stable fully qualified TestNG identities for later Xray and JUnit binding, and
  a separate `manual.` identity for manual tests.
- Explicit `NO_UI_SURFACE` handling instead of an ambiguous empty plan.
- Complete file blocks and suite entries that can be reviewed before they are
  copied into the test repository, with additive-only changes to existing files.
- A read-only base-branch source map that prevents generated implementation
  code from biasing requirements.
- Contract validation rules for traceability, duplicate identities, allowed
  paths, status envelopes, and recomputed coverage metrics.
- A contract check in code (bash, jq, awk) that fails the stage on a malformed
  plan or code, with each finding in the task log, so a bad output never reaches
  the exit assessor or the next stage. A Coverage summary mismatch only warns,
  because the exit assessor scores it.
- A source column in the traceability table, so the F12 ranking is auditable,
  and a Jira issue column that Step 6 uses to link each test.
- Coverage metrics and a BDD breakdown in every plan, and a stated reason for
  every manual test.
- Entry and exit assessors that score from rubrics instead of a self-reported
  figure. The exit assessor lists the browser-observable requirements itself
  before it reads the plan, so the plan's own Coverage % cannot anchor it, and it
  recomputes every metric.
- Missing test data is named as a `DATA:` gap (table, columns, unique values,
  parent rows, cleanup) instead of invented, so a later data owner can act on it.
- A fixed rule for what is manual: only what a person must judge or a browser
  cannot observe. A missing locator or missing data is a gap on an automated
  test.

**Status:** Contracts, the three packs (generator, entry assessor, exit
assessor), and the stage flow with its contract check are implemented; Steps 2
to 4 are done. Nothing has been run against a ticket, by decision (R13, R14).
Xray registration and the live runner remain subsequent project steps.

## Test Case Generation stage: post the plan to Jira for review

The manual process ended with an analyst reviewing the test cases before anything
reached Xray, and the pack keeps that as a "human review point". The stage as
wired has none: `testplan.md` is published to the artifact space, and the next
stage starts when the exit check passes. The use-case catalogue marks no human
governance on this stage, so a reviewer finds the plan only by opening the
artifact folder.

Recommended: add a `jira.post-update` task after the exit check, as Design does
with `transmitFailed: false`, to comment on the ticket with links to
`testplan.md` and `confidence-coverage.md`. It does nothing on a dry run, and it
gives the Senior QA Analyst a place to read the plan without a gate. Alternative:
make the plan a human gate (`humanGatedStages`), which pauses the run and costs a
decision per ticket. Neither is wired, because both change what the stage does
on a real ticket.

**Status:** Share with the Test Case Generation champion (Senior QA Analyst).

## Jira fetch: add Component, child tickets, issue links, and labels

`jira.fetch-ticket` writes `input.json` with the ticket's id, summary,
description, status, `design` and `requirements` fields, issue type (as
`bugType`), first fix version, comments, attachments, and linked pull requests.
Test Case Generation also needs:

- **Components.** Xray folders and reuse detection key on the Component (F5). The
  manual process read it from the Jira issue.
- **Child tickets.** F12 ranks child requirements first, and the manual process
  collected them for Epics and Stories. The snapshot has none, so the
  child-overrides-parent rule cannot run.
- **Issue links.** A defect's expected behavior can sit in a linked requirement.
- **Labels.** Xray labels come from the ticket.
- **Image attachments.** The fetch keeps only `text/*` and `application/pdf`
  files and drops images unless image support is enabled. Mockups and bug
  screenshots are then missing. METERSENSE-11680 lost a screenshot and a screen
  recording.

**Status:** Requested from the owner of the orchestration project. Until then,
the plan records `Component: none` and flags the gaps (R6).

## Second Security Review: take the generated test code as input

The use-case catalogue (step 22) has this stage review the generated code and
the generated Playwright suite for security gaps and committed secrets. Its
stage declaration has empty `resources`, so it receives neither. Test Case
Generation delivers `testCode` (`testcode.md`). The stage can declare it, and
`code`, as artifact resources.

**Status:** Requested from the owner of the Second Security Review stage.

## Orchestration: second confidence call and test repository snapshot

Test Case Generation checks confidence twice (F6). The `confidence` block runs
the entry check, and a second `agents.confidence-check` call with
`point: coverage` runs after the work. Six points need the owner's agreement:

- **Agent paths.** Stage flows request agents as `AIDLC/<Stage>/<agent>`, such
  as `AIDLC/Design/solution-designer`. The reference `AIAgents` clone is flat
  (`<agent>/manifest.json`) and has no `AIDLC` folder, and a local run resolves
  `LOCAL_AI_AGENTS/<agent name>` literally. Please confirm the live layout. It
  decides whether the three Test Case Generation packs move into
  `AIDLC/TestCaseGeneration/` or the stage names change.
- **An `exit` key under `confidence`.** The graph compiler ignores unknown keys,
  so the stage flow reads `confidence.exit` itself. The alternative is
  first-class support for several checks per stage.
- **The test repository as a dependency.** `baseBranch` is the product
  repository, so the agent also needs the Playwright repository. A branch
  resource needs a second `projects.yml` entry and credentials, so the stage
  reads a snapshot from the dependencies folder (`test-repository`) until then.
  Who populates and refreshes that folder? A stale snapshot means generated code
  compiles against old page objects.
- **Skip semantics.** `skipConfidenceCheck=First` skips both checks when the
  stage is the starting stage.
- **Extra files in a stage folder.** The stage's `check-result.sh` needs
  `validate.jq`, and the folder also holds `fixtures/` and `test.sh`. The stages
  README says a stage folder holds its workflow and the scripts it calls. Is a
  test fixture folder acceptable there, or should the validator and its tests
  live elsewhere?
- **The jq version.** The Dockerfile installs jq from apt without a pin, and the
  validator was tested only with jq 1.7.1. Please confirm the version the image
  carries, or pin it as the base image and `yq` are pinned.

**Status:** Needs agreement from the owner of the orchestration project.

## Test case generation: exit coverage gate

The use-case catalogue describes one entry confidence gate for Test Case
Generation. F6 adds an exit gate on coverage. A failed exit check stops
`SecondSecurityReview`, `Review`, and `Verification`, so a weak test plan blocks
a good code change. [Best practices and alternatives](#best-practices-and-alternatives)
describes an advisory alternative.

**Status:** Share with the Test Case Generation champion (Senior QA Analyst).

## Test repository: fix suite wiring

- `pom.xml` sets `suiteXmlFile` to `testng.xml`, but the file is `testNg.xml`. A
  case-sensitive (Linux) runner cannot find it.
- `pom.xml` hard-codes `suiteXmlFiles`, so `-DsuiteXmlFile` selects nothing.
  Surefire's user property for the parameter is `surefire.suiteXmlFiles`, and an
  explicit POM value takes precedence over a command-line property.
- The suites list `tests.RateImpactAnalysisTest` (all three) and
  `tests.IntelligentSegmentationTest` (`regressiontest.xml`), which do not exist.
  `tests.InstantInsightTest` exists and is missing from `testNg.xml`.
- Surefire 3.6.0 and later reject `suiteXmlFiles` and fail the build when it is
  set. The repository uses 3.2.5 today. Plan group-based selection (`groups`)
  before upgrading.

**Status:** Requested from the owner of the test repository (R7). Until then,
generated Verification notes use `mvn test -Dtest=<Class>`.

## Best practices and alternatives

1. **Run the contract checks in code.** Done in Step 4: `check-result.sh` and
   `validate.jq` (bash, jq, awk; the Kestra image has no Python) enforce both
   contracts, and `test.sh` runs them against the two samples in `fixtures/` and
   50 seeded violations and warnings. Keep the samples and add one case per new rule.
2. **Score the exit check from the metrics.** A model's self-reported confidence
   is poorly calibrated. Give the exit assessor a rubric built on recomputed
   Coverage %, coverage ratio, unresolved locators, and input levels. Keep the
   threshold hidden from it, as the helper already does.
3. **Bind results by key after the first import.** Definition matching creates a
   duplicate silently when a name differs, while an explicit `test_key` that does
   not resolve raises an error. Alternative: import by Definition once, read the
   keys back with `getTests`, store them in the artifact space, and inject
   `test_key` on later runs, so a rename fails fast. The cost is one more
   artifact and read call. The `xray-testng-extensions` annotations also give
   explicit keys, but they put Jira awareness into the test repository.
4. **Give unresolved-locator tests their own failure class.** Such a test fails
   by design, and F17 would call it a genuine failure and mark a blocker.
   Classify the `UNRESOLVED:` message as a separate class (recommended), or keep
   the test in a group that the suites exclude until the locator is resolved.
5. **Consider an advisory exit gate.** The use-case catalogue gates regression
   failures at the pull request instead of failing the stage. The same pattern
   fits coverage: write `confidence-coverage.md`, comment on the ticket, and keep
   only the entry gate blocking. It needs a non-blocking mode in
   `agents.confidence-check`, which fails the stage below threshold today. F6
   stands until you change it.
6. **Keep a golden set of tickets.** Keep three to five tickets as fixtures for
   later checks of the plan and code contracts: a defect without the bug
   template, a new feature with new UI, an extension, and a ticket with no UI.
   METERSENSE-11680 covers the first and IDEA-30 the second; none is a
   completion requirement for any step.
7. **Verify data-driven names early.** Check what Surefire 3.2.5 writes as the
   `testcase` name for `@DataSource` rows (V4) before Step 6. If it adds a
   suffix, Definition binding splits one method into many tests.
8. **Keep generated changes small.** Extending existing classes
   (`Mode: extend`) follows the repository's convention but can conflict with
   other tickets' edits. Decide the Step 5 branch and merge strategy before
   write-back.
9. **Nest the packs now.** The packs sit flat in `AIAgents/` and map to
   `AIDLC/TestCaseGeneration/<agent>` at migration (`AGENT-PATH`). If the owner
   confirms that layout, nesting them now makes a local run and the stage flow
   identical to production and avoids renaming three agents later. The cost is
   moving the packs and updating the three names in the stage block
   (`agent`, `confidence.agent`, `confidence.exit.agent`) and the docs.
10. **Decide the test-data owner before Step 7.** Every `DATA:` gap is a test at
    L2 or below that cannot pass until its data exists. Recommended: a
    data-generator agent that writes the JSON file and `databaseFixtures` from
    the plan's `DATA:` gaps, which already list table, columns, parent order,
    and cleanup. Alternative: an analyst-maintained data file per feature.
11. **Treat the first live ratings as calibration.** Neither rubric has run
    against a ticket. When the stage first runs, read each rating table against
    its contract before trusting the threshold, and adjust any deduction that
    over- or under-penalizes. Set the final thresholds (Q20) from those ratings
    rather than from the default of 50.
12. **Let the generator repair its own output once.** A contract failure stops
    the stage with no retry, so a missing field in an otherwise sound plan costs
    a whole run. Alternative: on a failed contract check, run the generator a
    second time with the findings appended to its prompt, then check again. It
    doubles the agent cost on a failure and needs a second `run-container` call
    and a bounded loop. Decide after the first live runs show how often the
    check fails and why.
13. **Detect contract drift.** `validate.jq` hand-encodes the three contracts
    (R15), and nothing flags a contract edit. Pin each contract's SHA-256 in
    `test.sh` so a change fails the test until someone re-reads the validator,
    and run `test.sh` in the orchestration repository's CI. The cost is one
    hash to update per contract edit.
