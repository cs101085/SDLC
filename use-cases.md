# AI-DLC — Use Case Catalogue

> **Shared source document.** This file is the single source of truth for AI-DLC use cases and is
> consumed by more than one deliverable:
>
> | Consumer | Uses | Tier |
> |---|---|---|
> | [AIDLC-31](https://harris-smartworks.atlassian.net/browse/AIDLC-31) — Functional Design & Operating Model | Use Case Catalogue | Tier 1 |
> | [AIDLC-30](https://harris-smartworks.atlassian.net/browse/AIDLC-30) — Technical Architecture & Design | Execution Sequences / sequence diagrams | Tier 2 |
>
> Maintain use cases **here** rather than in either document, so the two never drift apart.
> Minimising duplication across the four documents is an explicit goal of epic
> [AIDLC-28](https://harris-smartworks.atlassian.net/browse/AIDLC-28).

> **Status: not final.** These use cases describe current or intended behaviour at time of
> writing, not a commitment. They will be updated as development completes. Tier 2 detail is
> expected to churn most.

---

## How this document is organised

Each use case is written at **two altitudes**, so one entry serves both a functional and a
technical audience without being written twice.

**Tier 1 — Functional.** What happens and why, in business terms. No component names, no file
paths, no contracts. This is the altitude for AIDLC-31 and for any executive or product reader.
Every use case has a Tier 1.

**Tier 2 — Execution detail.** How it actually executes: participants, what is invoked, inputs
and outputs at each step, where artifacts are persisted, and where the flow can gate or fail.
This is the altitude for AIDLC-30's sequence diagrams. Tier 2 is optional — a use case that does
not warrant a sequence diagram still has a complete Tier 1 entry.

### Why two tiers rather than one

- The two documents need different depths of the same scenarios. One source, two altitudes, no
  divergence.
- Tier 2 can be omitted or cut per use case without leaving a gap.
- As development completes, Tier 2 churns while Tier 1 largely holds — which quarantines the
  volatile detail.

### A note on depth

Keep each use case to roughly one or two pages. The Word template used for these deliverables
supports only two heading levels, so within a use case the tiers and their sub-parts must be
expressed as bold labels, numbered lists, or tables rather than nested headings.

---

## Use case template

Copy this block for each new use case.

```markdown
## UC-n — <Short imperative name>

**Status:** Draft | Current behaviour | Intended behaviour
**AI involvement:** AI-Assisted | AI-Directed | AI-Delegated
**Primary actor:** <who or what initiates>

### Tier 1 — Functional

**Description**
<One short paragraph: what this use case accomplishes.>

**The journey at a glance** *(optional; include for any use case spanning more than a few steps)*
<A 10,000-foot picture: the shape of the whole flow in a short paragraph or two, plus a text flow
diagram. A reader should be able to grasp the whole scenario from this block alone, without reading
the narrative below it. Keep the flow diagram in text so it stays reviewable and version-controlled;
it doubles as the source for the rendered diagram in the Word deliverable.>

**Trigger**
<What starts it.>

**Inputs**
- <input>

**Outputs**
- <output>

**Expected benefits**
- <benefit>

**Narrative**
1. <Business-level step.>
2. <...>

### Tier 2 — Execution detail

**Systems involved**
| System | Part played |
|---|---|

**Sequence**
| Phase | Actor | Action | Writes to |
|---|---|---|---|

**Gates and failure points**
- <where the flow can pause for approval, or fail, and what happens.>
```

### Field notes

- **Status** — distinguish what exists today from what is designed but unbuilt. Readers will
  otherwise assume everything described is live.
- **AI involvement** — uses the three-level vocabulary already current at Harris
  (AI-Assisted / AI-Directed / AI-Delegated), so the catalogue is legible to readers outside
  SmartWorks R&D.
- **Trigger** — be specific about the mechanism, not just the intent.
- **Narrative** — names **external business systems** (Jira, Aha, the SDLC portal) because a
  functional reader needs them, but not **internal platform components** (orchestration engine,
  containers, repository paths, contract versions). If a step cannot be described without naming
  an internal component, it belongs in Tier 2.

### Markers

Two markers are used inline throughout, and carry the same meaning in every use case.

| Marker | Meaning |
|---|---|
| **[HUMAN GOVERNANCE]** | The step requires a person to review, approve, or decide before the flow continues. |
| **[NOT FINALISED]** | The step or mechanism is not yet settled and is expected to change. |

Every occurrence of **[NOT FINALISED]** is also collected in a register at the end of the use
case, so open items can be read without re-reading the narrative.

---

## Use case index

| # | Use case | AI involvement | Status | Tier 2 |
|---|---|---|---|---|
| UC-1 | [Ideation to feature release](#uc-1--ideation-to-feature-release) | AI-Delegated | Intended behaviour | Yes |

---

## Use cases

## UC-1 — Ideation to feature release

**Status:** Intended behaviour — target state. Alpha begins October 2026.
**AI involvement:** AI-Delegated
**Primary actor:** Product Manager

### Tier 1 — Functional

**Description**

Carries a single idea from the moment it is recorded through to a regression-tested feature merged
into a release branch with updated release notes — without a person performing the intermediate
requirements, analysis, design, implementation, test-authoring, security-review, or code-review work.
People remain responsible for approving the idea, for approving the resulting code, and for any point
where the system's own confidence in its work falls below an acceptable threshold.

This is the primary end-to-end scenario for the AI-DLC platform. Every other use case is either a
segment of this flow or a variation on it.

**The journey at a glance**

Nine phases. Three human decision points. Two pairs of phases that run simultaneously.

An idea is recorded in Jira and published to customers for feedback. When the feedback window
closes, AI summarises what came back and Product Management decides whether to proceed. From that
approval onward the platform works unattended: it generates requirements and raises a delivery Epic,
checks whether it understands the idea well enough to build it, produces a design and an
implementation guide, and reviews all of that for security risk *before* any code is written.
Implementation and test authoring then happen at the same time, and the result is security-reviewed a
second time — now looking for gaps the generated code introduced, and for secrets accidentally
committed. Finally, code review and full regression testing run in parallel and converge on a single
pull request. People approve and merge it into a release branch, which closes the ticket and updates
the release notes. Deploying that branch is owned by DevOps and out of scope.

People are asked for three things, and nothing else: approve the idea, approve the code, and resolve
anything the platform is not confident enough to continue on its own.

```text
        Idea recorded in Jira
                 │
     ┌───────────▼───────────┐
     │ 1  IDEATION           │  describe idea → publish to Aha → gather feedback
     └───────────┬───────────┘
                 ▼
        ◆ Product Management approves the idea
                 │
     ┌───────────▼───────────┐
     │ 2  REQUIREMENTS       │  generate requirements → raise Epic
     │ 3  ANALYSIS           │  is this understood well enough to build?
     │ 4  DESIGN             │  design specification + implementation guide
     └───────────┬───────────┘
                 ▼
     ┌───────────────────────┐
     │ 5  SECURITY REVIEW 1  │  risk in the idea, requirements and design
     └───────────┬───────────┘
                 ▼
     ┌─────────────────┬─────────────────┐
     │ 6a CODE GEN     │ 6b TEST CASE GEN│   ← in parallel
     └─────────────────┴────────┬────────┘
                                ▼
     ┌───────────────────────┐
     │ 7  SECURITY REVIEW 2  │  introduced gaps, committed secrets
     └───────────┬───────────┘
                 ▼
     ┌─────────────────┬─────────────────┐
     │ 8a REVIEW → PR  │ 8b VERIFICATION │  ← in parallel
     └─────────────────┴────────┬────────┘
                                ▼
        ◆ Peer review → requestor merges to the release branch
                 │
     ┌───────────▼───────────┐
     │ 9  RELEASE            │  merge to release branch → summarise → close ticket
     └───────────────────────┘

     ◆ = human decision point
     Every phase produces a confidence rating and is measured against its
     own threshold. Falling short sets the execution to FAILED with a
     stated reason, and hands it to a person to resolve.

     The flow above shows the primary entry — an idea. A Jira bug enters
     further along, and a champion can push existing work in on demand.
     See Secondary entry points below.
```

**Trigger**

*Primary.* An idea recorded in the Ideas project in Jira is moved to a "Ready for Feedback" state,
which fires a webhook to the AI-DLC workflow. **[NOT FINALISED]**

**Secondary entry points**

The workflow does not require an idea to start it. Two further entry points join the same flow
further along, skipping the phases that do not apply to how the work arrived:

| Entry point | Enters at | Skips | Why |
|---|---|---|---|
| A Jira **bug**, on creation — a webhook fires automatically | Analysis **[NOT FINALISED]** | Ideation, Requirements | A defect is its own justification. There is no idea to validate and no customer feedback period to run, and the bug report already carries the requirement. |
| **On demand** — a champion pushes an existing feature or ticket through the workflow | The phase selected at trigger time | Everything preceding that phase | Used to run work already recorded in Jira, or to re-enter a flow at a chosen point. |

**Downstream of the entry point, behaviour is identical regardless of how the execution began** —
the same phases, the same gates, the same confidence thresholds, the same artifact space. Only the
starting point differs. This is deliberate: a single flow with several doors into it, rather than a
separate pipeline per kind of work.

**Inputs**

- An idea recorded in the Ideas project in Jira
- Preliminary analysis and markers of success, detailed on the ticket
- The existing source code for the affected solution
- Customer feedback gathered in Aha

**Outputs**

- A written description of the idea, published for customer feedback
- A documented review of the feedback received
- Generated requirements, and an Epic in the relevant delivery project(s)
- An analysis of whether the idea is sufficiently understood to build
- A design specification and an implementation guide
- Two security assessments — one before implementation, one after
- Generated source code on a feature branch
- Unit and integration tests alongside the generated code, and an independently derived functional
  test suite
- A full regression test result
- A pull request with an AI-authored review description, approved by people and merged into a
  release branch
- Updated release notes and a closed Jira ticket
- A complete, retained record of every artifact produced at every phase

**Expected benefits**

- Compresses elapsed time from recorded idea to merged feature
- Produces the same artifacts in the same shape on every execution, so executions can be compared
  directly and improvement can be measured rather than asserted
- Retains every intermediate decision as an inspectable artifact instead of discarding it
- Applies security review twice — before and after implementation — as a standard step rather than
  an exception
- Places human attention only where it carries the most weight: approving the idea, approving the
  code, and resolving cases the system is not confident about
- Gathers customer input before build effort is committed, rather than after

**Phase overview**

| # | Phase | Purpose | Runs |
|---|---|---|---|
| 1 | Ideation | Describe the idea, gather and review customer feedback, secure approval to proceed | Sequential |
| 2 | Requirements | Generate requirements and raise the delivery Epic | Sequential |
| 3 | Analysis | Judge whether the idea is understood well enough to build, and determine the source repository and branch | Sequential |
| 4 | Design | Produce the design specification and implementation guide, and raise the delivery Stories | Sequential |
| 5 | First Security Review | Assess the idea, requirements and design for security risk | Sequential |
| 6a | Code Generation | Generate and commit the implementation, with its unit and integration tests | Parallel with 6b |
| 6b | Test Case Generation | Produce the test case document and executable functional test suite | Parallel with 6a |
| 7 | Second Security Review | Assess the generated code and tests for introduced risk and committed secrets | Sequential |
| 8a | Review | Derive the pull request and hand it to human reviewers | Parallel with 8b |
| 8b | Verification | Build and run the full regression suite | Parallel with 8a |
| 9 | Release | Confirm the merge into the release branch, summarise the feature, close the ticket, update release notes, resolve the execution | Sequential |

**Narrative**

*Ideation*

1. An idea is recorded in the Ideas project in Jira.
2. Preliminary analysis is performed, with the markers of success detailed on the ticket. The ticket
   is moved to a "Ready for Feedback" state. **[NOT FINALISED]**
3. That transition triggers the AI-DLC workflow, which creates an SDLC Execution and its artifact
   space, and enters the Ideation phase.
4. AI writes a description of the idea. The orchestrator publishes it to Aha requesting customer
   feedback, and establishes a feedback period, producing a feedback due date. The period takes a
   default length, which the person actioning the idea can override on the Jira Idea
   ticket. **[NOT FINALISED]**
5. Having published the idea, the platform's work is done for now and it stops. Feedback is gathered
   in Aha until the due date passes.
6. Once the feedback period has closed, a Product Manager resumes the work from where it left off.
7. AI reviews the gathered feedback and documents its findings in the artifact space.
8. AI evaluates the idea against the Ideation phase's confidence threshold, assessing the content of
   the Jira ticket together with the outcome of the feedback period. This assessment weighs return on
   investment, complexity, and what customers actually said — so the threshold serves as a judgement
   on whether the idea is worth pursuing at all, not merely whether it is well described.
9. If the confidence rating meets the threshold, Product Management is notified for
   approval. **[HUMAN GOVERNANCE]**
10. On approval, the execution moves to the Requirements phase.

*Requirements*

11. AI generates requirements for the idea and writes the requirements assets to the artifact space.
12. An Epic is created in the relevant delivery project(s) in Jira. The execution moves to the
    Analysis phase.

*Analysis*

13. AI reviews the Jira ticket together with the artifacts produced so far, to determine whether the
    idea is understood thoroughly enough to progress or whether information is missing. Its findings
    are written back to both the Jira ticket and the artifact space.
14. Analysis also determines **which source repository and branch** the work applies to, and records
    them for every later phase to work from. Where it cannot determine them, the phase fails with its
    reasoning. **[HUMAN GOVERNANCE]** The execution moves to the Design phase.

*Design*

15. The source branch recorded by Analysis is retrieved.
16. AI reviews the requirements and analysis artifacts alongside the checked-out code, and produces a
    Design document — the system design specification — and an Implementation Guide giving direct
    instructions for changing the code. Both are stored in the artifact space.
17. One or more Jira Stories are created under the Epic, each carrying the design detail relevant to
    it. Every Story also links to the overall Design document, so the complete design stays reachable
    from any individual Story rather than only from the Epic.
18. Where the design warrants more than one Story, the split follows either the **type of work**
    involved — front end, back end, API, and so on — or the **component being worked on** — rules
    engine, VEE, reports, and so on — whichever the design itself calls for. The execution moves to
    the First Security Review phase.

*First Security Review*

19. AI analyses the idea, requirements and design to identify security concerns or risks, and writes a
    security report to the artifact space. The report carries its own confidence rating, accounting
    for both the level of detail available to assess and the security risk profile identified. The
    threshold for this phase is deliberately high. Meeting it advances the execution to two phases
    that run in parallel; falling short fails the phase, with the report stating
    why. **[HUMAN GOVERNANCE]**

*Code Generation and Test Case Generation — in parallel*

20. **Code Generation.** A new feature branch is created. AI generates code into that branch using
    the Design document and Implementation Guide, together with its **unit and integration tests**,
    and commits the result.
21. **Test Case Generation.** AI produces a test case document describing what to test, and saves it
    to the artifact space. AI then generates an executable Playwright test suite and commits it to the
    test repository. The artifact space is updated accordingly. This phase covers **functional and
    end-to-end** testing only; unit and integration tests belong to Code Generation.

    The two phases run in parallel because neither depends on the other. Both derive their work from
    the Design output and everything preceding it — requirements and analysis — and neither consults
    the other. **Test Case Generation is deliberately kept unaware of the generated code**, because a
    test suite written with knowledge of the implementation carries an unconscious bias toward it and
    stops being an independent check.

*Second Security Review*

22. Once both parallel phases have completed, AI reviews the generated code and the generated
    Playwright suite to confirm that no security gaps have been introduced into the solution and that
    no secrets have been committed. As with the first security review, the resulting report carries a
    confidence rating weighing both detail and risk profile against a high threshold, and states its
    reasoning where that threshold is not met. **[HUMAN GOVERNANCE]** Results are written to the
    artifact space, and the execution proceeds to two further phases that run in parallel.

*Review and Verification — in parallel*

23. **Review.** AI reviews the generated code and derives a Pull Request with an appropriate review
    description and comments. The pull request is then placed for people to
    review. **[HUMAN GOVERNANCE]**
24. **Verification.** The solution is built and the full unit test and Playwright suites are
    executed, effectively performing a full regression test. The pull request and the artifact space
    are updated with the results.

*Release*

25. Creating the pull request triggers the CI stack and the repository's runners. The change enters
    peer review — currently two approvers minimum, at least one of them a senior developer or above
    — alongside the scanners and linters the CI stack runs against
    it. **[HUMAN GOVERNANCE]**
26. Once all approvals are granted, **the requestor merges** the change into the release branch,
    since they own the pull request. The CI stack then closes the source branch. **[HUMAN GOVERNANCE]** Conflict resolution is triggered there too, and feedback throughout goes to the
    requestor. **For the code, this is where AI-DLC's involvement ends.**
27. The requestor reviews their execution in the SDLC portal. Once the pull request has merged, they
    trigger the Release phase. **[HUMAN GOVERNANCE]** *AI-DLC does not poll the pull request; the
    transition is a deliberate human gate. Polling is a roadmap item for 1.0 or later.*
28. Where the pull request was **declined**, the Release phase fails the execution with the reason
    recorded and emails the requestor, who owns it. **[HUMAN GOVERNANCE]**
29. Where it was **merged**, AI generates an overview of the feature. The Jira ticket is updated and
    closed, the release notes are updated with the new feature, and the requestor is notified.
30. The execution is resolved and its artifact space closed. Deploying the release branch is owned by
    the DevOps team and out of scope.

**Phase confidence**

Confidence is the mechanism by which the platform decides whether it is safe to keep going without
asking a person. It applies at every phase, and it is the single most important control in the flow.

**What it is.** Before a phase does its own work, AI reviews the assets it has been handed and judges
whether they are sufficient to proceed — expressed as a **confidence rating in percent**. That rating
must **surpass a threshold** for the phase to proceed, and each phase defines **its own**, because the
phases are not equally tolerant of ambiguity.

**What it is not.** It is not a measure of quality of output, and it is not a test result. It is a
statement about whether the *inputs* support the work about to be done. A phase can be highly
confident and still produce something a person disagrees with — that is what the human governance
points are for.

**Entry gating versus exit gating.** The direction of the gate differs by phase, and the distinction
matters:

| Gating | Applies to | Question asked |
|---|---|---|
| **Entry** | Every phase except the security reviews | *Given what the previous phase produced, can I begin my work?* |
| **Exit** | Both security review phases | *Given what I have just assessed, is it safe to let this execution continue?* |

The security reviews generate their own confidence rating rather than judging a predecessor's assets
alone. That rating accounts for **both** the level of detail available to assess **and** the security
risk profile identified — a review can be blocked either because the risk is real or because there was
not enough substance to assess it properly. Their thresholds are set deliberately high, and where the
threshold is not met the security report itself carries the explanation.

**The Ideation phase is a special case** because it has no predecessor. Its gate assesses the content
of the Jira Idea ticket together with the outcome of the customer feedback period, weighing return on
investment, complexity, and customer response. This makes the Ideation threshold a judgement on
whether the idea is **worth pursuing**, not merely on whether it is adequately described — the one
gate that can end an execution on economics rather than on clarity.

**What happens below threshold.** The execution is **set to a failed state carrying a stated
reason**, and handed to a person to resolve. **[HUMAN GOVERNANCE]** The reason is part of the record
rather than just a status, so whoever picks it up can see why it stopped without re-running anything.
The **requestor** is emailed and picks it up — see **Roles and Governance** below.

**Why this is the control that matters.** Determinism is the point of orchestrating the SDLC rather
than delegating it wholesale to an AI. Confidence thresholds are what keep that determinism honest:
they stop the platform from proceeding confidently on a weak basis, and they place human attention
precisely where the system itself has signalled doubt — rather than at every step, or nowhere.

Threshold values are **not yet set** for any phase. **[NOT FINALISED]**

**Roles and Governance**

Two distinct kinds of ownership operate here, and keeping them apart is the point.

**Stage ownership — the SDLC Champions**

Each stage of the workflow is owned by an **SDLC Champion**, a domain representative drawn from the
committee of that name. Within AI-DLC, a champion is responsible for:

- **Writing and owning their stage's workflow** — the orchestration definition itself.
- **Owning their stage's agent** — the agent definition that stage invokes.
- **Continuously improving both** as the programme matures and the platform is used in anger.

A champion is the **author and steward of a stage**, not an operator of it. They are explicitly
**not** responsible for actioning failures in individual executions.

This is why the authoring surface matters: champions write workflows, so holding workflow definitions
in a Git repository — and using its pull request process as the publishing and peer-review model —
gives them a review path over each other's work. How far that tooling needs to go depends on what the
champions ask for once they are using it.

| Stage | SDLC Champion |
|---|---|
| Ideation | Director, Products & Research |
| Requirements | Solution Architect |
| Analysis | *Not yet assigned* **[NOT FINALISED]** |
| Design | Systems Architect |
| First Security Review | Security Officer |
| Code Generation | Senior Software Engineer |
| Test Case Generation | Senior QA Analyst |
| Second Security Review | Security Officer |
| Review | Senior Software Engineer — deliberately **not** the same individual as Code Generation |
| Verification | Senior QA Analyst — the **same** individual as Test Case Generation **[NOT FINALISED]** |
| Release | Director of Software Development |

Assignments are for Alpha and will change as the programme matures. Two stages are held by the same
title but deliberately different people — Code Generation and Review — and separating the authorship
of code generation from the authorship of code review is a control rather than an accident of
staffing.

**Execution ownership — the requestor**

Every execution is owned by **the person who requested it**, identified by the Requestor Email
supplied in the triggering request. That person:

- Is **emailed directly** whenever the execution raises a gate or fails.
- **Analyses the artifacts** and makes changes where necessary.
- **Owns the resulting pull request** — receives the CI stack's feedback on it, and **performs the
  merge** into the release branch once all approvals have been granted.

The separation from stage ownership is deliberate. The champion who built a stage is not interrupted
every time a run of it needs attention, and the person who asked for the work is the one accountable
for carrying it through. It also means **gate ownership does not vary by stage** — whoever started an
execution answers for all of it, from Ideation to Release.

**Human gates**

Every row below is a **human gate** — a point at which the workflow stops and a person decides
whether, and how, it continues. Gates are the mechanism by which governance is maintained over an
otherwise automated pipeline: between gates the platform runs unattended, and it cannot pass a gate
without a human judgement.

**Every one of these is answered by the requestor**, who is emailed when it is raised. The stage
column says where in the flow the gate sits, not who answers it.

Outcomes a phase produces are deliberately **not** listed here. An artifact being correct is not a
gate — a gate is specifically a trigger for human involvement. The **Customer** likewise holds no
gate: they contribute feedback during the Ideation feedback period, which the workflow then reviews
and puts to Product Management for approval.

| Stage | Gate |
|---|---|
| Ideation | Idea inception — the idea is recorded in Jira with its preliminary analysis and markers of success, and moved to "Ready for Feedback". Nothing enters the workflow without this<br>Approval of the idea by Product Management, following review of the customer feedback gathered in Aha<br>Confidence below the Ideation threshold |
| Requirements | Confidence below the Requirements threshold |
| Analysis | Confidence below the Analysis threshold — whether the gap is one of scope, intent, or technical feasibility |
| Design | Confidence below the Design threshold, including where the design carries structural or cross-component consequence |
| First Security Review | Confidence below the First Security Review threshold — gating **exit** from the stage. Whether an identified risk blocks or is accepted is decided here |
| Code Generation | Confidence below the Code Generation threshold |
| Test Case Generation | Confidence below the Test Case Generation threshold |
| Second Security Review | Confidence below the Second Security Review threshold — gating **exit** from the stage. Covers security gaps introduced by the generated code and any secrets committed |
| Review | Confidence below the Review threshold<br>The pull request itself — peer review, currently two approvers minimum with at least one a senior developer or above. Conducted through the repository's own process, not by AI-DLC. The principal human gate in the flow |
| Verification | Confidence below the Verification threshold<br>Regression failures in the unit or Playwright suites, gated through the pull request |
| Release | Entry to Release — the requestor, having reviewed the execution in the SDLC portal, confirms the pull request has merged and triggers the stage<br>A declined pull request — the execution has failed and requires the requestor's attention<br>Confidence below the Release threshold |

**How human governance is handled**

Where a phase is flagged for human governance, a reviewer inspects the execution and updates its
assets as needed through the SDLC portal. Doing so records a change against the execution's artifact
space, and the workflow then either continues from that point or is returned to an earlier phase
carrying the new detail.

**Portal capability differs between Alpha and 1.0** — and the governance loop above describes the
1.0 target, not Alpha:

| Capability | Alpha (MVP) | 1.0 |
|---|---|---|
| Review an execution and its generated assets | Yes | Yes |
| Edit assets in the portal | No | **Mandatory** |
| Reprocess or resume an execution from the portal | No | **Mandatory** |

For **Alpha**, the portal is a review surface only. A reviewer can see an execution, read its
artifacts, and understand why it stopped — but resolving a gate and resuming the workflow happens
outside the portal, through a direct call to the trigger API. **How a reviewer corrects assets before
re-triggering in Alpha is not yet defined.** **[NOT FINALISED]**

Further 1.0 portal capabilities are still being scoped. **[NOT FINALISED]**

**Visibility**

An execution and its artifact space can be reviewed at any time in the SDLC portal. Any human
governance that does not concern the pull request is handled there; pull request review happens in
the source repository as normal.

**Where AI-DLC stops and the CI stack starts**

The CI stack and the AI-DLC workflow are deliberately separated, so neither one's concerns can affect
the other. **For the code, the pull request is the end of the line for AI-DLC.**

| Responsibility | Owner |
|---|---|
| Create the pull request | **AI-DLC** |
| Advance to Release once the pull request has merged | **A human gate** — the requestor, having reviewed the execution in the SDLC portal |
| Peer review — two approvers minimum, at least one a senior developer or above | CI stack |
| Scanners and linters | CI stack |
| Triggering conflict resolution | CI stack |
| Providing feedback to the requestor | CI stack |
| Merge to the release branch, once all approvals are granted | **The requestor** — they own the pull request |
| Closing the source branch | CI stack |

**Creating the pull request is what triggers the CI stack** and the repository's runners for that
repository. From then on the code is out of AI-DLC's hands entirely — it does not watch, poll, or react to the pull
request. Advancing to Release is a human gate.

The Release phase's remaining work is deliberately **non-code** — the feature overview, the release
notes, and closing the Jira ticket. The code was finished at the merge.

**Scope boundary**

AI-DLC's responsibility **for the code** ends the moment the feature is merged into a release branch
in the repository. That is what "release" means throughout this document — the Release phase names
the branch the work lands in, not delivery to a customer.

**Deploying that release branch is out of scope and owned by the DevOps team.** Environment
promotion, release scheduling, rollback and any post-deployment verification sit entirely outside
AI-DLC.

**The Release phase does not end at the merge.** Once the merge is confirmed, the phase completes the
record of what shipped: AI generates a summary of the feature, the release notes are updated with it,
and that information is written back to the Jira ticket, which is then closed. Only then is the
execution resolved and its artifact space closed.

So the platform's deliverable is a merged feature in a release branch, described in the release
notes, recorded on its ticket, and backed by a complete artifact record of how it was produced.
Deploying it is someone else's step.

### Tier 2 — Execution detail

**Systems involved**

| System | Part played |
|---|---|
| Jira — Ideas project | Origin of the idea; the state transition that triggers the execution |
| Jira — delivery project(s) | Receives the generated Epic and Stories; ticket updated and closed at release |
| Aha | Publishes the idea to customers and collects feedback against a due date |
| AI-DLC Orchestrator | Owns stage sequence, execution state, parallel fan-out, gate evaluation, and the artifact space lifecycle |
| AI agent (per phase) | Executes that phase's unit of work against the artifacts it is given |
| SDLC-Artifacts repository | Holds the artifact space for every execution, and the manifest recording state and history |
| SDLC portal | Human review surface: inspect an execution, edit assets, resolve non-PR governance |
| Source repository | Holds the identified base branch, the generated feature branch, and the pull request |
| CI stack and repository runners | Triggered by the pull request. Owns peer review, scanners and linters, conflict resolution, requestor feedback, the merge to the release branch, and closing the source branch |
| Test repository | Receives the generated Playwright test suite |
| Release notes | Updated at release |

**Sequence**

UC-1 spans **four executions**, not one — an execution ends rather than waiting, so every point that
depends on something outside the platform is an execution boundary. Each resume is an ordinary
trigger request supplying the prior Execution ID and the stage to resume from.

*Before any execution*

| Actor | Action | Writes to |
|---|---|---|
| Originator | Record the idea; add preliminary analysis and markers of success | Jira Ideas project |
| Originator | Move the ticket to "Ready for Feedback" **[NOT FINALISED]** | Jira Ideas project |
| Jira | Fire the webhook to the orchestrator | Orchestrator |

*Execution 1 — publish the idea for feedback.* Triggered by the webhook; ends in **success** once the
idea is published.

| Phase | Actor | Action | Writes to |
|---|---|---|---|
| Initialization | Orchestrator | Create the execution and artifact space; retrieve the ticket; derive Ideation as the initial stage | SDLC-Artifacts |
| Ideation | AI agent | Write the idea description | Artifact space |
| Ideation | Orchestrator | Publish the idea for feedback; set the feedback period from default or ticket override; derive the due date | Aha, artifact space |
| Ideation | Orchestrator | Record `lastStep` in stageParams; end the execution in **success** | Manifest |

> *Between executions —* customers provide feedback in Aha until the due date passes.

*Execution 2 — review the feedback and seek approval.* Triggered by a Product Manager once the
feedback period has closed; ends in **success** once approval has been requested.

| Phase | Actor | Action | Writes to |
|---|---|---|---|
| Initialization | Orchestrator | Clone execution 1 — copy artifacts, carry stageParams forward, fresh registry | SDLC-Artifacts |
| Ideation | AI agent | Read stageParams; retrieve and review the gathered feedback; document findings | Artifact space |
| Ideation | AI agent | Rate confidence from ticket content and feedback outcome — RoI, complexity, customer response | Artifact space |
| Ideation | Orchestrator | Notify Product Management **[HUMAN GOVERNANCE]**; end the execution in **success** | Manifest |

> *Between executions —* Product Management decides whether the idea proceeds.

*Execution 3 — requirements through to an open pull request.* Triggered on approval, from
Requirements; ends in **success** once the pull request is open and regression results are published.

| Phase | Actor | Action | Writes to |
|---|---|---|---|
| Requirements | AI agent | Generate requirements | Artifact space |
| Requirements | Orchestrator | Create the Epic in the relevant delivery project(s) | Jira delivery project(s) |
| Analysis | AI agent | Assess completeness of understanding; record findings | Jira ticket, artifact space |
| Analysis | AI agent | Derive the source repository and branch for the work | Artifact space |
| Design | Orchestrator | Retrieve the base branch recorded by Analysis | Working copy |
| Design | AI agent | Produce the design specification and implementation guide | Artifact space |
| Design | Orchestrator | Create one or more Stories under the Epic, split by type of work or by component; carry design detail into each and link each to the design | Jira delivery project(s), artifact space |
| Security Review 1 | AI agent | Assess idea, requirements and design for security risk; rate confidence on detail and risk profile | Artifact space |
| Code Generation | Orchestrator | Create the feature branch | Source repository |
| Code Generation | AI agent | Generate code with its unit and integration tests; commit | Source repository, artifact space |
| Test Case Generation | AI agent | Produce the test case document — functional and end-to-end scope, without reference to the generated code | Artifact space |
| Test Case Generation | AI agent | Generate the Playwright suite; commit | Test repository, artifact space |
| Security Review 2 | AI agent | Triggered by both predecessors; proceeds exactly once under the registry guard. Review generated code and tests for introduced risk and committed secrets; rate confidence | Artifact space |
| Review | AI agent | Review the generated code; derive the pull request with description and comments | Source repository, artifact space |
| Review | Orchestrator | Place the pull request for human review **[HUMAN GOVERNANCE]** | Source repository |
| Verification | Orchestrator | Build the solution; execute the full unit and Playwright suites | — |
| Verification | Orchestrator | Publish regression results | Pull request, artifact space |
| — | Orchestrator | End the execution in **success** once both parallel stages have completed | Manifest |

> *Between executions —* the CI stack takes over the code: peer review, scanners and linters, and on
> closure of the source branch once merged. The requestor performs the merge itself, once all
> approvals are granted. AI-DLC watches the
> requestor reviews the execution in the SDLC portal and, once the pull request has merged, triggers
> the Release stage.

*Execution 4 — release.* Triggered once the pull request has been decided, from Release.

| Phase | Actor | Action | Writes to |
|---|---|---|---|
| Release | Orchestrator | **Declined path:** end the execution in **failure** with the reason recorded, and notify the requestor | Manifest |
| Release | AI agent | **Merged path:** generate the feature overview | Artifact space |
| Release | Orchestrator | Update and close the Jira ticket; update the release notes; notify the requestor | Jira, release notes |
| Release | Orchestrator | End the execution in **success**; close the artifact space | SDLC-Artifacts |

**Parallel execution**

Two parallel pairs exist. Neither is coordinated by a join — the successor is triggered by both
predecessors and proceeds exactly once under the Stage Registry's exclusive claim:

- After the First Security Review: **Code Generation** and **Test Case Generation** run
  concurrently, and neither consults the other's output. The First Security Review triggers both;
  the Second Security Review follows their completion.
- After the Second Security Review: **Review** and **Verification** run concurrently. Both
  contribute to the same pull request.

**Execution boundaries**

This use case does not run as one continuous execution. **An execution never waits** — where the
platform can do no more until something outside it happens, it records where it got to and completes
with *success*, and a person later resumes from that stage.

Resuming needs no special mechanism: it is the ordinary trigger request, supplying the prior
**Execution ID** and the stage to resume from as **Initial Stage**.

| Boundary | Waiting on | Resumes at |
|---|---|---|
| After the idea is published to Aha | The customer feedback period to close | Ideation |
| After Product Management is notified | Approval of the idea | Requirements |
| After the pull request is opened | The CI stack's peer review and checks, the requestor's merge, then the requestor to trigger the next stage | Release |
| Any confidence failure | The requestor to resolve the gate | The failed stage |

The first three are planned gates and end in **success** — the execution did what it was asked to.
The fourth is a **failure** carrying its reason. Both need someone to come back; only one means
something went wrong.

Context survives each boundary through **stageParams** in the manifest, which is carried forward when
an execution is cloned. Ideation resumed, for example, reads that it had already submitted to Aha and
knows the remaining work is to review the feedback.

*A consequence worth stating: nothing prompts. An idea published for feedback, or a pull request
sitting merged, waits on a person noticing — carried only by its Jira ticket in the meantime. A
recurring routine to detect ready executions and advance them is a recorded roadmap item for 1.0 or
later, deliberately excluded for now in favour of an explicit human gate.*

**Terminal states**

An execution ends as either **success** or **failure**, and nothing else — there is no held or
waiting state.

Any phase that cannot proceed sets the execution to **failure** carrying the reason, rather than
halting silently. This applies to a confidence rating below threshold, to either security review
falling short, and to a declined pull request. The reason is written into the execution record, so
whoever picks it up can see why it stopped without re-running anything.

**Success means the execution did its job, not that the feature shipped.** An execution that publishes
an idea and stops has succeeded. The manifest's recorded initial and final stage are what show how
much of the lifecycle a given execution covered; the terminal state alone does not.

**Artifact space**

The artifact space is a folder in the SDLC-Artifacts repository, holding every artifact generated in
every phase of that execution. It also contains a `manifest.md` recording the execution's current
state and its history. Because the artifact space is a repository folder, every change to it is a
commit, and the execution's history is therefore versioned and auditable by construction.

**Human governance loop**

A phase flagged for human governance is resolved by the **requestor**, working through the SDLC
portal. They analyse the artifacts and update them where necessary, which records a commit into the
SDLC-Artifacts repository. The workflow then
either continues from the flagged phase, or is returned to an earlier phase carrying the new detail —
so governance is a loop back into the flow, not a terminal state.

**Gates and failure points**

Mechanism only. Every one of these is answered by the **requestor** — see **Roles and Governance** above.

| Gate | Type | Condition | Outcome |
|---|---|---|---|
| Per-phase confidence, entry | Automatic → human | The assets handed to the phase do not support beginning its work, rated below the phase's threshold | Execution set to failed with reason |
| Ideation confidence | Automatic → human | Assessment of ticket content and feedback outcome — RoI, complexity, customer response — below threshold | Execution set to failed with reason; effectively a judgement that the idea is not worth pursuing as it stands |
| Idea approval | Human | Confidence met and Product Management notified | Approval advances to Requirements |
| First Security Review, exit | Automatic → human | Confidence on detail and risk profile below a deliberately high threshold | Phase fails with the security report stating why. Meeting it advances to Code Generation and Test Case Generation |
| Second Security Review, exit | Automatic → human | Confidence below threshold; security gaps introduced, or secrets committed | Phase fails with the security report stating why |
| Verification | Automatic → human | Regression failures in the unit or Playwright suites | Results are published to the pull request; the failure is gated there rather than failing the execution independently |
| Pull request review | Human, in the CI stack | Pull request created, triggering the repository's runners | Peer review, scanners and linters run there. Once all approvals are granted the requestor merges, and the CI stack closes the source branch, and AI-DLC's status check advances to the merged path in Release. A decline fails the execution and emails the requestor |

### Not-finalised register

| Item | Status |
|---|---|
| The "Ready for Feedback" Idea state that triggers the workflow | Not finalised. Depends on the Ideas Jira project workflow, which does not exist yet |
| Which phase a Jira bug enters at | Analysis is the working assumption — a defect needs no idea validation and carries its own requirement — but not settled |
| Session-based Requirements interaction — shaping requirements iteratively by conversation with AI, likely surfaced in the SDLC portal | Under consideration. **Not in the Alpha beginning October 2026.** |
| Feedback period length — intended as a default value, overridable on the Jira Idea ticket by the person actioning the idea | Approach agreed; mechanism not finalised |
| How the "relevant" delivery project(s) are selected for Epic creation | Not specified |
| Per-phase confidence threshold values | Not yet set. The security review thresholds are stated as deliberately high; no numbers are established for any phase |
| **Analysis has no SDLC Champion assigned** | Every other stage has one. Needs an owner before Alpha, since the stage has a confidence gate and therefore a first responder |
| **Verification champion title** | Recorded as Senior QA Analyst, being the same individual as Test Case Generation. Stated as "Senior Software Engineer (the same as Test Case Generation)" — read as the same *person*, so the title follows Test Case Generation. Confirm |
| **Polling the pull request** | Deferred to 1.0 or a post-1.0 roadmap item. For now the requestor reviews the execution in the SDLC portal and a human gate advances it to Release |
| SDLC portal scope | Alpha is review-only. Asset editing and execution reprocessing are mandatory for 1.0. Remaining 1.0 capabilities still being scoped |
| Alpha governance resolution path | With the portal review-only in Alpha, how a reviewer corrects assets and resumes an execution is not yet defined |
