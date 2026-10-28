# Test code contract

`testcode.md` is the generated-code handoff paired with `testplan.md`. It is
reviewable Markdown containing complete file contents and the suite changes
needed to execute the planned tests. The later implementation step copies the
declared files into the test repository; this artifact itself is not a source
checkout and must not contain credentials.

## Required status envelope

The first three non-empty lines are:

```markdown
# Status
STATUS_CODE:MESSAGE
Confidence: <integer>%
```

The heading, status line, and confidence range follow the
[test plan contract](./testplan-contract.md). Code-generation status codes
are:

- `SUCCESS` — all planned executable tests have complete file blocks.
- `NO_UI_SURFACE` — no browser tests are appropriate; no file blocks are
  required.
- `PLAN_INVALID` — `testplan.md` cannot be used to generate code.
- `GENERATION_FAILED` — code could not be generated.

Failure outputs contain only the status envelope and a concise diagnostic
paragraph; they must not present partial code as successful output.

## Required structure

After the status envelope:

1. `## Inputs`
2. `## Files`
3. `## Suite entries`
4. `## Identity map`
5. `## Verification notes`

`NO_UI_SURFACE` may contain `## No UI surface` followed by a reason instead of
the four code sections.

### Inputs

```markdown
## Inputs

- Test plan: `testplan.md`
- Source baseline: `baseBranch`
- Repository: `TEST-REPO-READ`
```

### File blocks

Each generated file is declared once and contains its complete contents:

````markdown
## Files

### File: src/test/java/tests/LoginTest.java

```java
package tests;

public class LoginTest extends BaseTest {
    // complete generated file content
}
```
````

The path is repository-relative and must use `/` separators in the artifact.
Allowed generated locations are `src/main/java/pages/`,
`src/main/java/models/`, `src/test/java/tests/`,
`src/test/java/assertion/`, `src/test/java/data/`, and
`src/test/resources/TestData/`. Do not overwrite `base/`, configuration, or
build files unless a later approved contract explicitly permits it.

Each test identity in `testplan.md` must have one corresponding test method in
the declared Java file. The method name and class name must follow
[the naming rules](./testcode-naming-rules.md). Each test method has a Javadoc
comment that quotes its Gherkin `Scenario:` line and its `Trace` IDs, so a reader
can map code to the plan without opening the artifact:

```java
/**
 * Scenario: Valid users can sign in
 * Trace: REQ-1, AC-1
 */
@Test(groups = {"smoke"})
public void testValidLogin() { ... }
```

A locator recorded as `UNRESOLVED` in the plan must not be guessed. Declare it once,
in its page object, as a constant whose value is `UNRESOLVED:<element name>`. The
file still compiles, the test fails at the unresolved element with a clear message,
and the fix is a one-line change. List every such constant under Verification notes.

### Suite entries

```markdown
## Suite entries

### File: regressiontest.xml

```xml
<class name="tests.LoginTest"/>
```
```

Entries are additive and must identify the suite (`smoketest.xml`,
`regressiontest.xml`, or `testNg.xml`) and the fully qualified class. A class
must not be listed twice in the same suite.

### Identity map

```markdown
## Identity map

| Test identity | Source file | Method | Group |
|---|---|---|---|
| tests.LoginTest.testValidLogin | src/test/java/tests/LoginTest.java | testValidLogin | smoke |
```

The identity map is the handoff used by Xray registration and result binding.
Every row must match a test-plan identity and a generated method exactly.

### Verification notes

```markdown
## Verification notes

- Compile command: `mvn -q -DskipTests compile test-compile`
- Suite command: `mvn test -DsuiteXmlFile=smoketest.xml`
- Expected JUnit output: `target/surefire-reports/TEST-*.xml`
- Assumptions: The existing login page object and environment configuration
  are unchanged.
- Unresolved locators: none
```

Notes must state the commands a reviewer can run and identify any assumption
that prevents a local run. They must not claim a command was run unless it
was actually run by the generation stage.

## Validation rules

- Parse and validate `testplan.md` before accepting `testcode.md`.
- Reject duplicate file paths, duplicate suite entries, and duplicate identity
  map rows.
- Reject code blocks whose declared language does not match the file extension.
- Reject a missing identity-map row, Java method, or suite entry for a planned
  test.
- Reject a planned test method without a Javadoc `Scenario:` and `Trace:` line.
- Reject an `UNRESOLVED:` locator constant that is not listed in Verification
  notes, or a locator that the plan marks `UNRESOLVED` but the code gives a
  concrete selector.
- Reject paths outside the allowed locations or any absolute path.
- Treat `NO_UI_SURFACE` as a complete successful artifact with zero files and
  zero suite entries.

