# AIAgents project deliverables

This folder contains the project-specific agent packs produced for the AI
testing project. Reference pack conventions and the manifest schema are in
`References/AIAgents/`.

## Packs

- `test-case-generator/` produces the requirements-first `testplan.md` (automated
  and manual tests, with coverage metrics) and repository-compatible
  `testcode.md` artifacts for stage 6b. Its contracts are in `instructions/`.
- `test-case-generator-confidence/` is the stage 6b entry assessor. It rates
  whether the ticket and the Analysis and Design artifacts support writing
  correct UI tests, and writes `confidence.md`.
- `test-case-generator-coverage-confidence/` is the stage 6b exit assessor. It
  recomputes the plan's metrics and rates coverage and plan-to-code consistency.
  It holds copies of the plan and code contracts.

Both assessors share `confidence-contract.md`, and the generator and exit packs
share the three test contracts. Keep each shared file byte-identical across its
copies, and across `SDLC-Orchestration/` for the test contracts (R11).

Pack folders are flat today. The stage flows request agents as
`AIDLC/<Stage>/<agent>`, so migration moves each pack to
`AIDLC/TestCaseGeneration/<agent>/` (placeholder `AGENT-PATH`).
