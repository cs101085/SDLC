# Test code naming rules

- Keep generated tests in the existing flat `tests` package.
- Use a PascalCase feature name followed by `Test` for the class, for example
  `LoginTest`.
- Use lowerCamelCase, behavior-oriented method names beginning with `test`, for
  example `testValidLogin`.
- The stable identity is `tests.<ClassName>.<methodName>`.
- A class or method rename changes the identity and invalidates the later Xray
  method-to-key mapping; regenerate the mapping rather than silently aliasing it.
- Use one class for a cohesive feature when that matches the baseline. Avoid
  duplicate class names and duplicate method names within a class.
- Match the baseline's TestNG annotations, groups, page-object calls,
  assertion-helper style, data providers, and formatting.
