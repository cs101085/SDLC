# Generated test naming rules

These rules make the fully qualified TestNG method stable across generation,
Xray registration, and JUnit result binding.

## Package decision

Generated tests remain in the existing flat `tests` package. The reference
repository already uses `src/test/java/tests/`, and changing to a namespaced
package would require moving existing classes and changing suite entries.
The flat package is a known global-collision risk, so the pipeline must use
the explicit Xray `test_key` mapping in result XML rather than relying on
Definition matching.

## Class names

- One feature-oriented class per test-plan feature group.
- Use PascalCase and the suffix `Test`, for example `EditMeterTest`.
- Derive the feature name from the ticket/design domain, not from an individual
  data row or environment.
- Do not include ticket keys, timestamps, branch names, or generated ordinals.
- If the name would collide with an existing class, append a stable domain
  qualifier, such as `AdminEditMeterTest`; do not append a numeric suffix.
- A class must remain unchanged when only test data or step wording changes.

## Method names and identity

- Use lower camel case and a verb-led behavior name, for example
  `testEditMeterInsertRecord`.
- Describe one observable behavior; do not include data values, timestamps, or
  randomized text.
- The identity is exactly:

  ```text
  tests.<ClassName>.<methodName>
  ```

- The identity must equal the TestNG class and method emitted by Surefire:
  `classname="tests.<ClassName>"` and `name="<methodName>"`.
- A class rename or method rename changes the identity and therefore changes
  the Xray Definition. It requires regeneration of the Step 6 identity map
  and the Step 7 JUnit `test_key` injection mapping.
- Data-driven rows intentionally share the method identity unless a later
  contract introduces a row-level identity scheme.

## Supporting files

- Page objects use PascalCase plus `Page`, for example `LoginPage`.
- Assertion helpers use PascalCase plus `Assertions`, for example
  `LoginAssertions`.
- JSON data files use lower kebab-free names matching the feature,
  for example `login.json`.
- Suite files retain the repository names `testNg.xml`, `smoketest.xml`, and
  `regressiontest.xml`.

## Stable example

For a feature named “Edit meter”, the generated identity is:

```text
tests.EditMeterTest.testEditMeterInsertRecord
```

The corresponding source and result fields are:

```text
src/test/java/tests/EditMeterTest.java
classname="tests.EditMeterTest"
name="testEditMeterInsertRecord"
```

