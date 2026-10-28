# Test code contract

`testcode.md` is a complete, reviewable handoff of generated Java Playwright
test files and suite additions. It is not a source checkout.

## Status envelope

The first three non-empty lines are:

```markdown
# Status
STATUS_CODE:MESSAGE
Confidence: <integer>%
```

Allowed codes are `SUCCESS`, `NO_UI_SURFACE`, `PLAN_INVALID`, and
`GENERATION_FAILED`. Failure output contains only the envelope and a concise
diagnostic. `NO_UI_SURFACE` contains no file blocks or suite entries.

## Required sections

For a successful plan, include `## Inputs`, `## Files`, `## Suite entries`,
`## Identity map`, and `## Verification notes`, in that order.

Each file block declares one repository-relative path and its complete contents.
Allowed paths are `src/main/java/pages/`, `src/main/java/models/`,
`src/test/java/tests/`, `src/test/java/assertion/`,
`src/test/java/data/`, and `src/test/resources/TestData/`. Do not overwrite
base classes, configuration, or build files.

Suite entries must name `smoketest.xml`, `regressiontest.xml`, or `testNg.xml`.
Each planned identity has exactly one generated Java method, one identity-map row,
and a matching fully qualified class entry. Each method has a Javadoc comment
quoting its Gherkin `Scenario:` line and `Trace:` IDs. Reject duplicate paths,
entries, identities, or map rows, absolute paths, and language/extension
mismatches.

Never guess a locator the plan marks `UNRESOLVED`. Declare it once in its page
object as a constant whose value is `UNRESOLVED:<element name>`, and list it
under Verification notes.

Verification notes must include compile and suite commands, expected
`target/surefire-reports/TEST-*.xml` output, assumptions, and unresolved
locators. Do not claim a command ran unless it actually ran.
