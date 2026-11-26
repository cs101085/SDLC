# Test code naming rules

These rules keep the fully qualified TestNG method stable across generation,
Xray registration, and JUnit result binding. They follow the conventions in the
test repository README.

## Identity

An automated test's identity is exactly:

```text
tests.<ClassName>.<methodName>
```

It equals the class and method that Surefire writes: `classname="tests.<ClassName>"`
and `name="<methodName>"`. Xray concatenates those two attributes into the
Definition that binds results (F10, F11).

A manual test has no class. Its identity is:

```text
manual.<Feature>.<behavior>
```

`<Feature>` is PascalCase and `<behavior>` is a verb-led lowerCamelCase name,
for example `manual.Login.verifyBrandColors`. The `manual.` prefix keeps the
two namespaces apart.

A class or method rename changes the identity and therefore the Xray
Definition. Regenerate the identity map and the Step 7 property injection
rather than aliasing the old name.

## Package

Generated tests stay in the flat `tests` package, as the repository does.
Namespacing the package would move existing classes and change suite entries.
The flat package can collide with another project's tests. Xray searches other
Jira projects before it creates a test, so the Step 7 read-back fails any result
that binds outside the target project (R1).

## Classes and methods

- **Class.** Use PascalCase and the suffix `Test`, such as `EditMeterTest`, one
  class per feature. Name it from the ticket or design domain, not from a data
  row, an environment, a ticket key, a date, or a branch.
- **Existing feature.** When `TEST-REPO-READ` already has a test class for the
  feature, add methods to that class with `Mode: extend`. Create a new class
  only for a feature with none. Never append a numeric suffix to avoid a
  collision.
- **Method.** Start with `test` and describe one observable behavior, such as
  `testEditMeterInsertRecord`. Do not include data values or random text. A
  method name is unique within its class and does not reuse an existing method.
- **Stability.** A name does not change when only test data or step wording
  changes.
- **Data-driven rows.** Rows of one `@DataSource` method share its identity
  until Q11 is decided.

## Supporting files

| File | Name | Example |
|---|---|---|
| Page object | `<Feature>` | `EditMeter` |
| Assertion class | `<Feature>Assertion` | `EditMeterAssertion` |
| Model | `<Feature>Data` | `LocationSummaryData` |
| Test data | `<Feature>Data.json` | `HeatMapData.json` |
| Suite files | Existing names | `testNg.xml`, `smoketest.xml`, `regressiontest.xml` |

Match the baseline's TestNG annotations, groups, page-object calls, assertion
style, data providers, and formatting.

## Stable example

For a feature named "Edit meter", the identity is:

```text
tests.EditMeterTest.testEditMeterInsertRecord
```

The source and result fields are:

```text
src/test/java/tests/EditMeterTest.java
classname="tests.EditMeterTest"
name="testEditMeterInsertRecord"
```
