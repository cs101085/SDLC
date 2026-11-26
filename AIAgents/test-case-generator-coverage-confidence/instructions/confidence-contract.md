# Confidence assessment contract

`confidence.md` is the assessment that a stage's confidence check reads. The
orchestration compares its rating with a threshold that the assessor never
sees. This contract governs both Test Case Generation assessors, and their
copies must stay byte-identical.

## Output

Write `/work/resources/output/confidence.md` and overwrite it completely on
every run. Put reasoning in STDOUT, not in the file. The check flow publishes the
file as
`SDLC-Artifacts/Artifacts/<ticket>/<executionId>/TestCaseGeneration/confidence-<point>.md`,
where `<executionId>` is the run's `YYYYMMDD_HHMMSS` and `<point>` is `inputs` for
the entry check and `coverage` for the exit check. Write nowhere else.

## Status envelope

The first three non-empty lines are:

```markdown
# Status
SUCCESS:<one short sentence>
Confidence: <integer>%
```

For example, `SUCCESS:Assessment completed.` has no space after the colon.

- `SUCCESS:` means the assessment was carried out, however poor the material. A
  low rating is a successful assessment.
- `FAILED:<reason>` means the assessment could not be carried out, such as an
  absent or unreadable ticket. Write `Confidence: 0%` and no other section.
- The orchestration takes the first `SUCCESS:` or `FAILED:` line and the first
  `Confidence:` line in the file. Write each once, in the envelope, and start no
  later line with `Confidence:`.
- `Confidence` is the `Total` of the rubric in `# Rating`, an integer from 0
  through 100.

## Sections

A `SUCCESS` output has these sections after the envelope, in this order:

1. `# Assessment`: what the check judges and why, in two to four sentences.
2. `# Inputs`: one bullet for each input that the assessor's instructions list,
   with its name, `Found` or `Missing`, and `usable` or the reason it is not.
3. `# Rating`: a table with the columns `Component`, `Max`, `Score`, and
   `Reason`, one row for each rubric component, and a final `Total` row.
4. `# Findings`: evidence bullets. Each names the file and the section or line
   that supports it.
5. `# Limitations`: what could not be checked and was left out of the rating.

## Rules

- Never state or guess a threshold, and never say whether the stage should
  proceed. The rating is the whole answer.
- Score from evidence in the inputs. Do not raise a score for effort, or for
  material that the check was not given.
- An input that the prompt lists as not found is `Missing` under `# Inputs` and
  lowers the component it would have supported.
- Do not modify any input. Write nothing outside
  `/work/resources/output/confidence.md`.
