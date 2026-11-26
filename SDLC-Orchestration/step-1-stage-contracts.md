# Step 1: Stage contract proposal

This proposal declares the Test Case Generation (6b) and Verification (8b)
interfaces in the current stage schema (`resources`, `deliverables`,
`nextStages`, `confidence`). The reference orchestration project is unchanged.
Step 4 applied the Test Case Generation block in
[`test-case-generation/main_test-case-generation.yml`](./test-case-generation/main_test-case-generation.yml),
and Step 7 applies the Verification block.

The blocks compile against the reference stage graph. The check ran
`build-stage-graph.sh` on a scratch copy of the reference stages with both
blocks applied, starting at `Analysis` (bug tickets), `Requirements` (default),
and `TestCaseGeneration`. All three runs passed. Three negative controls failed
with the expected errors: the first draft's legacy keys, a branch resource in a
repository with no `projects.yml` entry, and a required value that no stage
delivers.

## Changes from the first draft

| First draft | Current schema | Reason |
|---|---|---|
| `requires`, `provides`, `successors` | `resources`, `deliverables`, `nextStages` | The compiler reads only the current keys. With the legacy keys, `Verification` reports that no stage delivers `testCode`. |
| `predecessors` | Removed | The graph derives predecessors from other stages' `nextStages`. |
| `ordinal`, `gate`, `confidenceThreshold` | Removed; see [Confidence](#confidence) | The schema has no gate field. The `confidence` block holds the entry check, and a second `agents.confidence-check` call is the exit check. |
| `repoSlug`, `bitbucketProject` | Removed | The repository comes from `_config/projects.yml`. |
| `baseBranch` (bare name) | `baseBranch: { kind: branch, as: source }` | Every resource declares its kind. Design uses the same declaration. |
| "Keep the helper that records 100" | `confidence.agent` names the Step 3 assessor | The current helper runs a real assessor. An empty agent skips the check, so Step 3 names both assessors. |

## Test case generation

```yaml
stage:
  name: TestCaseGeneration
  artifactDir: TestCaseGeneration
  stub: false
  # PLACEHOLDER: AGENT-PATH (interim); the packs are flat in AIAgents/ today
  agent: test-case-generator                             # the generator

  nextStages:
    - SecondSecurityReview

  resources:
    - ticket: { kind: artifact, as: input.json }
    - ticketAttachments: { kind: artifact, as: attachments.tar }
    - design: { kind: artifact, as: design.md }
    - implementation: { kind: artifact, as: implementation.md }
    - baseBranch: { kind: branch, as: source }
    # PLACEHOLDER: TEST-REPO-READ (interim); becomes a branch resource in Step 10
    - testRepository: { kind: dependency, path: test-repository, as: test-repository }

  deliverables:
    - testPlan: { kind: artifact, file: testplan.md }
    - testCode: { kind: artifact, file: testcode.md }
    - xrayPayload: { kind: artifact, file: xray-payload.json }   # written from Step 6

  confidence:
    agent: test-case-generator-confidence                  # entry check
    threshold: 50
    exit:                     # read only by this stage's second confidence task
      agent: test-case-generator-coverage-confidence
      threshold: 50
      point: coverage
```

### Inputs

- **No feature branch (F1).** `featureBranch` is absent, so Code Generation
  stays parallel and the agent cannot see its output. `agents.run-container`
  mounts `baseBranch` read-only when the stage sets `readOnly: true`.
- **Jira ticket, attachments, and upstream artifacts.** `analysis.md` is not a
  stage resource, so the stage never stages it for either agent, even when
  Analysis wrote it. The packs and F12 still rank it second for every work kind;
  see the Step 4 working note in the project plan. Add an optional `requirements`
  artifact when the Requirements stage produces one.
- **Test repository.** `baseBranch` is the Compass product repository, not the
  Playwright repository, so the agent needs a second source for page objects,
  conventions, and `.github/copilot-instructions.md`. A branch resource in a
  second repository fails the compile until `projects.yml` has an entry for it.
  The interim `dependency` resource reads a snapshot from the orchestration
  dependencies folder (`DEPENDENCIES_DIR`, default
  `/var/kestra-wd/dependencies/test-repository`) and avoids that entry. The
  stage passes it to the agent through `filesIn` (`"{{ inputs.resources }}"`
  includes it). A missing dependency is named in `missing` for the entry check.
- **Container mounts.** With the default `readsFrom: inputs/artifacts`:

  | Path in the container | Content |
  |---|---|
  | `/work/resources/inputs/artifacts/input.json` | Ticket snapshot |
  | `/work/resources/inputs/artifacts/attachments/` | Attachments the fetch kept (text and PDF only) |
  | `/work/resources/inputs/artifacts/{design,implementation}.md` | Upstream artifacts |
  | `/work/resources/inputs/artifacts/test-repository/` | Test repository snapshot (interim) |
  | `/work/resources/inputs/source/` | `baseBranch`, read-only |
  | `/work/resources/output/` | `testplan.md`, `testcode.md` |

### Deliverables

The three declared artifacts are the ones other stages can require.
`confidence-inputs.md` and `confidence-coverage.md` are written to the stage
folder by `agents.confidence-check` and published with it, so they are not
declared. Step 6 adds a value deliverable, `automatedExecutionKey`, because
`Verification` needs the key to import results. Declaring it earlier fails the
compile when `Verification` requires it, and an unset value would block the
run at the Verification entry check.

### Confidence

- **Entry.** The first task of the stage reads `confidence.agent`. It checks
  the ticket, `design.md`, and `implementation.md` against the
  source (F6, F14).
- **Exit.** A second `agents.confidence-check` call runs after domain work, with
  `point: coverage` and `filesIn` set to the stage resources plus the two
  outputs. The assessor needs the resources to judge whether the plan misses
  a requirement. The `exit` key is a local extension: the compiler ignores it
  and only this flow reads it. Ask the orchestration owner to accept it (see
  [`suggestionbox.md`](../suggestionbox.md)).
- **Assessor output.** `confidence.sh` takes the first `SUCCESS:` or `FAILED:`
  line and the first `Confidence:` line anywhere in `confidence-<point>.md`.
  Both Step 3 assessors write both lines, as their `confidence-contract.md`
  defines. A `FAILED:` line blocks the stage; a low rating is a `SUCCESS:`
  assessment of poor material.
- **Skipping.** `skipConfidenceCheck=First` skips both checks when
  `TestCaseGeneration` is the starting stage, because the exit call uses the
  same bypass rule.
- **Blocking.** A failed exit check fails the stage after the plan and code are
  generated. The files are still published, and `SecondSecurityReview`,
  `Review`, and `Verification` do not run. This follows the use-case catalogue,
  where a phase below its threshold fails the execution.

### Assessors

Both assessors are packs in `AIAgents/`. Their instructions never mention the
threshold, and each scores from a rubric so that the rating can be audited.

| | Entry | Exit |
|---|---|---|
| Pack | `test-case-generator-confidence` | `test-case-generator-coverage-confidence` |
| `point` | `inputs` (the default) | `coverage` |
| Prompt | `Perform TestCaseGeneration confidence assessment.` | `Perform TestCaseGeneration coverage confidence assessment.` |
| `filesIn` | Empty: the context's `assessmentInputs` (ticket, artifact resources, `source`) | `"{{ inputs.resources }}"` plus `<workspace>/output/testplan.md` and `testcode.md` |
| Reads, under `/work/resources/inputs/` | `artifacts/input.json`, `design.md`, `implementation.md`, and `source/` | The same, plus `artifacts/testplan.md`, `testcode.md`, and `test-repository/` |
| Writes | `confidence-inputs.md` | `confidence-coverage.md` |
| Rubric (points) | Requirements 35, Accuracy 25, UI mechanics 25, Consistency 15 | Coverage 40, Test design 20, Plan and code consistency 20, Input level 15, Metrics accuracy 5 |

- The check flow runs each assessor with the default `readsFrom`
  (`inputs/artifacts`) and `readOnly: true`, so the paths above hold.
- The prompt does not need to say where to write: the assessors know
  `/work/resources/output/confidence.md`. The check flow appends the names of
  declared resources that were not found, and each assessor reads them as
  `Missing`.
- The entry assessor does not receive the test repository, which a `dependency`
  reaches only through `filesIn`. The exit assessor does, through
  `"{{ inputs.resources }}"`; list `testRepository` explicitly if a run shows the
  folder missing.
- The exit rubric derives from recomputed metrics and an independent
  requirements list, not from a self-reported rating. The coverage ratio
  (tests ÷ covered requirements) is reported but not scored (Q22).

### Tasks

Step 4 wired these tasks in `main_test-case-generation.yml`:

| Task | What it does |
|---|---|
| `confidence` | Entry check. Writes `confidence-inputs.md`. |
| `run-test-case-generation-container` | `agents.run-container` with `agentName: "{{ vars.stage.agent }}"`, `readOnly: true`, and `filesIn` of `"{{ inputs.resources }}"`, which adds the test repository dependency that the default leaves out. `filesOut` is `testplan.md`, `testcode.md`, and the reasoning log (`test-case-generation_reasoning.txt`). |
| `check-result` | Runs `check-result.sh` (below). |
| `confidence-coverage` | Exit check, `point: coverage`. Runs only when `check-result` passes. |

The stage reports the outcome of `check-result`; the exit check blocks by
failing the flow, as the entry check does.

### Contract check

`check-result.sh` replaces the Design stage's script, which passes only
`SUCCESS`. It is bash, jq, and awk, because the Kestra image has no Python. The
parsing and rules are in `validate.jq`, which hand-encodes the three test
contracts.

- **Status.** It reads the first `# Status` envelope of both outputs.
  `SUCCESS` and `NO_UI_SURFACE` pass. Any other code fails the stage with the
  agent's message. The `Confidence` line must be well formed but is not scored.
- **Errors fail the stage.** Any rule under "Parsing and validation rules" in
  `testplan-contract.md` and "Validation rules" in `testcode-contract.md`, except
  the Coverage summary items below. The failure reason shows the first five
  findings; the task log has all of them.
- **Coverage summary mismatches warn.** A value that differs from its
  recomputed value (and an incomplete `Unresolved locators` list) is logged and
  does not fail the stage. The exit assessor scores them (Metrics accuracy), so
  failing here would hide that rubric row and block a run over a derived number.
- **Test repository checks.** `TEST_REPO` is the `testRepository` dependency.
  It decides whether a new file already exists, whether an `extend` file keeps
  every original line in order, and whether a class is registered in the suite
  files. Without the snapshot the script logs that and skips these checks. The
  agent's paths are validated before any lookup, so a path cannot leave the
  repository.
- **Order.** `testcode.md` is checked only after `testplan.md` is valid,
  because it is checked against the plan's identities and groups.
- **Not checked.** That a method sits in the class file that declares it (the
  check reads all generated Java together), that a concrete selector has not
  replaced a locator the plan marks `UNRESOLVED`, that code compiles, and that a
  `Scenario` reads as the requirement it traces. The exit assessor reads the
  first two.

The folder also holds `fixtures/` (the two Step 2 samples) and `test.sh`, which
runs the script against them and against seeded violations. Run it after any
change to `validate.jq`, `check-result.sh`, or a contract.

## Verification

```yaml
stage:
  name: Verification
  artifactDir: Verification
  stub: false             # Step 7 sets this

  nextStages:
    - Release

  resources:
    - testCode: { kind: artifact, as: testcode.md }
    - xrayPayload: { kind: artifact, as: xray-payload.json }
    # Step 6 adds: automatedExecutionKey: { kind: value }

  deliverables:
    - junitResults: { kind: artifact, file: junit-results.xml }
    - verificationReport: { kind: artifact, file: verification-report.md }

  confidence:
    agent: ""
    threshold: 50
```

- The runner stays a placeholder (F4). `Verification` hands it the suite and the
  Xray execution key and collects JUnit XML as the stage result.
- The use-case catalogue gates regression failures at the pull request instead
  of failing the stage. Step 7 reports failures in `verification-report.md` and
  keeps `statusCode` `SUCCESS` unless the stage itself breaks.
- The runner needs the solution under test. `Review` requires
  `featureBranch: { kind: branch }`; confirm with the runner owner whether
  `Verification` declares it the same way. It does not break F1, which applies
  only to Test Case Generation.
- `junit-results.xml` is one file, but F17 retries produce several. Step 7
  merges the files or archives them.

## Edge check

No graph edges change.

| Stage | Predecessors | Successors |
|---|---|---|
| `TestCaseGeneration` | `FirstSecurityReview` | `SecondSecurityReview` |
| `Verification` | `SecondSecurityReview` | `Release` |

`FirstSecurityReview` names `TestCaseGeneration`, and `SecondSecurityReview`
names `Verification`. `Release` is named by both `Verification` and `Review`.
`TestCaseGeneration` is an ancestor of `Verification` through
`SecondSecurityReview`, so `testCode` and `xrayPayload` resolve.

## Placeholder registry

Paths under `References/` are read-only. Generated output goes to the
matching top-level folder. A README is the deliberate stand-in where the live
output does not exist yet.

| Name | Resolves to today | Becomes | On migration |
|---|---|---|---|
| `TEST-REPO-READ` | `References/Test Repository/`; the stage reads it as the interim `testRepository` dependency | Read-only branch resource through a second `projects.yml` entry | Add the repository to `_config/projects.yml`, supply credentials, replace the dependency with a branch resource that sets `repo:` |
| `TEST-REPO-WRITE` | `Test Repository/README.md` | Branch, commit, and push to the test repository | Decide the branch strategy and merge owner; mirror `commit-and-push.sh` |
| `SOURCE-REPO-READ` | `References/metersense-release-5.1/` | Read-only clone of `baseBranch` as `source` | Use the existing `git.clone-source` integration |
| `ARTIFACT-SPACE` | `SDLC-Artifacts/Artifacts/<ticket>/<YYYYMMDD_HHMMSS>/<artifactDir>/`, the layout of `References/SDLC-Artifacts/`; a stage's `/work/resources/output/` publishes there | `artifacts.publish-workspace` into SDLC-Artifacts | Replace direct writes with the workspace lifecycle subflows |
| `TICKET-INPUT` | `Example Tickets/METERSENSE-10101.md` (empty today) | `Jira/input.json` staged by `execution.initialize` | Read the staged ticket instead of the sample |
| `XRAY-API` | `SDLC-Artifacts/README.md` | Direct Xray API calls from Kestra | Supply credentials as secrets and send the payload instead of writing it |
| `JUNIT-RESULTS` | `SDLC-Artifacts/README.md` | `target/surefire-reports/TEST-*.xml` from a real run | Point collection at the runner's output directory |
| `VERIFICATION-RUNNER` | This file's Verification section | The runner container from a later project | Implement the documented runner interface |
| `AGENT-PATH` | Flat pack folders in `AIAgents/`: `test-case-generator`, `test-case-generator-confidence`, `test-case-generator-coverage-confidence` | The stage requests `AIDLC/TestCaseGeneration/<agent>`, as the other stages do | Confirm the `AGENT_CONFIGS_REPO` layout, move each pack, and change the three names in the stage block (`agent`, `confidence.agent`, `confidence.exit.agent`) |

The registry is descriptive. This workspace has no Xray payload or JUnit result
XML yet. Ticket snapshots such as METERSENSE-11680 and IDEA-30 are in
`References/SDLC-Artifacts/Artifacts/`.

## Owner follow-up

The project-level [`suggestionbox.md`](../suggestionbox.md) lists the requests
for the Design, Jira, orchestration, and test repository owners. It is a
reference for later meetings; no step waits on an owner's reply.
