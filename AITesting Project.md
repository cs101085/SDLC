# AI testing project plan

| Item | Value |
|---|---|
| Project | AI testing: Test Case Generation (stage 6b) and Verification (stage 8b) |
| Product | Metersense (feature modules inside Compass) |
| Plan stage | Planning draft, built against local clones in `References/` |
| Last updated | 2026-10-23 |

## Overview

### What this project is trying to accomplish

The AI-DLC pipeline has two QA stages that exist today only as stubs: **Test Case Generation**
(stage 6b) and **Verification** (stage 8b). This project fills them in so that, for any Jira ticket,
an AI agent:

1. Checks that the upstream requirements, analysis, and design are good enough to test from.
2. Writes a reviewable test plan with Cucumber-style scenarios and coverage metrics.
3. Generates executable Playwright (Java) UI tests in the style of the existing test repository.
4. Writes manual Xray tests only for requirements that automation cannot cover.
5. Registers the plan, tests, and executions in Xray, and reports run results back to Jira.

This replaces today's manual, analyst-driven process, which creates manual Xray test definitions
one at a time and has no path for importing execution results.

### Objectives

| # | Objective | Measure of success |
|---|---|---|
| O1 | Automate UI test design from requirements | Stage 6b produces a test plan and test code that satisfy the contracts for a sample ticket |
| O2 | Automate first, manual only as a last resort | Every manual test names the requirement it covers and why automation cannot |
| O3 | Make coverage visible | Each run reports test counts, Coverage %, and coverage ratio |
| O4 | Close the results gap | JUnit results import into an Xray Test Execution linked to the ticket |
| O5 | Avoid test sprawl | Existing tests are reused where suitable; single-use tests are retired quarterly |
| O6 | Make go-live mechanical | Every placeholder has a recorded migration path |

### Approach in brief

The project is a **draft that evolves into the real repositories**. Every external system (test
repository, source repository, artifact space, Jira, Xray) is represented by a placeholder that
resolves to a local folder today and becomes a live integration in the final phase. Test Case
Generation is designed through to working code. Verification is scaffolded so a future runner
container can attach.

## Scope

| In scope | Out of scope (later project or owner) |
|---|---|
| Functional, end-to-end **frontend (UI)** tests | Unit and integration tests (owned by Code Generation, stage 6a) |
| Entry (acceptance) and exit (coverage) confidence checks in stage 6b | Backend and API tests, except inserting rows to set up a UI test |
| `testplan.md` with metrics, BDD scenarios, and automated/manual breakdown | The Verification runner container and its environment |
| Generated Playwright test code in house style | The test data generator (see Q6) |
| Manual Xray tests for requirements automation cannot cover | Live connections to Bitbucket, Jira, Xray, or SDLC-Artifacts until Phase 5 |
| Xray payloads: Test Plan, two Test Executions, manual Test issues | Changes to any stage other than 6b and 8b |
| JUnit results collection and import into Xray | |
| Verification frame, failure classification, and stability report | |
| Placeholder registry, migration step, and `xray-integration.md` | |

Requests to change another stage go into [`suggestionbox.md`](suggestionbox.md) for discussion with
the owning department, not into another team's folder.

## Guiding decisions

These decisions shape the work. They are summarized here; detailed rationale lives in the
contracts. IDs are kept stable because other documents reference them.

| ID | Decision |
|---|---|
| F1 | **No bias toward generated code.** The agent tests the requirements, not the implementation. It reads the ticket, `analysis.md`, `design.md`, `implementation.md`, the test repository, and the source at `baseBranch`, and never the feature branch. New-UI locators come from the Design implementation plan. |
| F2 | **Frontend only, automated first.** Manual tests are the last resort. A ticket with no UI surface is reported as `NO_UI_SURFACE`, not guessed at. |
| F3 | **Every integration is a placeholder** until Phase 5 (see [Placeholders](#placeholders)). |
| F4 | **Verification runner is out of scope.** This project hands a runner the suite and an Xray Test Execution key, and defines how results are classified and reported. |
| F5 | **One product, many modules.** The Jira Component field identifies the module and drives Xray folder structure and reuse detection. |
| F6 | **Two confidence checks in stage 6b.** Entry checks input quality; exit checks coverage. Both thresholds start at 50 and stay configurable. |
| F7–F11 | **Xray Cloud by direct API from Kestra.** One Test Plan and two Test Executions (automated, manual) per run. Automated tests are created by JUnit import and bind by Definition; Kestra injects requirement links, summaries, and descriptions into the JUnit XML, so the test repository needs no Jira awareness. |
| F12 | **Sources are ranked by kind of work** (see [Appendix A](#appendix-a--source-ranking-f12)). Code supplies mechanics only and never decides an expected outcome. |
| F13 | **BDD lives in the test plan**, written Cucumber style. Generated code stays Java Playwright with TestNG; no Cucumber dependency. |
| F14 | **Testability is judged at stage 6b entry**, not by Analysis or Design. |
| F15 | **Code completeness depends on input detail level** (see [Appendix B](#appendix-b--input-detail-levels-f15)). Gaps are flagged and lower confidence; they do not stop the run. |
| F16 | **Stage 6b runs five steps:** entry check, test plan, automated tests, manual tests, exit check with coverage metrics. |
| F17 | **Verification classifies every failure:** infra (retry once), data (regenerate and rerun), genuine (notify and mark as blocker). It reports passed, retried, confirmed failures, and Stability Index. |

## Project plan matrix

Update **Status** and **Updated** as work progresses. Detailed procedures and exit
criteria for each step are in [Phase procedures](#phase-procedures).

**Status legend:** ⬜ Not started · 🟦 In progress · 🟨 Blocked · ✅ Done · ⏸️ Deferred

| Step | Phase | Task | Key deliverable | Depends on | Open decisions | Status | Updated | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | 1. Foundations | Settle the stage contracts | Contract proposal and placeholder registry | — | — | 🟦 In progress | 2026-10-23 | Draft in `SDLC-Orchestration/step-1-stage-contracts.md`; fields must be translated to the current stage schema |
| 2 | 1. Foundations | Define artifact and test contracts | `testplan-contract.md`, `testcode-contract.md`, naming rules | 1 | — | 🟦 In progress | 2026-10-23 | Files exist; need metrics, BDD, manual tests, Component, and Definition binding |
| 3 | 2. Test Case Generation | Build the agent pack | `test-case-generator` pack with entry and exit assessors | 2 | Q6 | 🟦 In progress | 2026-10-23 | Pack scaffold exists in `AIAgents/test-case-generator/`; not yet validated against a sample ticket |
| 4 | 2. Test Case Generation | Wire the stage | `main_test-case-generation.yml` | 3 | — | ⬜ Not started | | |
| 5 | 2. Test Case Generation | Emit test code to the artifact space | Generated test code and `TEST-REPO-WRITE` handover | 4 | — | ⬜ Not started | | |
| 6 | 2. Test Case Generation | Register tests in Xray | Xray client, payloads, `xray-integration.md` | 5 | Q11, Q13 | ⬜ Not started | | Verify Xray behaviors V1–V3 first |
| 7 | 3. Verification | Scaffold Verification and results path | `main_verification.yml`, runner interface | 6 | Q21, Q6 | ⬜ Not started | | |
| 8 | 4. Optimization | Detect reusable tests | Coverage index and reuse check | 5 | Q7 | ⬜ Not started | | Can run in parallel with 6–7 |
| 9 | 4. Optimization | Add the retention decision | Retention marker and review step | 5 | Q8 | ⬜ Not started | | Can run in parallel with 6–7 |
| 10 | 5. Migration | Migrate into the real repositories | Stages running live in SDLC-Orchestration | 1–9 | Q20 | ⬜ Not started | | Runs last |

### Milestones

| Milestone | Reached when | Status |
|---|---|---|
| M1 — Contracts agreed | Steps 1–2 done | ⬜ |
| M2 — Test Case Generation works end to end in the draft | Steps 3–6 done | ⬜ |
| M3 — Results path proven with hand-placed JUnit XML | Step 7 done | ⬜ |
| M4 — Reuse and retention in place | Steps 8–9 done | ⬜ |
| M5 — Live in production repositories | Step 10 done | ⬜ |

## Phase procedures

Each step lists its procedure in order, followed by the condition that closes it.

### Phase 1 — Foundations

Goal: agree what each stage consumes and produces before building anything.

#### Step 1 — Settle the stage contracts

1. Declare stage 6b and 8b inputs in `resources`, outputs in `deliverables`, and edges in
   `nextStages`, using the current stage schema.
2. Confirm stage 6b still follows `FirstSecurityReview`, runs parallel to `CodeGeneration`, and
   converges at `SecondSecurityReview`.
3. Require the ticket (with Component), analysis and design artifacts, `repoSlug`, `baseBranch`, and
   `bitbucketProject`. Do **not** require `featureBranch` (F1).
4. Configure an entry assessor before domain work and an exit assessor after it, both at threshold
   50 (F6).
5. Declare the placeholder registry.
6. Translate the draft's `requires`, `provides`, `predecessors`, `successors`, `gate`, and
   `confidenceThreshold` fields into the current schema.
7. Record cross-department requests in [`suggestionbox.md`](suggestionbox.md) and share them with
   the owners named there.

**Done when:** the declarations compile against the existing graph, every placeholder resolves to a
real path in this project, and affected stage owners have seen the suggestion box.

#### Step 2 — Define the artifact and test contracts

1. Define `testplan.md`: the `# Status` / `STATUS_CODE:MESSAGE` / `Confidence: <n>%` envelope and the
   `SUCCESS`, `NO_UI_SURFACE`, and failure outcomes.
2. Add the F16 sections: metrics, Cucumber-style BDD breakdown, and automated/manual frontend
   breakdown.
3. Define per-test records: requirement trace, scenario, numbered steps, expected outcome, input
   level, locator evidence, flagged gaps, and coverage rationale. Manual tests also state why
   automation cannot cover them.
4. Record ticket type, Jira Component, work kind, and the source ranking applied (F12).
5. Fix naming rules so test identity (`tests.<ClassName>.<methodName>`) stays stable.
6. Update the test-code contract to bind by Definition with injected properties, not `test_key`.

**Done when:** a sample document can be hand-written against the contracts and parsed by the stage.

### Phase 2 — Test Case Generation

Goal: a working stage 6b that turns a ticket into a test plan, test code, and Xray payloads.

#### Step 3 — Build the agent pack

1. Resolve Q6 (test data ownership).
2. Assemble the `test-case-generator` pack: `manifest.json`, `agents/`, `instructions/`, and the
   Step 2 contracts.
3. Add the Confidence Agent (entry) and the coverage assessor (exit).
4. Derive test design style from `References/Manual Process/Test Generator Agents/Plan/` and code
   style from `References/Test Repository/.github/copilot-instructions.md`.
5. State the F1 no-bias rule and F12 source ranking in the instructions.
6. Implement the five F16 steps, including defect-versus-expected-behavior handling and gap
   flagging instead of invented selectors.
7. Record improvements over the manual process in [`suggestionbox.md`](suggestionbox.md).

**Done when:** the pack runs against a sample ticket and produces a plan and code that satisfy the
contracts.

#### Step 4 — Wire the stage

1. Replace the stub's domain work with a read-only `agents.run-container` call (`cloneSource: true`,
   `newBranch: ""`) and `artifacts.persist-agent-output`.
2. Set `stub: false` and declare the deliverables.
3. Place the entry assessor before domain work and the exit assessor after it.

**Done when:** a run writes `confidence-entry.md`, the test plan, `stage-record.json`, and
`confidence-exit.md`; inadequate inputs stop at the entry check; and the container mounts show no
feature branch.

#### Step 5 — Emit test code to the artifact space

1. Generate test classes, page objects, data JSON, and suite XML entries.
2. Write them through `ARTIFACT-SPACE`; mark the push path `TEST-REPO-WRITE`.
3. Record what live write-back needs: a second repository mapping, credentials, branch strategy,
   and merge owner.

**Done when:** a reviewer can copy the files into the test repository and they compile and are
discoverable through the suite XML.

#### Step 6 — Register tests in Xray

1. Resolve Q11 and Q13, and run the empirical checks V1–V3.
2. Authenticate and cache the token for under an hour.
3. Create the Test Plan from `testplan.md`; update it with metrics after the exit check.
4. Create the automated Test Execution, linked to the ticket, Test Plan, and test code. Keep its key
   for Step 7.
5. Create each manual test as type **Manual**, in its Component folder, linked to the ticket and
   added to the Test Plan.
6. Create the manual Test Execution, linked to the ticket and Test Plan.
7. Prepare the JUnit properties for each automated method: `requirements`, `test_summary`,
   `test_description`, and `tags`.
8. Keep creation sequential for rate limits. Behind `XRAY-API`, write payloads beside the artifacts
   instead of sending them.
9. Write `xray-integration.md`: endpoints, credentials, result binding, configurable fields, failure
   modes and safeguards, and a manual `curl` check.

**Done when:** sending the payloads would create the Test Plan, both Test Executions, and correctly
typed and linked manual tests, and the guide can be followed on its own.

### Phase 3 — Verification

Goal: a stage 8b frame and a complete results path, ready for a runner to attach.

#### Step 7 — Scaffold Verification and the results path

1. Resolve Q21 (failure classification and blocker definition).
2. Add the stage frame with a marked `VERIFICATION-RUNNER` placeholder.
3. Write the runner contract:

   | Direction | Item |
   |---|---|
   | Into the runner | Suite XML, target environment, automated Test Execution key |
   | Out of the runner | JUnit XML (`target/surefire-reports/TEST-*.xml`) for every run including retries, exit code, duration |

4. Record failure actions (F17) and, for defect tickets, the red-then-green requirement: fail on the
   base branch, pass on the fix.
5. Build the results path: collect JUnit XML → classify failures → inject Step 6 properties →
   import with `testExecKey` → read back and fail if any result bound outside the target project →
   write the Verification report.

**Done when:** hand-placed JUnit XML, including a retried infra failure and a genuine failure, is
stored, classified, injected, turned into a correct import request, and reported with the correct
metrics and Stability Index.

### Phase 4 — Optimization

Goal: keep the suite lean. Can start once Step 5 is done.

#### Step 8 — Detect reusable tests

1. Resolve Q7 (match rule and what is queried).
2. Build an index of existing coverage, respecting the 100-ID retrieval cap and sequential calls.
3. Add a reuse check to stage 6b before anything is authored.

**Done when:** an already-covered ticket produces a reuse recommendation instead of a new test.

#### Step 9 — Add the retention decision

1. Resolve Q8 (who decides and what enforces it).
2. Add a review step that marks each generated test as durable or single-use.
3. Carry the marker into the test code (as a TestNG group) and into Xray.
4. Define the quarterly job that retires single-use tests.

**Done when:** a bug-ticket test is marked single-use and excluded from regression, and an epic test
is marked durable and included.

### Phase 5 — Migration

Goal: retire placeholders and run live.

#### Step 10 — Migrate into the real repositories

1. Resolve Q20 (final thresholds).
2. Move each top-level folder into its target repository.
3. Replace placeholders in dependency order: `SOURCE-REPO-READ` and `TEST-REPO-READ`, then
   `ARTIFACT-SPACE` and `TICKET-INPUT`, then `XRAY-API` and `JUNIT-RESULTS`, then `TEST-REPO-WRITE`.
4. Supply credentials as Kestra globals.
5. Hand `VERIFICATION-RUNNER` and its interface to the runner project.

**Done when:** no `PLACEHOLDER:` marker remains except `VERIFICATION-RUNNER`, and a full pipeline run
produces test artifacts and Xray records end to end.

## Risks and assumptions

| # | Risk or assumption | Impact | Mitigation | Status |
|---|---|---|---|---|
| R1 | Xray Definition matching silently creates duplicates on a mismatch, and can match tests in other projects | Duplicate or misattributed tests | Confirm the separator (V1), keep identity stable, read back after import and fail on cross-project binding | Open |
| R2 | Data-driven `@DataSource` rows collapse onto one Xray test | Per-row results are lost | Decide Q11 before Step 6 | Open |
| R3 | Test data generator is undefined | Tests cannot run; data-failure recovery is unspecified | Decide Q6 before Step 3 | Open |
| R4 | Upstream artifacts may lack locators or detail | Incomplete code | Entry confidence check; flag gaps and lower confidence | Mitigated by design |
| R5 | No runner exists | Defect red-then-green cannot be enforced | Documented as a runner requirement (F4) | Accepted |
| A1 | The Jira bug template requires reproduction steps, expected result, and actual result | Defect tests depend on it | Recorded in [`suggestionbox.md`](suggestionbox.md) | Assumed |

### To verify against the Xray tenant

| # | Behavior | Needed by | Status |
|---|---|---|---|
| V1 | The separator Xray uses between `classname` and `name` for JUnit Definitions | Step 6 | ⬜ |
| V2 | Whether setting the project default test type changes import auto-provisioning from Generic | Step 6 | ⬜ |
| V3 | Whether `requirements` and `tags` properties apply on update as well as creation | Step 6 | ⬜ |

## Decision log

### Open

| # | Question | Decide before | Owner | Status |
|---|---|---|---|---|
| Q6 | Who generates test data, and from what? | Step 3 | TBD | Open |
| Q7 | What counts as a matching existing test, and what is queried? | Step 8 | TBD | Open |
| Q8 | Who makes the retention decision, and what enforces it? | Step 9 | TBD | Open |
| Q11 | How are data-driven results reported? | Step 6 | TBD | Open |
| Q13 | Which Test Plan and Test Execution fields must be set? | Step 6 | TBD | Open |
| Q20 | What are the final confidence thresholds? | Step 10 | TBD | Open |
| Q21 | How is a failure classified, and what does "mark as a blocker" set? | Step 7 | TBD | Open |

### Settled

| # | Question | Outcome |
|---|---|---|
| Q1 | How the agent reads existing tests | Placeholder resolving to the local clone (F3) |
| Q2 | Manual-to-automated linking | Link to the requirement; record superseded manual key as a label (F9) |
| Q3 | Xray test type | Automated: Generic via import. Manual: Manual type (F8) |
| Q4 | Xray issues per test | Automated tests share one Test Execution; manual issues only where automation cannot cover (F8) |
| Q5 | Test identity | Fully qualified method is the Definition (F10, F11) |
| Q9 | Confidence thresholds and timing | Threshold 50; entry and exit checks (F6) |
| Q10 | Product scope | Metersense only (F5) |
| Q12 | Execution ticket scope | One Test Plan and two Test Executions per run (F8) |
| Q14 | Modules and Xray folders | Jira Component field (F5) |
| Q15 | Tests below input level L3 | Generated with gaps flagged and confidence lowered (F15) |
| Q16 | Element IDs for new UI | From the Design implementation plan (F1, F15) |
| Q17 | Testability in upstream confidence | Judged at stage 6b entry (F14) |
| Q18 | Jira bug template | Assumed (A1) |
| Q19 | Entry and exit checks in one stage | Allowed (F6) |

## Deliverables

### By repository

New files go in a top-level folder named for their target repository.

| Folder | Contents |
|---|---|
| `SDLC-Orchestration/` | `main_test-case-generation.yml`, `main_verification.yml`, scripts, Xray client, `xray-integration.md` |
| `AIAgents/` | `test-case-generator` pack, and any reuse or retention-review agent |
| `Test Repository/` | Instruction files and templates that shape generated code |
| `SDLC-Artifacts/` | Sample outputs for both stages, including generated code and a sample JUnit XML |

### Run artifacts

| Artifact | Purpose |
|---|---|
| `confidence-entry.md` | Rating of whether upstream inputs are good enough to test from |
| `testplan.md` | Reviewable test cases: metrics, BDD scenarios, automated/manual breakdown |
| `testcode.md` | Generated Playwright tests in house style |
| `confidence-exit.md` | Coverage rating from Coverage % and coverage ratio |
| `xray-payload.json` | Request payloads for the Test Plan, Test Executions, manual tests, and JUnit properties |
| `junit-results.xml` | Actual outcomes, imported into the automated Test Execution |
| `verification-report.md` | Failure classifications, actions, and stability metrics |

## Placeholders

A placeholder marks where this draft stands in for a live integration. Write each as a comment or
resolvable variable so migration is a search, not an audit:

```
PLACEHOLDER: <NAME>
  resolves-to: <path in this project today>
  becomes:     <the real integration>
  on-migration: <what has to change>
```

| Name | Resolves to today | Becomes | Status |
|---|---|---|---|
| `TEST-REPO-READ` | `References/Test Repository/` | Read-only clone of the test repository | Placeholder |
| `TEST-REPO-WRITE` | `Test Repository/README.md` | Branch, commit, and push to the test repository | Placeholder |
| `SOURCE-REPO-READ` | `References/metersense-release-5.1/` | Read-only `git.clone-source` at `baseBranch` | Placeholder |
| `ARTIFACT-SPACE` | `SDLC-Artifacts/README.md` | `artifacts.publish-workspace` into SDLC-Artifacts | Placeholder |
| `TICKET-INPUT` | `Example Tickets/METERSENSE-10101.md` | `Jira/input.json` staged by the orchestrator | Placeholder |
| `XRAY-API` | `SDLC-Artifacts/README.md` | Direct Xray API calls from Kestra | Placeholder |
| `JUNIT-RESULTS` | `SDLC-Artifacts/README.md` | `target/surefire-reports/TEST-*.xml` from a real run | Placeholder |
| `VERIFICATION-RUNNER` | `SDLC-Orchestration/step-1-stage-contracts.md` | Runner container from a later project | Handed off, not retired |

Placeholders are retired only in Step 10. Until then, a placeholder that calls a real service is a
defect, because it makes the draft non-reproducible.

## Change control

- Update the [project plan matrix](#project-plan-matrix) when a step changes status or scope.
- Record new questions in the [decision log](#decision-log) with the step they block; move them to
  **Settled** with the outcome when decided.
- Changes that affect another stage go to [`suggestionbox.md`](suggestionbox.md), not into that
  stage's folder.

## Restrictions

1. Use Kestra for orchestration.
2. Keep artifacts, orchestration, and test automation code in separate repositories.
3. Do not alter any file under `References/`.
4. Create new files in the top-level folder, separated by repository name.
5. Keep code, readmes, and documentation efficient, clear, and non-verbose.
6. Follow the Google developer documentation style guide.

## Resources

- [`use-cases.md`](use-cases.md): the reference use case and stage sequence.
- [`suggestionbox.md`](suggestionbox.md): cross-department change requests.
- `References/`: current work from other departments. Read-only.
- `Example Tickets/METERSENSE-10101.md`: a sample ticket.
- `.github/instructions/xray-graphql.instructions.md`: authoritative reference for every Xray call.
- [Import Execution Results REST v2](https://docs.getxray.app/space/XRAYCLOUD/44577176/Import+Execution+Results+-+REST+v2)
- [Taking advantage of JUnit XML reports](https://docs.getxray.app/space/XRAYCLOUD/44564745/Taking+advantage+of+JUnit+XML+reports)
- [Global Settings: Test Types](https://docs.getxray.app/space/XRAYCLOUD/44566485/Global+Settings%3A+Test+Types)
- [xray-testng-extensions](https://github.com/Xray-App/xray-testng-extensions): optional upgrade
  path; not used.

## Appendix A — Source ranking (F12)

Stage 6b classifies each ticket, then ranks sources. When sources conflict, the higher rank wins.

| Kind | Rule |
|---|---|
| `defect` | `bugType` is `Bug` |
| `extension` | Not a bug; the named screens already exist |
| `new-feature` | Not a bug; the named screens do not exist yet |

| Rank | `new-feature` | `extension` | `defect` |
|---|---|---|---|
| 1 | Ticket acceptance criteria and child tickets | Ticket acceptance criteria and child tickets | Expected behavior in the bug or linked requirement |
| 2 | `analysis.md` | `analysis.md` | Observed defect: reproduction steps and actual result |
| 3 | `design.md`, `implementation.md` | `design.md`, `implementation.md` | `analysis.md` and design fix intent |
| 4 | Test repository (mechanics only) | Existing page objects and tests (mechanics and regression evidence) | Base-branch source and existing tests (mechanics) |
| 5 | Base-branch source (mechanics only) | Base-branch source (mechanics only) | — |

## Appendix B — Input detail levels (F15)

| Level | Inputs available | What can be produced |
|---|---|---|
| L1 | Acceptance criteria only | A Gherkin scenario; no runnable code |
| L2 | L1, plus screen, navigation, role, labels, controls, and messages | A scenario and code draft using role or label locators |
| L3 | L2, plus stable element IDs, exact data, and database setup rows | Complete code in house style |
