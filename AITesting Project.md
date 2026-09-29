# AI testing project

## Summary

Fill in the two QA stages of the AI-DLC pipeline — **Test Case Generation** (stage 6b) and
**Verification** (stage 8b) — so that an AI agent derives functional UI test coverage for a Jira
ticket, registers it in Xray, and reports results back to Jira.

Both stages already exist in SDLC-Orchestration as declared stubs. This project replaces the manual,
analyst-driven process that writes Xray test cases today with a Kestra-orchestrated process that
writes executable Playwright tests.

This is the **planning draft**. It is built against the local clones in `References/`, and every
external system is represented by a placeholder with a recorded migration path. Test Case Generation
is designed through to working code; Verification is scaffolded to the point where a future runner
container can attach.

## Scope

In scope:

- Functional, end-to-end **UI** test coverage only.
- A test case document written to the artifact space.
- Generated Playwright test code, in the style of the existing test repository.
- The Xray payload for test issues and a Test Execution ticket, designed for direct API call.
- JUnit XML results collected as an artifact and imported into Xray.
- A Verification stage that hands the generated suite to a placeholder runner.
- A placeholder registry and the migration step that retires it.
- `xray-integration.md`, the guide for turning the Xray integration on.

Out of scope, deferred to a named later project:

- Unit and integration tests. Code Generation (stage 6a) owns those.
- Backend and API test coverage. The only backend work here is inserting rows to support a UI test.
- The Verification runner container, the environment it runs against, and test data generation.
- Live connections to Bitbucket, Jira, Xray, or SDLC-Artifacts. See "Placeholders".
- Changes to any stage other than Test Case Generation and Verification. Requests for upstream or
  downstream change go into a suggestion box rather than into another champion's folder.

## Current state

### Orchestration

`References/SDLC-Orchestration/_flows/stages/` holds eleven stages. Two are relevant here, and both
are stubs (`stub: true`, `requires: []`, `provides: []`):

| Stage | Ordinal | Artifact dir | Predecessors | Successors |
|---|---|---|---|---|
| `TestCaseGeneration` | 6 | `TestCaseGeneration` | `FirstSecurityReview` | `SecondSecurityReview` |
| `Verification` | 8 | `Verification` | `SecondSecurityReview` | `Release` |

Facts that constrain the work:

- A stage declares its whole contract in `variables.stage` in its own `main_<stage>.yml`. The
  compiler checks that every `successors` edge is matched by a `predecessors` edge on the stage
  named, so contract changes are agreed at both ends. See `_flows/stages/README.md`.
- Every stage uses the same frame: `stage.initialize` → domain work → `stage.finalize`, with an
  `errors` block that captures a reason and calls `stage.fail`.
- `agents.run-container` sparse-checks out one agent folder from AIAgents, bind-mounts it at
  `/work/agent`, stages inputs under `/work/resources/`, and collects outputs from
  `/work/resources/output/`.
- `agents.run-container` can clone exactly one source repository, resolved from
  `_config/bitbucket-projects.json` by Jira project key. Today `METERSENSE` maps to `repoSlug:
  metersense` in project `MS`. There is no second-repository path.
- `git.clone-source` runs `git clone --branch <baseBranch> --depth 1` and creates a feature branch
  only when `newBranch` is non-empty. Design already calls it read-only. **A depth-1 clone of the
  base branch cannot see the branch Code Generation pushes**, so read-only base access enforces the
  no-bias rule by construction rather than by prompt wording.
- The `TestCaseGeneration` stub records why it is stubbed: it previously borrowed the
  `code-generator` agent with no source tree and that agent returned `SOURCE_NOT_FOUND`. It needs an
  agent of its own, and that agent's source requirement decides whether the stage can keep running
  in parallel with Code Generation.
- The `Verification` stub records an unsettled argument that the stage should trigger existing CI
  and consume the result rather than own a build environment for every SmartWorks product.
- The repository root `README.md` is stale. It describes five phases under `_flows/phases/` and an
  `enableTestGeneration` input that no longer exists. Treat `_flows/stages/` as authoritative.

### Manual process today

`References/Manual Process/` documents the process being replaced. Reduced to what carries forward:

1. A QA analyst triggers generation from the Jira issue panel.
2. The ticket is collected, along with its children when it is an Epic or Story.
3. An AI pass turns that into a requirements document.
4. A second AI pass turns the requirements into structured test cases.
5. The analyst reviews and edits them.
6. The approved cases are created as **Manual** Xray Test issues, one at a time, and linked to the
   requirement ticket.

Three properties of that process are worth keeping: the two-pass split between deriving requirements
and deriving test cases, the human review point, and sequential creation to stay inside Xray's rate
limits. The tooling that implemented it is being retired in full, so no part of its architecture
constrains this project.

The one gap it never closed: it creates **test definitions only** and has no path for importing
execution results. That gap is the substance of F11.

### Test repository

`References/Test Repository/` is the Metersense UI suite that generated tests must match:

- Java 21, Playwright for Java 1.57.0, TestNG 7.12.0, ExtentReports 5.1.2, Surefire 3.2.5.
- `src/main/java/` holds `pages/`, `models/`, `utils/`, `database/`.
- `src/test/java/` holds `base/`, `tests/`, `assertion/`, `data/`.
- Packages are **flat and undomained** — a test class declares `package tests;`, not a reverse-domain
  name. This keeps fully qualified names short but makes them weak as global identifiers. See F11.
- Tests are grouped `smoke` and `regression`, and composed into `testNg.xml`, `smoketest.xml`, and
  `regressiontest.xml`.
- JSON-driven data through `@DataSource` plus `DataProviderUtil`, with data in
  `src/test/resources/TestData/`.
- Assertions go through per-feature helper classes built on `assertion.AssertionHelper`.
- Runs as `mvn test -DsuiteXmlFile=<suite>.xml`, against an environment named in
  `src/main/resources/config.properties`.
- Results land in `test-output/CompassTestReport.html` and, by Maven convention,
  `target/surefire-reports/`.
- There is **no CI workflow** and **no Jira or Xray identifier** anywhere in the test code.

### Agent packs

`References/AIAgents/` holds four packs: `jira-reviewer`, `solution-designer`, `code-generator`, and
`ping-test`. A pack is a `manifest.json` plus `agents/<name>.agent.md`, and optionally
`instructions/`, `skills/`, and `mcp-config.json`. `manifest.json` must list every file in the pack,
and the schema rejects undeclared fields.

Every artifact-producing agent writes a file whose first heading is `# Status`, followed by a
`STATUS_CODE:MESSAGE` line and a `Confidence: <n>%` line. Orchestration parses that heading and
fails the stage when it is missing.

### Artifact space

Each execution writes to `Artifacts/<TICKET>/<YYYYMMDD_HHMMSS>/`. Each stage owns one directory and
is its only writer. A stage directory holds `stage-inputs.json`, `stage-record.json`, a
`confidence-entry.json` or `confidence-exit.json`, and the stage's own outputs. Confidence is
currently a stub assessor that always returns 100.

## Project facts

The settled conditions this project works within. Some are imposed by the surrounding systems,
others were decided and are now closed. Either way, the plan below treats them as given.

### Scope and method

#### F1 — The agent never sees the generated code

The purpose of the test is to confirm the **requirements**, not the implementation. A test written
against the implementation only proves the code does what it does. It cannot catch Code Generation
drifting from the requirements mid-run, because it would faithfully encode the drift.

The agent reads:

| Input | Why |
|---|---|
| Requirements, analysis, and design artifacts | The statement of what must be true |
| The existing test repository | House style, page objects, locators, assertion helpers, data shapes |
| The source repository at **`baseBranch`** | Where existing screens and elements live today |

It never reads the feature branch. `git.clone-source` enforces this once live: called with
`newBranch: ""`, the depth-1 base-branch clone cannot resolve the generated branch at all. Stage 6b
therefore stays parallel to stage 6a and needs no ordering change.

In the draft, `TEST-REPO-READ` and `SOURCE-REPO-READ` stand in for those clones, so the guarantee
rests on the agent instructions until step 10 makes it structural. State the rule in both places.

**Known limit.** UI elements that the ticket is introducing do not exist at `baseBranch`, so the
agent cannot discover their locators. It must take them from the design and implementation guides.
This makes "name the test identifiers for new UI elements" a suggestion box entry against the Design
stage.

#### F2 — UI only

Only UI fixes are tested. The single permitted backend action is inserting rows into tables to set
up a UI test, which the repository already supports through `src/main/java/database/` and
`src/test/java/database/`. A ticket with no UI surface is out of scope and must be reported as such
rather than guessed at.

#### F3 — This workspace is a draft, and every integration is a placeholder

This project is the preliminary draft and plan. It holds clones of each repository under
`References/` as they currently stand, and no live connection to any of them.

Every integration point is therefore written as a **placeholder** that resolves to a sibling folder
in this project today and becomes a real connection when the work moves into its target repository.
Placeholders are not deferred decisions — they are working stand-ins with a recorded migration path.
See "Placeholders" below.

This applies to the test repository, the source repository, the artifact space, Jira, and Xray
alike. The project evolves into those repositories rather than integrating with them from here.

#### F4 — Verification is a placeholder

A future project builds the container that runs the suite, provisions its environment, and generates
its data. This project ends at handing that runner a suite to execute and an Xray Test Execution
ticket to report into. Stage 8b carries the frame, the contract, and a clearly marked placeholder.

#### F5 — One product, many modules

Metersense is the product. It sits inside Compass, and it is divided into feature modules. The test
repository targets it, `bitbucket-projects.json` maps `METERSENSE`, and the source is cloned at
`References/metersense-release-5.1`. There is no second product to design for.

Modules are the axis that matters instead. They shape the Xray test repository folder structure,
which in turn drives reuse detection in step 8 and makes coverage reporting legible. The existing
page objects and test classes already group by feature — `EditMeter`, `ManageKPI`, `LocationSummary`
— so that grouping is the natural starting point rather than an invention. See Q14.

#### F6 — Confidence thresholds are 50, and configurable

Both stages keep `confidenceThreshold: 50` at entry and exit. The value belongs in the stage
contract, not in agent instructions or scripts, so raising the bar later is a one-line change that
the compiler and the stage record both see.

Stage 6b currently declares `gate: entry`. An exit gate is the meaningful one for test generation,
since the question worth asking is whether the produced coverage is trustworthy, not whether the
inputs looked adequate. Raise that with the stage owner in step 1 rather than assuming it.

### Xray

#### F7 — Xray Cloud, reached by direct API from Kestra

The tenant is **Xray Cloud**. The orchestration calls it directly: authenticate at
`/api/v2/authenticate` with a client ID and secret, manage test definitions through
`/api/v2/graphql`, submit results through `/api/v2/import/execution/junit`, and link to requirements
through Jira's REST API. Credentials are Kestra globals. No intermediate app or Atlassian-hosted
component is involved, and `agents.run-container` needs no MCP plumbing. This supersedes the custom
app currently in use, which is being retired.

`.github/instructions/xray-graphql.instructions.md` is the authoritative reference for every call.
Xray's Playwright tutorial documents Server and Data Center — different endpoints, different auth,
and a different multipart field name — so treat that page as conceptual only.

**Use the multipart endpoint when Test Execution fields matter.** The base endpoint takes query
parameters (`projectKey`, `testExecKey`, `testPlanKey`, `testEnvironments`, `revision`,
`fixVersion`) and creates an execution with a generic summary. The multipart variant takes `info`
and `testInfo` JSON parts and can set any Jira field, which is what makes the execution ticket
readable.

#### F8 — Generated tests are the Automated test type

The **Automated** test type exists in Xray, of kind **Unstructured**. Kind matters more than name:
the Definition field that result import matches on belongs to the Unstructured kind, so an
Unstructured type carries it whatever it is called. A type of kind Steps or Gherkin would not.

Only **one** Xray issue is created per generated test. There is no manual counterpart, no review
copy, and no second issue to keep in step.

**The constraint this creates.** JUnit import auto-provisions missing tests as **Generic**, and that
is not configurable — selecting a type at creation time is supported only by the Xray JSON format.
So the Automated type holds only because step 6 creates each test explicitly before any results
arrive. If a result fails to match, Xray silently creates a *Generic* duplicate, and the symptom is
a wrong-typed twin rather than an error. F11 removes that failure mode.

#### F9 — Tests link to requirements; superseded manual tests become labels

Xray's native coverage model is Test *covers* Requirement. Both a legacy manual test and its
replacement automated test point at the same Jira ticket, so the relationship between them is
derivable without a direct link between the two.

Tests are not linked one-to-one. That would create maintenance on records scheduled for deletion,
and it needs the same matching problem Q7 already covers. Where a generated test supersedes a known
manual test, that manual test's key is recorded as a **label** on the automated test — cheap,
searchable during the transition, and disposable once manual tests are gone, which a structural
issue link is not.

**No repository change is needed to link a test to its ticket.** Step 6 creates each Xray test
through the API, so it sets the *covers* link at creation time. The Java code never names a ticket,
and nothing in the test repository has to know Jira exists. The link lives on the Xray issue, the
binding from code to that issue is the `test_key` injection in F11, and the two meet in
orchestration rather than in source.

#### F10 — Results reach Xray as JUnit XML

The suite emits JUnit XML. That file is both an artifact, stored with the execution record, and the
payload imported into Xray so results land on the Jira ticket.

**The repository already produces it.** Maven Surefire writes JUnit-format XML to
`target/surefire-reports/TEST-*.xml` for TestNG runs with no configuration change. The Xray
Playwright tutorial's `npx ... --reporter=junit` does **not** apply here — that is Playwright Test
for Node, and this repository is Playwright for **Java** driven by TestNG. Nothing needs adding to
produce the file; what needs designing is identity, transport, and the import call.

**Identity.** Surefire writes the fully qualified class into `classname` and the method into `name`.
The repository uses flat packages (`tests`, `pages`, `models`), so the key is short:

```
tests.EditMeterTest.testEditMeterInsertRecord
```

That key becomes the test's Definition field when step 6 creates it. How results bind to that test
at import time is settled separately in F11, because Definition matching alone is not reliable
enough to depend on — and the flat package naming is part of why.

**Known limit.** `@DataSource` data-driven methods emit one `<testcase>` per row, all sharing a name.
They collapse onto a single Xray Test unless the suite is configured otherwise. Confirm the intended
behaviour before relying on per-row results. See Q11.

**Upgrade path, not required now.** `app.getxray:xray-testng-extensions` (Maven Central) adds
`@XrayTest(key = "...")` and `@Requirement(...)`, binding a method to an Xray test from within the
code. F11 achieves the same binding without touching the test repository, so this is only worth
revisiting if the tests ever need to run outside this pipeline and still report correctly.

#### F11 — Results bind by explicit key, not by string matching

Xray resolves a JUnit `<testcase>` to a test in three ways, in order: a `test_id` property on the
element, a `test_key` property, or Definition matching against `classname` + `name`.

Definition matching alone is fragile here, for three reasons:

- The separator Xray inserts between `classname` and `name` is documented for NUnit but **not** for
  JUnit.
- A mismatch creates a duplicate rather than raising an error.
- Matching searches the target project first and then **every other Jira project**. The repository
  uses flat packages, so a Definition like `tests.LoginTest.testValidLogin` carries nothing
  product-specific and could match another team's test in an unrelated project.

**`test_key` is the binding.** Step 6 creates the tests, so the orchestration already holds the
mapping from fully qualified method to Xray key. Step 7 injects that mapping into the JUnit XML as a
`test_key` property per `testcase` before importing:

```xml
<testcase classname="tests.EditMeterTest" name="testEditMeterInsertRecord">
  <properties>
    <property name="test_key" value="MSTEST-456"/>
  </properties>
</testcase>
```

An unresolvable `test_key` raises an error, so the pipeline fails loudly instead of quietly
polluting Xray. The injection happens in Kestra, so the test repository needs no annotation, no
dependency, and no knowledge of Jira — which keeps F3 intact.

The Definition field is still set to the fully qualified method when each test is created. It costs
nothing, documents the binding for anyone reading the issue, and leaves a fallback if injection is
ever skipped.

## Target outcome

1. `TestCaseGeneration` runs a new agent pack that produces a test case document and Playwright test
   code, derived from requirements and existing tests, with no sight of the generated code.
2. Generated tests have a complete Xray payload — test issues linked to their ticket, and a Test
   Execution ticket for a future runner — ready to send once `XRAY-API` goes live.
3. `Verification` carries a working stage frame with a marked placeholder for the runner, and a
   results path that turns JUnit XML into an Xray import.
4. Neither stage requires a change to another stage's folder. Anything that does is proposed in
   `suggestionbox.md`.
5. The pipeline reuses an existing test instead of writing a new one where a suitable test exists.
6. Each generated test carries a retention decision, so single-use tests can be retired at quarter
   end and durable tests enter quarterly regression.
7. Every placeholder has a recorded migration path, so moving into the real repositories is
   mechanical rather than exploratory.

## Open questions

| # | Question | Why it matters | Decide before |
|---|---|---|---|
| Q6 | Does the agent generate JSON test data files now? | Data generation is deferred by F2, but `src/test/resources/TestData/` JSON files are part of the test code and `@DataSource` will not run without them. Distinguish authored fixtures from generated volume data. | Step 3 |
| Q7 | What counts as a matching existing test, and what is queried? | Jira returns summary and description but not steps; Xray returns steps but caps retrieval at 100 IDs per request and rate-limits. Needs an agreed match rule and an index. | Step 8 |
| Q8 | Who makes the retention decision, and what enforces it? | A `temp` marker needs a home in the test code, a home in Xray, and a quarterly job that acts on it. | Step 9 |
| Q11 | How are data-driven results reported? | A `@DataSource` method emits one JUnit `<testcase>` per row under one name, so rows collapse onto a single Xray Test. Options are to accept the collapse, split rows into distinct methods, or move to the Xray JSON format, which supports per-iteration results. | Step 6 |
| Q13 | Which Test Execution fields must be set, and which are configurable? | `info` accepts any Jira field. Fix version, components, test environment, and a summary naming the ticket and run are the usual minimum for readable history. | Step 6 |
| Q14 | Which Metersense modules does the suite cover, and how does that map to Xray folders? | Per F5, Metersense is one product with distinct feature modules. Folder structure drives both reuse detection in step 8 and readable coverage reporting. | Step 6 |

### Settled

| Was | Settled by |
|---|---|
| Q1 — how the agent reads existing tests | F3. A placeholder resolving to the local clone. |
| Q2 — manual-to-automated linking | F9. Link to the requirement; record the superseded manual key as a label. |
| Q3 — Xray test type | F8. A custom **Automated** type of kind Unstructured. |
| Q4 — one Xray issue or two | F8. One only. No manual counterpart. |
| Q5 — test identity | F10 and F11. The fully qualified method is the key; `test_key` injection is the binding. |
| Q9 — confidence thresholds | F6. Entry and exit at 50, set in the stage contract so they stay changeable. |
| Q10 — product scope | F5. One product, Metersense, with feature modules inside Compass. |
| Q12 — execution ticket scope | One per pipeline run. Step 6 creates it and passes the key to step 7, which imports with `testExecKey`. |

### To verify empirically

Two behaviours are not documented by Xray and must be checked against the tenant rather than
assumed. Both are cheap to test and expensive to get wrong:

1. **The JUnit Definition separator.** Documented as `.` for NUnit, described only as "concatenated"
   for JUnit. F11 removes the dependency on it, but confirm before relying on Definition as a
   fallback.
2. **Whether a custom default test type changes auto-provisioning.** Every Xray page says imports
   auto-provision as *Generic*, unconditionally. If setting the project default to Automated also
   changes auto-provisioning, the F8 failure mode disappears; if not, F11 remains the only
   protection.

## Plan

Each step names a deliverable and the condition that closes it. Steps 1 to 7 are sequential. Steps 8
and 9 can start once step 5 is done. Step 10 runs last and retires the placeholders.

### Step 1 — Settle the stage contracts

Write the `variables.stage` blocks for both stages, including the `requires` and `provides` names,
and confirm the predecessor and successor edges still agree at both ends. Stage 6b requires the
design artifacts, `repoSlug`, `baseBranch`, and `bitbucketProject`, and must **not** require
`featureBranch` — that omission is what keeps F1 true. Set both thresholds to 50 per F6, and raise
the entry-versus-exit gate question there.

Declare the placeholder registry at the same time, since the contracts name what each placeholder
stands in for.

Deliverable: a contract proposal covering both stages, the placeholder registry, and
`suggestionbox.md`. Seed the suggestion box with the Design-stage request from F1: design and
implementation guides should name the test identifiers for UI elements the ticket introduces.
Exit: the proposed edges compile against the existing graph, every placeholder resolves to a real
path in this project, and the owners of any affected stage have seen the suggestion box.

### Step 2 — Define the artifact and test contracts

Specify the test case document that stage 6b writes, following the `# Status` /
`STATUS_CODE:MESSAGE` / `Confidence` convention that orchestration parses. Record the `NO_UI_SURFACE`
outcome that F2 requires for a ticket with nothing to drive.

Test identity follows F10: the fully qualified TestNG method is the key. Fix the package and class
naming rules that make it stable, since renaming a class changes the Definition and, per F11, the
step 6 mapping has to be regenerated with it. Decide here whether generated tests keep the flat
`tests` package or move to a namespaced one — F11 notes the collision risk the flat form carries.

Deliverable: `testplan-contract.md`, `testcode-contract.md`, and the naming rules.
Exit: a sample document can be hand-written against the contract and parsed by the stage.

### Step 3 — Build the test-case-generation agent pack

Resolve Q6. Create a `test-case-generator` pack in the AIAgents layout: `manifest.json`, `agents/`,
`instructions/`, and the contracts from step 2. Derive test design style from
`References/Manual Process/Test Generator Agents/Plan/` and code style from
`References/Test Repository/.github/copilot-instructions.md`.

The instructions must state the F1 rule explicitly — author from requirements, use existing tests as
the pattern, treat the base-branch source as the map of what already exists — so that the constraint
survives independently of how the stage happens to call the container.

Deliverable: the agent pack, and a suggestion box entry recording improvements over the manual
process it replaces.
Exit: the pack runs against a sample ticket and produces a document and test code that satisfy the
contracts.

### Step 4 — Wire the Test Case Generation stage

Replace the stub's `domain-work` with a read-only `agents.run-container` call (`cloneSource: true`,
`newBranch: ""`) plus `artifacts.persist-agent-output`. Set `stub: false` and declare `provides`.
The confidence assessor stays a stub for now; F6 fixes the thresholds it will be judged against.

Deliverable: a working `main_test-case-generation.yml`.
Exit: a run produces `Artifacts/<TICKET>/<EXEC>/TestCaseGeneration/` with the test case document,
`stage-record.json`, and a passing gate — and an inspection of the container mounts confirms the
feature branch is absent.

### Step 5 — Emit test code to the artifact space

Generate Playwright test classes, page objects, data JSON, and suite XML entries. Write them through
`ARTIFACT-SPACE` and mark the push path `TEST-REPO-WRITE`. Record what the real integration needs: a
second repository mapping, credentials, a branch strategy, and a merge owner.

Deliverable: generated test code under the stage's artifact directory, plus the `TEST-REPO-WRITE`
handover.
Exit: a reviewer can copy the generated files into the test repository, and they compile and are
discoverable through the suite XML.

### Step 6 — Register tests in Xray by direct API

Resolve Q11, Q13, and Q14. Design the Kestra-side Xray client:

1. Authenticate, and cache the token for under an hour.
2. Create each test as type **Automated** (F8), with its Definition set to the fully qualified
   TestNG method.
3. Place it in the folder for its module, link it to the ticket as a covered requirement, and label
   any manual test it supersedes (F9).
4. Create the Test Execution ticket through the multipart `info` part, and keep its key for step 7.
5. Record the mapping from fully qualified method to Xray key — step 7 depends on it.

Keep creation sequential to stay inside Xray's rate limits. Behind `XRAY-API`, write each payload
beside the artifacts rather than sending it.

Confirm the Automated type's ID or name as Xray's API expects it, since the client passes it on
every create. The type itself already exists (F8).

Deliverable: the Xray client, its payload contract, the method-to-key mapping, and
**`xray-integration.md`** — the guide requested for when this goes live. It must cover: which
endpoints and why, credentials and where they live, how a test binds to its Xray issue, which `info`
and `testInfo` fields are set and which are free to customise, both failure modes from F8 and F11,
and how to verify an import by hand with `curl` before trusting the pipeline.
Exit: the payload for a generated test is complete enough that sending it would create a correctly
typed and linked Xray test plus an execution ticket, and the guide is specific enough to follow
without rereading this document.

### Step 7 — Scaffold Verification and the results path

Per F4, give stage 8b the stage frame, the contract, and an explicit placeholder where the runner
container will attach. The runner interface is concrete on both sides, so write it as a contract:

| Direction | Item |
|---|---|
| Into the runner | Suite XML to run, target environment, Test Execution key from step 6 |
| Out of the runner | JUnit XML at `target/surefire-reports/TEST-*.xml`, exit code, run duration |

Then build the results path, which is in scope even though execution is not:

1. Collect the JUnit XML and store it in the artifact space.
2. Inject a `test_key` property into each `<testcase>` from the step 6 mapping (F11).
3. Import to `/api/v2/import/execution/junit?testExecKey=<key>`.
4. Treat an unresolved `test_key` as a stage failure, not a warning.

Deliverable: `main_verification.yml` with a marked placeholder, the runner interface, and the
collect-inject-import path.
Exit: given a JUnit XML file placed by hand where the runner would write it, the stage stores it,
injects the correct keys, and produces an import request that would land results on the right tests
in the right execution ticket.

### Step 8 — Detect reusable tests

Resolve Q7. Build an index of existing coverage and add a reuse check to stage 6b that runs before
anything is authored. Respect the 100-ID retrieval cap and the sequential import rule.

Deliverable: an index plus a reuse check in the agent instructions.
Exit: a ticket already covered produces a reuse recommendation rather than a new test.

### Step 9 — Add the retention decision

Resolve Q8. Add a reviewing agent or instruction that marks each generated test as durable or
single-use, carry the marker into both the test code and Xray, and define the quarterly job that
retires single-use tests. A TestNG group is the natural home in the code, since the suite already
selects on `smoke` and `regression`.

Deliverable: a retention marker convention and the review step that applies it.
Exit: a bug-ticket test is marked single-use and excluded from the regression suite; an epic test is
marked durable and included.

### Step 10 — Migrate into the real repositories

Retire the placeholders. Move each top-level folder into its target repository, replace each
placeholder with its real integration, and supply credentials as Kestra globals. Work through the
registry in order of dependency: `SOURCE-REPO-READ` and `TEST-REPO-READ` first, because they only
need wiring; then `ARTIFACT-SPACE` and `TICKET-INPUT`; then `XRAY-API`; and `TEST-REPO-WRITE` last,
since it is the only one needing a branch and merge policy.

`VERIFICATION-RUNNER` is not retired here — it is handed to the runner project with its interface.

Deliverable: the stages running in SDLC-Orchestration against live repositories.
Exit: no `PLACEHOLDER:` marker remains except `VERIFICATION-RUNNER`, and a full pipeline run
produces test artifacts and Xray records end to end.

## Deliverables by repository

New files go in a top-level folder named for their target repository:

| Folder | Contents |
|---|---|
| `SDLC-Orchestration/` | `main_test-case-generation.yml`, `main_verification.yml`, their scripts, the Xray client, and `xray-integration.md` |
| `AIAgents/` | The `test-case-generator` pack, and any reuse or retention-review agent |
| `Test Repository/` | Instruction files and templates that shape generated test code |
| `SDLC-Artifacts/` | Sample artifact outputs for both stages, including generated test code and a sample JUnit XML result |
| `suggestionbox.md` | Proposed changes to upstream outputs and downstream inputs, and process improvements over the manual process |

## Placeholders

A placeholder marks a point where this draft stands in for a live integration. Write each one as a
comment or a resolvable variable carrying the same four facts, so that migration is a search rather
than an audit:

```
PLACEHOLDER: <NAME>
  resolves-to: <path in this project today>
  becomes:     <the real integration>
  on-migration: <what has to change>
```

### Registry

| Name | Resolves to today | Becomes | On migration |
|---|---|---|---|
| `TEST-REPO-READ` | `References/Test Repository/` | Read-only `git.clone-source` of the test repository | Add the repo to `_config`, supply credentials, call with `newBranch: ""` |
| `TEST-REPO-WRITE` | Top-level `Test Repository/` folder | Branch, commit, and push to the test repository | Decide branch strategy and merge owner; mirror `commit-and-push.sh` |
| `SOURCE-REPO-READ` | `References/metersense-release-5.1/` | Read-only clone of the source repo at `baseBranch` | Already supported by `git.clone-source`; needs only the stage wiring |
| `ARTIFACT-SPACE` | Top-level `SDLC-Artifacts/` folder | `artifacts.publish-workspace` into SDLC-Artifacts | Replace direct writes with the workspace lifecycle subflows |
| `TICKET-INPUT` | `Example Tickets/METERSENSE-10101.md` | `Jira/input.json` staged by the orchestrator | Read the staged file instead of the sample |
| `XRAY-API` | A JSON payload written beside the artifacts | Direct Xray API calls from Kestra (F7) | Supply credentials as globals; send instead of writing |
| `JUNIT-RESULTS` | A sample JUnit XML committed with the artifacts | `target/surefire-reports/TEST-*.xml` from a real run | Point collection at the runner's output directory |
| `VERIFICATION-RUNNER` | Nothing — records the handover only | The runner container from a later project (F4) | Implement the documented runner interface |

Every placeholder now resolves to real content. `SOURCE-REPO-READ` points at
`References/metersense-release-5.1`, a full clone of the 5.1 release, so F1's third input can be
exercised in the draft rather than deferred.

### Migration

Placeholders are retired in step 10, not opportunistically. Until then, a placeholder that starts
calling a real service is a defect — it makes the draft non-reproducible for anyone else holding the
same clones.

## Resources

- `use-cases.md` — the reference use case, including the stage sequence and the not-finalised
  register.
- `References/` — current work from other departments. Read-only.
- `References/metersense-release-5.1/` — the Metersense 5.1 source clone, behind `SOURCE-REPO-READ`.
- `Example Tickets/METERSENSE-10101.md` — a sample ticket.
- `.github/instructions/xray-graphql.instructions.md` — Xray Cloud authentication, GraphQL reads,
  and the result-import endpoints. The authoritative reference for every Xray call in this project.
- [Import Execution Results REST v2](https://docs.getxray.app/space/XRAYCLOUD/44577176/Import+Execution+Results+-+REST+v2)
  — Cloud endpoints and their parameters.
- [Taking advantage of JUnit XML reports](https://docs.getxray.app/space/XRAYCLOUD/44564745/Taking+advantage+of+JUnit+XML+reports)
  — how a `testcase` binds to a test, including the `test_id` and `test_key` properties F11 uses.
- [Global Settings: Test Types](https://docs.getxray.app/space/XRAYCLOUD/44566485/Global+Settings%3A+Test+Types)
  — creating the **Automated** type and the three kinds a type can take.
- [Testing web applications using Playwright](https://docs.getxray.app/space/XRAY/301676766/Testing+web+applications+using+Playwright)
  — Xray's tutorial. Documents Server and DC with the Node runner, so the concept carries over but
  no endpoint, command, or auth detail does. Prefer the instructions file.
- [xray-testng-extensions](https://github.com/Xray-App/xray-testng-extensions) — `@XrayTest` and
  `@Requirement`. Not used; F11 achieves the same binding without a repository change.

## Restrictions

1. Use Kestra for orchestration.
2. Keep artifacts, orchestration, and test automation code in separate repositories.
3. Do not alter any file under `References/`.
4. Create new files in the top-level folder, separated by repository name.
5. Keep code, readmes, and documentation efficient, clear, and non-verbose.
6. Follow the Google developer documentation style guide.