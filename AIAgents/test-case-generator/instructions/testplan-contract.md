# Test plan contract

`testplan.md` is a reviewable UI test-design artifact. It describes proposed
tests and does not register them in Xray.

## Status envelope

The first three non-empty lines are exactly:

```markdown
# Status
STATUS_CODE:MESSAGE
Confidence: <integer>%
```

Allowed codes are `SUCCESS`, `NO_UI_SURFACE`, `INPUT_NOT_FOUND`,
`INPUT_MALFORMED`, and `GENERATION_FAILED`. `SUCCESS` requires one or more
test sections. `NO_UI_SURFACE` is successful only when there are no test
sections and the document contains `## No UI surface`.

## Required body

Sections must occur in this order:

1. `## Scope` with Ticket, Ticket type, Work kind (`new-feature`, `extension`,
   or `defect`), Classification evidence, Source precedence, Design,
   Implementation, and UI surface fields.
2. `## Traceability` with `ID`, `Requirement or acceptance criterion`, and
   `Covered by` columns. Use `REQ-` or `AC-` IDs for intended behavior and
   `DEF-` IDs for observed defect behavior.
3. One `## Test: tests.<Class>.<method>` section per test, or `## No UI surface`.
4. `## Coverage summary`, including `Test count`, `Input levels`, and
   `Unresolved locators`.

Each test record contains:

- `Identity`, `Title`, `Type`, `Trace`, `Scenario`, `Steps`,
  `Expected outcome`, `Coverage rationale`, `Input level`, and
  `Locator evidence`.
- Optional `Preconditions`, `Data`, and `Group`.

`Scenario` is one fenced `gherkin` block with at least one `Given`, `When`, and
`Then` line. `Input level` is `L1`, `L2`, or `L3`. `Locator evidence` names
each needed locator and where it came from: a page object, base-branch source,
or a design artifact. A locator without evidence is `UNRESOLVED` and is also
listed in the coverage summary.

Steps must contain at least one numbered action. Trace must reference an ID in
the traceability table, and at least one traced ID must be `REQ-` or `AC-`,
because a `DEF-` ID cannot define the expected outcome. Preserve the identity
exactly because it is the later Xray and JUnit binding key. `Test count` must
equal the number of test sections.
