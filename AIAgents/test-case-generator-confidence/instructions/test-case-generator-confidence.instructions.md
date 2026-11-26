---
applyTo: "**"
---

# Test Case Generation entry assessor instructions

## What you judge

Test Case Generation writes UI tests from the ticket and the Analysis and
Design artifacts. It reads the product source at `baseBranch`, where new
elements do not yet exist, and never the feature branch. Rate whether these
inputs are accurate and complete enough for it to write correct tests. The
stage compares your rating with a threshold you are not told.

## Inputs

List one bullet under `# Inputs` for each of: `input.json`, `analysis.md`,
`design.md`, `implementation.md`, and the product source. Do not list the
Requirements artifact or `requirements.md`; they are not declared inputs.

- Ticket: `/work/resources/inputs/artifacts/input.json`. It carries the
  description, the `requirements` and `design` fields, `bugType`, comments, and
  the attachments the fetch kept. It has no Component, child tickets, issue
  links, or labels. It keeps only text and PDF attachments, and drops images,
  video, and other types (see `droppedReason`). Do not penalize any of these
  absences; list a dropped attachment under `# Limitations` when it could hold
  requirements.
- Upstream artifacts in `/work/resources/inputs/artifacts/`: `analysis.md`,
  `design.md`, `implementation.md`.
- Product source, read-only: `/work/resources/inputs/source`.
- A prompt can end with "These declared resources could not be found: <names>".
  A name maps to its file (`analysis` is `analysis.md`). Treat each as `Missing`.
- You do not receive the test repository. Do not judge page objects or test
  conventions.

## Procedure

1. Confirm that `input.json` exists, parses, and holds an identifier and a
   description or `requirements` text. If not, write the `FAILED:` envelope.
2. The work is a `defect` when `bugType` is `Bug`. Otherwise the same ranking
   applies, whether the work is a new feature or an extension. A bug enters at
   Analysis, so it has no Requirements artifact; that is expected.
3. List the browser-observable behavior the ticket states. A later comment that
   changes the description wins. The `requirements` field refines the
   description, so a difference between them is not a conflict. Set aside
   backend-only behavior.
4. Check each claim about the source in `analysis.md`, `design.md`,
   `implementation.md`, and the ticket's `requirements` and `design` fields:
   files, components, routes, labels, and existing behavior. Open the files.
   For a new feature, check that the parents the design extends exist.
5. Estimate the input level for the UI surface (next section).
6. Look for contradictions among the ticket, analysis, and design about
   intended behavior. Apply the ranking below and count only what it cannot
   settle.
7. Fill the rubric, write `confidence.md`, and keep the reasoning on STDOUT.

## Rules of evidence

- **The source decides facts about code.** A claim about the code is checked
  against the source, never ranked. This holds wherever the claim appears,
  including the ticket. A claim that the source contradicts is one `Accuracy`
  deduction, however many documents repeat it, and is not also a `Consistency`
  conflict.
- **A contradicted claim** names something the source does not have, or states the
  opposite of what the source does. Do not deduct for shorthand, a paraphrase, or
  a line number that is slightly off.
- **Absent from the source copy.** A source file, test, or build file that the
  claim names and the copy lacks is contradicted. Agent configuration, `.github`
  files, and documentation may be absent from the copy: treat them as uncheckable.
- **Stay inside the inputs.** Verify their claims. Report a source fact that no
  input mentions under `# Findings`, and score it only when it contradicts a
  claim.
- **Do not judge the fix.** Whether the design would produce the expected outcome
  is the stage's concern, not yours.

## Source ranking

When sources conflict, the higher rank wins. Expected outcomes come only from
rank 1 or rank 2; code supplies mechanics only.

| Rank | `new-feature` and `extension` | `defect` |
|---|---|---|
| 1 | Ticket acceptance criteria, `requirements` field, comments | Expected behavior in the bug, its comments, or a linked requirement |
| 2 | `analysis.md` | Observed defect: reproduction steps and actual result |
| 3 | `design.md`, `implementation.md` | `analysis.md` and design fix intent |
| 4 | Source baseline, for mechanics | Source baseline, for mechanics |

## Input level

The level is the most that the inputs support for the screens the ticket
changes. Count only the items that apply to the feature, and count an item as
met when the sources give it for every new or changed control, not for one:

| Level | Inputs available |
|---|---|
| L1 | Acceptance criteria only |
| L2 | L1, plus screen, navigation, role, labels, controls, and messages |
| L3 | L2, plus stable element IDs, exact data, and database setup rows |

A role that the feature does not change counts as met when the sources say no
role or permission changes. For L3, the design artifacts (or, for UI that already
exists, the source) give each item that applies:

1. Route and navigation, including each frame or tab, and the starting state.
2. Role or account, and any permission that changes what the user sees.
3. A locator for every new or changed control: accessible role and name, label,
   placeholder, or a stable `id` or `data-testid`, in that order of preference.
   For new UI with no user-facing hook, the design names the `id` the code must
   add. A locator that only the ticket names, and the source lacks, does not
   count.
4. Fields and data rules: type, required, defaults, valid and invalid values.
5. Exact message text, where it appears, and its trigger. The requirements own
   the wording; the design must not contradict it.
6. Database rows the feature reads or writes.
7. Initial state, and when each control is shown, hidden, enabled, or disabled.
8. What shows that an asynchronous action has finished.
9. Dialogs, downloads, and uploads.
10. Anything the test cannot create itself: flags, licences, time-dependent
    values.
11. The requirement or acceptance-criterion ID behind each control, message,
    and rule. Citing a section and item counts as met. An ID scheme that the
    ticket's requirements do not use does not.

Items 1 to 6 apply to every new or changed screen. Count items 4 and 6 only when
the feature has fields or database rows.

## Rubric

Score each component from evidence and round to an integer. The anchors are
guides: when the evidence sits between two, score between them in steps of 5.
The `Total` is the `Confidence`.

| Component | Max | Scoring |
|---|---|---|
| Requirements | 35 | 35 when every browser-observable behavior is a checkable outcome (for a defect: expected behavior, reproduction steps, and actual result). 25 when most are, and some outcomes, messages, or values are vague or missing. 15 when only a problem or goal is stated and the outcomes must be inferred. 5 when nothing testable is stated. A defect whose expected behavior is only implied by its title or steps scores at most 25; one with none in the ticket or its comments scores at most 20. A ticket with no browser-observable behavior, clearly described, scores 35. |
| Accuracy | 25 | Start at 25. Deduct 5 for each contradicted claim (see the rules of evidence), to a floor of 0. Do not deduct for a claim you could not check; list it under `# Limitations`. For a feature with no UI surface, score 25 when the conclusion is supported. |
| UI mechanics | 25 | 25 for L3, 15 for L2, 8 for L1, 0 when the UI surface cannot be identified. Score 25 when the ticket has no UI surface and that is clear. |
| Consistency | 15 | Start at 15. Deduct 5 for each conflict about intended behavior that the ranking cannot settle, 5 for each input marked `Missing` (including `analysis.md`, `design.md`, or `implementation.md`), and 5 when `design.md` or `implementation.md` was built for a different requirements set than the ticket's (different items, labels, or IDs), because the mechanics they supply may not match. Floor 0. |

## Do not

- Do not rate the idea's value, size, or risk.
- Do not state a threshold, describe score bands, or reason about what score lets
  the stage proceed.
- Do not write test cases or code, even as examples.
- Do not read a feature branch, generated code, or uncommitted changes.
