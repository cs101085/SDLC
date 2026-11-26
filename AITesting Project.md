# AI testing project plan

| Item | Value |
|---|---|
| Project | AI testing: Test Case Generation (stage 6b) and Verification (stage 8b) |
| Product | Metersense (feature modules inside Compass) |
| Plan stage | Planning draft, built against local clones in `References/` |
| Last updated | 2026-11-25 |

## Overview

### What this project is trying to accomplish

The AI-DLC pipeline has two QA stages that exist today only as stubs, which log and succeed with no
`resources` or `deliverables`: **Test Case Generation** (stage 6b, `stage.test-case-generation`) and
**Verification** (stage 8b, `stage.verification`). This project fills them in so that, for any Jira ticket,
an AI agent:

1. Checks that the upstream requirements, analysis, and design are good enough to test from.
2. Writes a reviewable test plan with Cucumber-style scenarios and coverage metrics.
3. Generates executable Playwright (Java) UI tests in the style of the existing test repository.
4. Writes manual Xray tests only for requirements that automation cannot cover.
5. Registers the plan, tests, and executions in Xray, and reports run results back to Jira.

This replaces today's manual, analyst-driven process (two Forge apps: a MatchaAI-backed generator and
an Xray importer), which creates manual Xray test definitions one at a time and has no path for
importing execution results.

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
| F6 | **Two confidence checks in stage 6b.** Entry checks input quality; exit checks coverage. Both thresholds start at 50 and stay configurable. Both run through `agents.confidence-check`, whose `point` input names the assessment (`confidence-<point>.md`). |
| F7–F11 | **Xray Cloud by direct API from Kestra.** One Test Plan and two Test Executions (automated, manual) per run. Automated tests are created by JUnit import and bind by Definition; Kestra injects requirement links, summaries, and descriptions into the JUnit XML, so the test repository needs no Jira awareness. |
| F12 | **Sources are ranked by kind of work** (see [Appendix A](#appendix-a--source-ranking-f12)). Code supplies mechanics only and never decides an expected outcome. |
| F13 | **BDD lives in the test plan**, written Cucumber style. Generated code stays Java Playwright with TestNG; no Cucumber dependency. |
| F14 | **Testability is judged at stage 6b entry**, not by Analysis or Design. |
| F15 | **Code completeness depends on input detail level** (see [Appendix B](#appendix-b--input-detail-levels-f15)). Gaps are flagged and lower confidence; they do not stop the run. |
| F16 | **Stage 6b runs five steps:** entry check, test plan, automated tests, manual tests, exit check with coverage metrics. |
| F17 | **Verification classifies every failure:** infra (retry once), data (regenerate and rerun), genuine (notify and mark as blocker). It reports passed, retried, confirmed failures, and Stability Index. |
| F18 | **Two identity namespaces.** Automated tests are `tests.<Class>.<method>`; manual tests are `manual.<Feature>.<behavior>`. Both are stable keys (see the naming rules). |
| F19 | **Metrics are recomputed from the plan.** Coverage % is covered requirements ÷ requirements. Coverage ratio is tests ÷ covered requirements (proposed; see Q22). The exit rating derives from them. |
| F20 | **Generated code is additive.** Files are new, or `Mode: extend` keeps every original line. Tests for an existing feature go into its existing class. |
| F21 | **The test repository enters as a `dependency` resource** until Step 10 (`TEST-REPO-READ`), because a branch resource needs a `projects.yml` entry. |
| F22 | **`Type: defect` marks the reproducing test**, so the runner can apply red-then-green (F4). |

## Project plan matrix

Update **Status** and **Updated** as work progresses. Detailed procedures and exit
criteria for each step are in [Phase procedures](#phase-procedures).

**Status legend:** ⬜ Not started · 🟦 In progress · 🟨 Blocked · ✅ Done · ⏸️ Deferred

| Step | Phase | Task | Key deliverable | Depends on | Open decisions | Status | Updated | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | 1. Foundations | Settle the stage contracts | Contract proposal and placeholder registry | — | — | ✅ Done | 2026-11-25 | `SDLC-Orchestration/step-1-stage-contracts.md`: current schema, mounts, confidence tasks, registry. Compiles against the reference graph from three start stages; three negative controls fail as expected. Every placeholder resolves to a real path. `suggestionbox.md` requests do not gate any step |
| 2 | 1. Foundations | Define artifact and test contracts | `testplan-contract.md`, `testcode-contract.md`, naming rules | 1 | Q22 (non-blocking) | ✅ Done | 2026-11-25 | Pack and `SDLC-Orchestration/` copies are byte-identical. A scratch parser accepts two hand-written samples (defect, no UI surface) and catches 30 of 30 seeded violations; the stage's own parser is `validate.jq` (Step 4) |
| 3 | 2. Test Case Generation | Build the agent pack | Generator pack plus entry and exit assessor packs | 2 | Q6 (deferred), Q22 (settled) | ✅ Done | 2026-11-25 | Three flat packs in `AIAgents/`: `test-case-generator`, `test-case-generator-confidence` (rubric: Requirements 35, Accuracy 25, UI mechanics 25, Consistency 15), and `test-case-generator-coverage-confidence` (Coverage 40, Test design 20, Plan and code consistency 20, Input level 15, Metrics accuracy 5). Contract copies byte-identical; manifests match their folders. Not run against any ticket, by decision (R13). Packs request `AIDLC/...` paths at migration (`AGENT-PATH`) |
| 4 | 2. Test Case Generation | Wire the stage | `main_test-case-generation.yml`, `check-result.sh` | 3 | Q23 (settled) | ✅ Done | 2026-11-25 | In `SDLC-Orchestration/test-case-generation/`: the stub is replaced by the entry check, the generator (`readOnly: true`, no feature branch), the contract check, and the exit check (`point: coverage`). `check-result.sh` and `validate.jq` (bash, jq, awk) pass `SUCCESS` and `NO_UI_SURFACE` and enforce both contracts. Coverage summary mismatches warn instead of failing (Q23). `fixtures/` and `test.sh` hold the two samples and 55 checks (5 on the samples, 50 seeded violations or warnings), all passing on the last full run. Not compiled against the stage graph or run, by decision (R14) |
| 5 | 2. Test Case Generation | Emit test code to the artifact space | Generated test code and `TEST-REPO-WRITE` handover | 4 | — | ⬜ Not started | | Decide the branch and merge strategy for `Mode: extend` files (R12). Verify discovery (R7) |
| 6 | 2. Test Case Generation | Register tests in Xray | Xray client, payloads, `xray-integration.md` | 5 | Q11, Q13 | ⬜ Not started | | Verify Xray behaviors V1–V4 first. Declare the `automatedExecutionKey` value deliverable |
| 7 | 3. Verification | Scaffold Verification and results path | `main_verification.yml`, runner interface | 6 | Q21, Q6 | ⬜ Not started | | Add an `unresolved` failure class (R9). Keep `statusCode` `SUCCESS` on regression failures. Merge or archive retried JUnit files |
| 8 | 4. Optimization | Detect reusable tests | Coverage index and reuse check | 5 | Q7 | ⬜ Not started | | Can run in parallel with 6–7 |
| 9 | 4. Optimization | Add the retention decision | Retention marker and review step | 5 | Q8 | ⬜ Not started | | Can run in parallel with 6–7 |
| 10 | 5. Migration | Migrate into the real repositories | Stages running live in SDLC-Orchestration | 1–9 | Q20 | ⬜ Not started | | Runs last |

### Milestones

| Milestone | Reached when | Status |
|---|---|---|
| M1 — Contracts agreed | Steps 1–2 done | ✅ |
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
3. Require `ticket` (`Jira/input.json`), `design.md`, `implementation.md`, and the
   `baseBranch` branch checked out as `source`. Step 4 removed `analysis.md` from this list. The
   repository and project come from
   `_config/projects.yml`, not from `repoSlug` or `bitbucketProject` values. Do **not** require
   `featureBranch` (F1).
4. Set `variables.stage.confidence` for the entry assessor, which runs before domain work, and add a
   second `agents.confidence-check` task after domain work for the exit assessor with its own
   `point`. Both use threshold 50 (F6). The exit settings sit under `confidence.exit`, which only
   this stage's flow reads.
5. Declare the placeholder registry.
6. Translate the draft's fields into the current schema: `requires` to `resources`, `provides` to
   `deliverables`, `successors` to `nextStages`, and `gate` plus `confidenceThreshold` to
   `confidence`. Drop `predecessors`; the graph derives them from other stages' `nextStages`.
7. Record cross-department requests in [`suggestionbox.md`](suggestionbox.md). The box is a
   reference for later meetings; no step waits on an owner's reply.

**Done when:** the declarations compile against the existing graph and every placeholder resolves
to a real path in this project.

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

1. Settle how the pack handles test data (Q6). Ownership stays open for a future
   data-generator project; the pack reuses existing data and names the rest as
   `DATA:` gaps.
2. Assemble the `test-case-generator` pack: `manifest.json`, `agents/`, `instructions/`, and the
   Step 2 contracts.
3. Add the entry assessor and the coverage assessor (exit), each as its own pack, because the
   runtime runs one agent per pack. Stage flows request agents by path, for example
   `AIDLC/Analysis/jira-reviewer-confidence`; the packs are flat until migration
   (`AGENT-PATH`).
4. Derive test design style from `References/Manual Process/Test Generator Agents/Plan/` and code
   style from `References/Test Repository/.github/copilot-instructions.md`.
5. State the F1 no-bias rule and F12 source ranking in the instructions.
6. Implement the five F16 steps, including defect-versus-expected-behavior handling and gap
   flagging instead of invented selectors.
7. Record improvements over the manual process in [`suggestionbox.md`](suggestionbox.md).

**Done when:** the generator and both assessor packs exist with valid manifests, the shared contract
copies are byte-identical, and the instructions state the F1 rule, the F12 ranking, and the five F16
steps. No sample run is required.

#### Step 4 — Wire the stage

1. Replace the stub's domain work with an `agents.run-container` call that sets `readOnly: true` and
   lists the deliverables in `filesOut`. Declare `baseBranch` as `{ kind: branch, as: source }`, as
   Design does, and no feature branch, so the agent sees the source at `/work/resources/inputs/source`.
2. Set `stub: false` and declare `resources` and `deliverables`. Stage finalization publishes the
   deliverables to the artifact space, so no separate persist task is needed.
3. Add `check-result.sh` between the generator and the exit check. It reads the first `# Status`
   envelope of both outputs, passes `SUCCESS` and `NO_UI_SURFACE`, and fails any other status or
   contract violation with the findings. A Coverage summary mismatch is logged as a warning, not a
   failure (Q23). Ship it with `validate.jq`, `fixtures/`, and `test.sh`, in bash, jq, and awk.
4. Keep the entry assessor in `variables.stage.confidence` (default `point` `inputs`, writing
   `confidence-inputs.md`). Add the exit assessor after the contract check, passing the stage's
   resources and both outputs as `filesIn` and a distinct `point` such as `coverage`. It runs only
   when the contract check passes.

**Done when:** the flow declares the Step 1 resources, deliverables, and confidence block and requests
no feature branch; it runs the entry check, the generator, the contract check, and the exit check in
that order, so inadequate inputs stop at entry; and the contract check passes the two samples and
fails every seeded violation. No run is required.

#### Step 5 — Emit test code to the artifact space

1. Generate test classes, page objects, data JSON, and suite XML entries.
2. Write them through `ARTIFACT-SPACE`; mark the push path `TEST-REPO-WRITE`.
3. Record what live write-back needs: a second repository entry in `_config/projects.yml` (read
   through a branch resource that names it with `repo:`), credentials, branch strategy, and merge
   owner. Live write-back would use `git.branch` and `git.commit-and-push`.

**Done when:** a reviewer can copy the files into the test repository and they compile and are
discoverable through the suite XML (see R7).

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
2. Move each top-level folder into its target repository. Stage flows go under
   `_flows/stages/test-case-generation/` and `_flows/stages/verification/`; confirm the
   `AIDLC/<Stage>/<agent>` layout of `AGENT_CONFIGS_REPO` before placing the pack.
3. Replace placeholders in dependency order: `SOURCE-REPO-READ` and `TEST-REPO-READ`, then
   `ARTIFACT-SPACE` and `TICKET-INPUT`, then `XRAY-API` and `JUNIT-RESULTS`, then `TEST-REPO-WRITE`.
4. Supply credentials as Kestra secrets (`SECRET_*` entries in `.env`, read with `secret()`) and
   non-secret settings such as `XRAY_REGION` as globals.
5. Hand `VERIFICATION-RUNNER` and its interface to the runner project.

**Done when:** no `PLACEHOLDER:` marker remains except `VERIFICATION-RUNNER`, and a full pipeline run
produces test artifacts and Xray records end to end.

## Working notes for later steps

Facts found while closing Steps 1 to 4, checked against `References/`.

| Step | Note |
|---|---|
| 3 | The ticket snapshot (`input.json`) holds id, summary, description, status, `design`, `requirements`, issue type (`bugType`), first fix version, comments, attachments, and pull requests. It has no Component, children, issue links, or labels, and it keeps only text and PDF attachments (images and video are dropped). Decisions can sit in comments (METERSENSE-11680). |
| 3 | Assessors write `confidence-<point>.md`. `confidence.sh` takes the first `SUCCESS:` or `FAILED:` line and the first `Confidence:` line. A `FAILED:` line blocks the stage; the assessor never sees the threshold. |
| 3 | `baseBranch` is the Compass product repository. Page objects, conventions, and `.github/copilot-instructions.md` are in the test repository (F21). |
| 3 | The `requirements` field of a feature ticket carries `requirements.md`, because the live Requirements stage raises the delivery Epic with it (`customfield_10047`). A bug has none. |
| 4 | `analysis` is not a stage resource (removed in Step 4), so neither agent receives `analysis.md`, and a missing one is no longer named in `Missing`. The generator and both assessors, F12 (rank 2), and the contracts' `analysis` source still name it, and an assessor may report it missing. To keep it when it exists without requiring it, declare it `optional: true`, as FirstSecurityReview does for `requirements`. |
| 4 | `agents.run-container` stages files under `resources/<readsFrom>/`; the default is `inputs/artifacts/`. A `dependency` reaches the agent only through `filesIn`. The exit check's `filesIn` is the stage resources plus the two outputs. |
| 4 | Wire the assessors as `Step 1` lists: entry `point` `inputs`, exit `point` `coverage` with prompt `Perform TestCaseGeneration coverage confidence assessment.` The check flow appends the names of missing resources, and each assessor reads them as `Missing`. The entry assessor does not receive the test repository; the exit assessor does. |
| 4 | Agent names are flat today (`AGENT-PATH`). Use `test-case-generator`, `test-case-generator-confidence`, and `test-case-generator-coverage-confidence`, and rename them at migration. All three sit in the stage's `variables.stage` block (`agent`, `confidence.agent`, `confidence.exit.agent`), so migration edits one place. |
| 4 | Design's `check-result.sh` passes only `SUCCESS`. Test Case Generation also passes `NO_UI_SURFACE`. |
| 4 | `check-result.sh` reads the test repository snapshot (`TEST_REPO`) to check new files, additive `extend` files, and suite registration. Without it, it logs and skips those checks, so confirm `dependencies/test-repository` exists before the first run. |
| 4 | A contract failure skips the exit check, so no `confidence-coverage.md` exists for that run. The stage fails with the first five findings; the task log has all of them. |
| 4 | The generator's `stdout.log` is published as `test-case-generation_reasoning.txt`, as Design and Analysis publish theirs. |
| 5 | `validate.jq --arg mode files` lists each declared file's path, safety, and `extend` flag but not its content. Step 5 needs the contents to write the files. |
| 5 | Suites register classes as `<class>` lines in `smoketest.xml` and `regressiontest.xml`, and as a `<test>` block in `testNg.xml`. The test repository's Definition of done requires both. |
| 6 | `test_summary` and `test_description` overwrite an existing test's summary and description on every import, so injecting them each run replaces edits made in Jira. |
| 6 | A test that runs in several suites of one import gets a joint status: any failure fails it. |
| 7 | Use cases gate regression failures at the pull request, not on the stage. A test with an `UNRESOLVED:` locator fails by design (R9). |
| 7 | `Type: defect` marks the tests to run on the base branch and on the fix (F22). |

## Risks and assumptions

| # | Risk or assumption | Impact | Mitigation | Status |
|---|---|---|---|---|
| R1 | Xray Definition matching silently creates duplicates on a mismatch, and can match tests in other projects | Duplicate or misattributed tests | Confirm the separator (V1), keep identity stable, read back after import and fail on cross-project binding | Open |
| R2 | Data-driven `@DataSource` rows collapse onto one Xray test | Per-row results are lost | Decide Q11 before Step 6 | Open |
| R3 | Test data generator is undefined (Q6 deferred) | Tests with a `DATA:` gap cannot pass until data exists; data-failure recovery is unspecified | Step 3 pack reuses existing data and names the rest as `DATA:` gaps with input level L2 at most; decide the owner and the failure class (Q21) in Step 7 | Accepted for Step 3; open for Step 7 |
| R4 | Upstream artifacts may lack locators or detail | Incomplete code | Entry confidence check; flag gaps and lower confidence | Mitigated by design |
| R5 | No runner exists | Defect red-then-green cannot be enforced | Documented as a runner requirement (F4) | Accepted |
| R6 | The ticket snapshot `Jira/input.json` carries `bugType` (the issue type), `targetVersion`, `design`, `requirements`, and comments, but no Jira Component, child tickets, issue links, or labels, and it drops image attachments | F5 folder structure, reuse detection, and the F12 child-overrides-parent rule have nothing to key on; mockups and screenshots are missing | The plan records `Component: none` and flags the gaps; ask the orchestration owner to extend `jira.fetch-ticket` via [`suggestionbox.md`](suggestionbox.md) | Open |
| R7 | Test repository suite wiring: `pom.xml` names `testng.xml` but the file is `testNg.xml`; it hard-codes `suiteXmlFiles`, so `-DsuiteXmlFile` selects nothing; the suites list `RateImpactAnalysisTest` and `IntelligentSegmentationTest`, which do not exist; Surefire 3.6 and later reject `suiteXmlFiles` | Generated tests may not be discovered, and the runner may run the wrong suite or fail the build | Contracts use `mvn test -Dtest=<Class>`; verify discovery in Step 5; hand the runner an explicit suite file; raise with the test repository owner | Open |
| R8 | A failed exit check stops `SecondSecurityReview`, `Review`, and `Verification` | A weak test plan blocks a good code change | Accepted by F6; an advisory alternative is in [`suggestionbox.md`](suggestionbox.md) | Accepted |
| R9 | A test with an `UNRESOLVED:` locator or a `DATA:` gap fails by design, and F17 would mark it a genuine failure and a blocker | False blockers and failed Xray results | Add an `unresolved` failure class in Step 7 (Q21) that covers both markers | Open |
| R10 | The interim test repository snapshot (F21) goes stale | Generated code compiles against old page objects | Ask the orchestration owner who refreshes it; replace it in Step 10 | Open |
| R11 | The three test contracts sit in three places (`SDLC-Orchestration/`, the generator pack, the exit assessor pack), and `confidence-contract.md` in two (the assessor packs) | The agent, the assessor, and the stage disagree | Keep each set byte-identical and compare hashes before every contract change; drop the stage copy at migration. Step 4 changed no contract. The stage cannot compare the copies at run time, because the packs are not in the Kestra container, so the comparison stays manual | Mitigated |
| R12 | `Mode: extend` edits existing test files that other tickets also change | Merge conflicts at write-back | Keep extensions small; decide the branch and merge strategy in Step 5 | Open |
| R13 | The Step 3 packs have not been run against any ticket, by decision | Ambiguities or rubric mistakes in the instructions go unseen until the stage runs live | Static checks only (contract hashes, manifests, rubric totals); review the first live assessments against the contracts when the stage runs | Accepted |
| R14 | The Step 4 stage flow is not compiled against the stage graph or run, by decision. `validate.jq` is proven only on two hand-written samples and seeded violations, with jq 1.7.1, and the Kestra image installs jq from apt without a pin | A YAML or expression error stops the stage from loading. A rule stricter than the agent's real output fails valid runs with no retry. A different jq version could change results | The flow reuses the Design stage's constructs. At the first live run, read the `check-result:` log lines against the contracts, and run `test.sh` in the Kestra image after `jq --version`. Repeat the Step 1 compile check when a current copy of the orchestration project arrives | Accepted |
| R15 | `validate.jq` hand-encodes the three test contracts, and nothing detects a contract edit | A contract change leaves the stage enforcing the old rule, or rejecting output the contract now allows | Re-read `validate.jq` and run `test.sh` after every contract change. A pinned hash of each contract in `test.sh` would flag a change (see [`suggestionbox.md`](suggestionbox.md)) | Open |
| A1 | The Jira bug template requires reproduction steps, expected result, and actual result | Defect tests depend on it | Recorded in [`suggestionbox.md`](suggestionbox.md) | Assumed; METERSENSE-11680 is evidence against it |

### To verify against the Xray tenant

| # | Behavior | Needed by | Status |
|---|---|---|---|
| V1 | The separator Xray uses between `classname` and `name` for JUnit Definitions | Step 6 | ⬜ |
| V2 | Whether setting the project default test type changes import auto-provisioning from Generic | Step 6 | ⬜ |
| V3 | Whether `requirements` and `tags` properties apply on update as well as creation | Step 6 | ⬜ |
| V4 | The `testcase` name that Surefire 3.2.5 writes for each `@DataSource` row | Step 6 | ⬜ |

## Decision log

### Open

| # | Question | Decide before | Owner | Status |
|---|---|---|---|---|
| Q6 | Who generates test data, and from what? Interim rule for Step 3: the pack reuses existing data files and arrays, writes no new data or fixtures, and names each need as a `DATA:` gap | Step 7 (data failure class); a data-generator project owns it | TBD | Deferred |
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
| Q5 | Test identity | Fully qualified method is the Definition (F10, F11); manual tests use `manual.<Feature>.<behavior>` (F18) |
| Q9 | Confidence thresholds and timing | Threshold 50; entry and exit checks (F6) |
| Q10 | Product scope | Metersense only (F5) |
| Q12 | Execution ticket scope | One Test Plan and two Test Executions per run (F8) |
| Q14 | Modules and Xray folders | Jira Component field (F5) |
| Q15 | Tests below input level L3 | Generated with gaps flagged and confidence lowered (F15) |
| Q16 | Element IDs for new UI | From the Design implementation plan (F1, F15) |
| Q17 | Testability in upstream confidence | Judged at stage 6b entry (F14) |
| Q18 | Jira bug template | Assumed (A1) |
| Q19 | Entry and exit checks in one stage | Allowed (F6) |
| Q22 | What "coverage ratio" means | Tests ÷ covered requirements (F19), reported but not scored. The exit rubric scores Coverage %, test design, plan and code consistency, input level, and metrics accuracy |
| Q23 | Does a Coverage summary mismatch fail the stage? | No. `check-result.sh` logs it as a warning, and the exit assessor scores it (Metrics accuracy, 5 points). A failure there would block the run before the assessor sees it and make that rubric row unreachable. Every other contract violation fails the stage |

## Deliverables

### By repository

New files go in a top-level folder named for their target repository.

| Folder | Contents |
|---|---|
| `SDLC-Orchestration/` | `main_test-case-generation.yml`, `check-result.sh`, `validate.jq`, `test.sh`, and `fixtures/` (in `test-case-generation/`); `main_verification.yml`, scripts, Xray client, `xray-integration.md` |
| `AIAgents/` | `test-case-generator` pack, the entry and exit assessor packs (`test-case-generator-confidence`, `test-case-generator-coverage-confidence`), and any reuse or retention-review agent |
| `Test Repository/` | Instruction files and templates that shape generated code |
| `SDLC-Artifacts/` | Run output folders (`Artifacts/<ticket>/<YYYYMMDD_HHMMSS>/<stage>/`) once the stages run, including generated code and a sample JUnit XML |

### Run artifacts

Each run's artifacts go in
`SDLC-Artifacts/Artifacts/<ticket>/<YYYYMMDD_HHMMSS>/<stage folder>/`, the layout
of `References/SDLC-Artifacts/`. Stage 6b's folder is `TestCaseGeneration/`;
Verification's is `Verification/`. The execution folder also holds a
`manifest.json`, which the orchestration writes. An agent writes only its own
files there (see Restrictions).

| Artifact | Purpose |
|---|---|
| `confidence-inputs.md` | Entry rating of whether upstream inputs are good enough to test from |
| `testplan.md` | Reviewable test cases: metrics, BDD scenarios, automated/manual breakdown |
| `testcode.md` | Generated Playwright tests in house style |
| `confidence-coverage.md` | Exit rating from Coverage % and coverage ratio |
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
| `TEST-REPO-READ` | `References/Test Repository/`, read as the interim `testRepository` dependency (F21) | Read-only checkout through a second `_config/projects.yml` entry and a branch resource with `repo:` | Placeholder |
| `TEST-REPO-WRITE` | `Test Repository/README.md` | `git.branch` and `git.commit-and-push` to the test repository | Placeholder |
| `SOURCE-REPO-READ` | `References/metersense-release-5.1/` | Read-only checkout of `baseBranch` as `source` (`git.clone-source`) | Placeholder |
| `ARTIFACT-SPACE` | `SDLC-Artifacts/Artifacts/<ticket>/<YYYYMMDD_HHMMSS>/TestCaseGeneration/`, the layout of `References/SDLC-Artifacts/`; each execution folder holds a `manifest.json` | `artifacts.publish-workspace` into SDLC-Artifacts | Placeholder |
| `TICKET-INPUT` | `References/SDLC-Artifacts/Artifacts/<ticket>/<execution>/Jira/input.json` (for example METERSENSE-11680 and IDEA-30) | `Jira/input.json` written by `execution.initialize` | Placeholder |
| `XRAY-API` | `SDLC-Artifacts/README.md` | Direct Xray API calls from Kestra | Placeholder |
| `JUNIT-RESULTS` | `SDLC-Artifacts/README.md` | `target/surefire-reports/TEST-*.xml` from a real run | Placeholder |
| `VERIFICATION-RUNNER` | `SDLC-Orchestration/step-1-stage-contracts.md` | Runner container from a later project | Handed off, not retired |
| `AGENT-PATH` | Flat pack folders in `AIAgents/` | The stage requests `AIDLC/TestCaseGeneration/<agent>` | Placeholder |

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
5. An agent writes only into its stage folder under
   `SDLC-Artifacts/Artifacts/<ticket>/<YYYYMMDD_HHMMSS>/`, and only the files its
   pack names. It writes nowhere else, including scratch notes. In a container the
   stage publishes `/work/resources/output/` to that folder; in a draft run the
   output paths map to it directly.
6. Keep code, readmes, and documentation efficient, clear, and non-verbose.
7. Follow the Google developer documentation style guide.

## Resources

- [`use-cases.md`](use-cases.md): the reference use case and stage sequence.
- [`suggestionbox.md`](suggestionbox.md): cross-department change requests.
- [`SDLC-Orchestration/step-1-stage-contracts.md`](SDLC-Orchestration/step-1-stage-contracts.md): the stage contracts.
- [`SDLC-Orchestration/testplan-contract.md`](SDLC-Orchestration/testplan-contract.md),
  [`testcode-contract.md`](SDLC-Orchestration/testcode-contract.md), and
  [`testcode-naming-rules.md`](SDLC-Orchestration/testcode-naming-rules.md): the artifact contracts. Byte-identical
  copies are in `AIAgents/test-case-generator/instructions/` (R11).
- `References/`: current work from other departments. Read-only. `References/README.md` maps it.
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

Ticket comments belong to rank 1: a later comment that changes the description wins. The plan's
Traceability table records each requirement's source, so the ranking can be audited.

## Appendix B — Input detail levels (F15)

| Level | Inputs available | What can be produced |
|---|---|---|
| L1 | Acceptance criteria only | A Gherkin scenario; no runnable code |
| L2 | L1, plus screen, navigation, role, labels, controls, and messages | A scenario and code draft using role or label locators |
| L3 | L2, plus stable element IDs, exact data, and database setup rows | Complete code in house style |
