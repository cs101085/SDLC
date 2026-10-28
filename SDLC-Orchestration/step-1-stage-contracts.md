# Step 1: Stage contract proposal

This proposal settles the stage interfaces without changing the reference
orchestration project. The blocks below are the proposed `variables.stage`
contracts to apply when the stages are implemented.

## Test Case Generation

```yaml
stage:
  name: TestCaseGeneration
  ordinal: 6
  artifactDir: TestCaseGeneration
  gate: exit
  confidenceThreshold: 50
  stub: false

  predecessors:
    - FirstSecurityReview

  successors:
    - SecondSecurityReview

  requires:
    - analysis: { as: analysis.md }
    - design: { as: design.md }
    - implementation: { as: implementation.md }
    - repoSlug
    - baseBranch
    - bitbucketProject

  provides:
    - testPlan: { kind: artifact, file: testplan.md }
    - testCode: { kind: artifact, file: testcode.md }
    - xrayPayload: { kind: artifact, file: xray-payload.json }
```

`featureBranch` is deliberately absent. The source-repository placeholder is
the base-branch source only, so Code Generation can remain parallel and the
agent cannot inspect its generated feature branch. `repoSlug`, `baseBranch`,
and `bitbucketProject` are execution bootstrap values in the current graph.

`analysis` is required alongside the design artifacts because there is no
Requirements artifact, and F12 ranks `analysis.md` second for every work kind.
Analysis provides it, and Design already requires it the same way.

The confirmed `exit` gate follows F6: generation should be judged on the
coverage it produces, not only on the confidence of its inputs. The gate
setting selects one assessment point for this stage; it does not request
both entry and exit assessments. At entry, confidence is assessed after input
resolution and before domain work. At exit, it is assessed after domain work,
so a low rating can prevent successful completion and block successors.
Both choices use the same threshold of 50.

This changes the current stub's `entry` gate. The TestCaseGeneration gate is
confirmed as `exit`. Keep the current orchestration confidence helper as a
test stand-in that records 100 and passes; wiring a real confidence assessor
is deferred.

## Verification

```yaml
stage:
  name: Verification
  ordinal: 8
  artifactDir: Verification
  gate: entry
  confidenceThreshold: 50
  stub: false

  predecessors:
    - SecondSecurityReview

  successors:
    - Release

  requires:
    - testCode: { as: testcode.md }
    - xrayPayload: { as: xray-payload.json }

  provides:
    - junitResults: { kind: artifact, file: junit-results.xml }
```

The runner itself remains a placeholder. The contract describes its handoff:
the suite and the Xray execution payload go to the runner, and JUnit XML is
collected as the stage result. Steps 2, 6, and 7 define the exact artifact
formats and implement these outputs. Keep the current stub until that work
exists.

## Edge check

No graph edge changes are proposed:

| Stage | Predecessors | Successors |
|---|---|---|
| TestCaseGeneration | FirstSecurityReview | SecondSecurityReview |
| Verification | SecondSecurityReview | Release |

Both ends still agree in the reference stage declarations:

- `FirstSecurityReview` names `TestCaseGeneration` as a successor.
- `SecondSecurityReview` names `TestCaseGeneration` as a predecessor and
  `Verification` as a successor.
- `Release` names `Verification` as a predecessor.

The proposed `testCode` and `xrayPayload` outputs are provided by
TestCaseGeneration, an ancestor of Verification through SecondSecurityReview.
The contracts add no branch output and do not reorder the parallel stage 6
pair.

## Placeholder registry

The entries below name real local paths for this draft. Paths under
`References/` are read-only; generated output is staged in the corresponding
top-level repository folder. A README is the deliberate local stand-in where
the live integration or its output does not exist yet.

| Name | Resolves to today | Becomes | On migration |
|---|---|---|---|
| `TEST-REPO-READ` | `References/Test Repository/` | Read-only `git.clone-source` of the test repository | Add the repository to `_config`, supply credentials, and call with `newBranch: ""` |
| `TEST-REPO-WRITE` | `Test Repository/README.md` | Branch, commit, and push to the test repository | Decide the branch strategy and merge owner; mirror `commit-and-push.sh` |
| `SOURCE-REPO-READ` | `References/metersense-release-5.1/` | Read-only clone of the source repository at `baseBranch` | Use the existing `git.clone-source` integration |
| `ARTIFACT-SPACE` | `SDLC-Artifacts/README.md` | `artifacts.publish-workspace` into SDLC-Artifacts | Replace direct writes with the workspace lifecycle subflows |
| `TICKET-INPUT` | `Example Tickets/METERSENSE-10101.md` | `Jira/input.json` staged by the orchestrator | Read the staged ticket instead of the sample |
| `XRAY-API` | `SDLC-Artifacts/README.md` (local stand-in only) | Direct Xray API calls from Kestra | Supply credentials as globals and send the payload instead of writing it |
| `JUNIT-RESULTS` | `SDLC-Artifacts/README.md` (destination documented; no result XML yet) | `target/surefire-reports/TEST-*.xml` from a real run | Point collection at the runner's output directory |
| `VERIFICATION-RUNNER` | This contract's Verification section | The runner container from a later project | Implement the documented runner interface |

The registry is descriptive, not evidence that a live service is connected.
In particular, this workspace has no Xray payload or JUnit result XML yet;
their output formats and files belong to the later steps listed above.

## Owner follow-up

The TestCaseGeneration `exit` gate and confidence helper's test rating of 100
are confirmed. Before Step 1 is closed, share the project-level
`suggestionbox.md` with the Design-stage owner. No owner notification
mechanism is available in this draft workspace, so that cross-department
review remains outstanding.
