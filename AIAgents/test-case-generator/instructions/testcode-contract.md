# Test code contract

`testcode.md` is the generated-code handoff that pairs with `testplan.md`. It is
reviewable Markdown that contains complete file contents and the suite changes
needed to run the planned automated tests. Step 5 copies the declared files into
the test repository. The artifact is not a source checkout and contains no
credentials.

## Status envelope

The first three non-empty lines follow the
[test plan contract](./testplan-contract.md):

```markdown
# Status
STATUS_CODE:MESSAGE
Confidence: <integer>%
```

| Code | Meaning |
|---|---|
| `SUCCESS` | Every planned automated test has a complete file block. |
| `NO_UI_SURFACE` | No browser tests are appropriate. No file blocks are required. |
| `PLAN_INVALID` | `testplan.md` cannot be used to generate code. |
| `GENERATION_FAILED` | Code could not be generated. |

A plan whose tests are all manual is `SUCCESS` with no file blocks, no suite
entries, and an empty identity map. Failure outputs contain only the envelope
and a concise diagnostic. They never present partial code as successful.

## Required structure

After the envelope, a `SUCCESS` output has these sections in order:

1. `## Inputs`
2. `## Files`
3. `## Suite entries`
4. `## Identity map`
5. `## Verification notes`

`NO_UI_SURFACE` has `## No UI surface` and a reason instead.

### Inputs

```markdown
## Inputs

- Test plan: `testplan.md`
- Source baseline: `baseBranch`
- Repository: `TEST-REPO-READ`
```

### Files

Each generated file is declared once with its complete contents:

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

- **Paths.** A path is repository-relative and uses `/`. Allowed locations are
  `src/main/java/pages/`, `src/main/java/models/`, `src/test/java/tests/`,
  `src/test/java/assertion/`, `src/test/java/data/`, and
  `src/test/resources/TestData/`. Never write to `base/`, `config.properties`,
  or build files.
- **New and extended files.** A file is new by default, and its path must not
  exist in `TEST-REPO-READ`. To add to an existing file, such as a page object
  or the test class of an existing feature, put `Mode: extend` on the line
  after the heading. An extended file keeps every line of the original in order
  and only adds lines. The block still holds the complete file.
- **Language.** The fence language matches the file extension.

Each automated test identity in `testplan.md` has one test method in the
declared class. [Naming rules](./testcode-naming-rules.md) define the names. A
Javadoc comment quotes the Gherkin `Scenario:` line and the `Trace` IDs, so a
reader can map code to the plan:

```java
/**
 * Scenario: Valid users can sign in
 * Trace: AC-1
 */
@Test(groups = {"smoke", "regression"})
public void testValidLogin() { ... }
```

Generated code follows the test repository's conventions: the class extends
`BaseTest`, creates page objects in an `initPages` method annotated
`@BeforeMethod`, takes data through `@DataSource` and the `jsonData` provider,
keeps locators in page objects, and keeps assertions in the assertion class.
Every `@Test` carries the plan's `Group`, because the suites select by group.

Never guess a locator that the plan marks `UNRESOLVED`. Declare it once, in its
page object, as a constant whose value is `UNRESOLVED:<element name>`. The file
still compiles, the test fails at that element with a clear message, and the fix
is a one-line change.

### Suite entries

Each entry names a suite file and holds the XML to add:

````markdown
## Suite entries

### File: testNg.xml

```xml
<test name="Login">
    <classes>
        <class name="tests.LoginTest"/>
    </classes>
</test>
```

### File: smoketest.xml

```xml
<class name="tests.LoginTest"/>
```
````

- `testNg.xml` gets a `<test>` block for every generated class.
- `smoketest.xml` and `regressiontest.xml` get a `<class>` line when a method
  of the class belongs to that group. They already filter by group.
- A class appears once per suite. Existing classes that the plan extends are
  already registered and are not listed again.

### Identity map

```markdown
## Identity map

| Test identity | Source file | Method | Group |
|---|---|---|---|
| tests.LoginTest.testValidLogin | src/test/java/tests/LoginTest.java | testValidLogin | smoke, regression |
```

The map covers automated tests only. Every row matches a plan identity and a
generated method exactly. Manual tests have no code; Step 6 keys them by their
`manual.` identity.

### Result binding

Xray binds each JUnit `testcase` to a Generic test by its Definition: the
`classname` and `name` attributes concatenated (F10, F11). The identity
`tests.<ClassName>.<methodName>` is the canonical key for that pair. Kestra
injects the `requirements`, `test_summary`, `test_description`, and `tags`
properties into the JUnit XML from `testplan.md`, joined by identity (Steps 6
and 7). Generated code therefore holds no Jira key, `test_key`, `test_id`, or
Xray annotation, and the test repository needs no Jira awareness.

The separator that Xray inserts between `classname` and `name` is unverified
(V1), and the `name` that Surefire writes for each `@DataSource` row is
unverified (V4). Rows of one method share one identity until Q11 is decided.

### Verification notes

```markdown
## Verification notes

- Compile command: `mvn -q -DskipTests test-compile`
- Class command: `mvn test -Dtest=LoginTest`
- Expected JUnit output: `target/surefire-reports/TEST-*.xml`
- Assumptions: The existing login page object and environment configuration
  are unchanged.
- Unresolved locators: none
```

State the commands a reviewer can run, the assumptions that prevent a local
run, and every `UNRESOLVED:` constant. Do not claim that a command ran unless
the stage ran it. Do not give a suite command that uses `-DsuiteXmlFile`:
`pom.xml` never reads that property, and it names `testng.xml` while the file
is `testNg.xml` (R7).

## Validation rules

- Validate `testplan.md` before accepting `testcode.md`.
- Reject duplicate file paths, suite entries, and identity-map rows.
- Reject a path outside the allowed locations, an absolute path, or a fence
  language that does not match the extension.
- Reject a `new` file whose path exists in `TEST-REPO-READ`, and reject an
  `extend` file that removes or reorders an original line.
- Reject an automated plan identity that lacks a method or an identity-map row,
  or whose class is neither in a `testNg.xml` entry nor already registered there
  in `TEST-REPO-READ`.
- Reject a method without a Javadoc `Scenario:` and `Trace:` line, or without
  the plan's `Group`.
- Reject a `test_key`, `test_id`, or Xray annotation anywhere in the code.
- Reject an `UNRESOLVED:` constant that Verification notes omit, and reject a
  concrete selector for a locator that the plan marks `UNRESOLVED`.
- Treat `NO_UI_SURFACE` as complete with zero files and zero suite entries.
