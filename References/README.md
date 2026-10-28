# References — Folder Context

> `References` is a folder holding five independent, unbuilt reference projects that together
> describe an AI-assisted software delivery lifecycle. `SDLC-Orchestration` runs a five-phase
> Kestra pipeline. `AIAgents` supplies the containerized agent definitions each phase invokes.
> `SDLC-Artifacts` stores every artifact the phases emit. `Test Repository` is a Java Playwright
> automation codebase used as a target for generated code and tests. `Manual Process` is a
> separate, non-pipeline QA workflow that turns a Jira ticket into Xray test cases through two
> Atlassian Forge apps. `References` produces no combined build output; each subfolder builds,
> deploys, and runs on its own.

| Field | Value |
| --- | --- |
| Analysis root | `References/` |
| Analyzed on | `2026-09-25` |
| Branch / commit | `[unknown]` — `References/` is not a Git repository and no Git metadata was read |
| Shape | Folder containing 5 independent projects, no shared manifest or build |
| Primary languages | `Java 21`, `JavaScript` (ES modules and Node), `JSX` (React 18), `YAML`, `Markdown` |
| Runtimes | Kestra `v1.3.31`, Atlassian Forge `nodejs22.x`, Node.js `>=18.0.0`, JVM `21` |
| Build / package managers | `Maven`, `npm`, `forge` CLI, `docker compose` / `podman-compose` |
| Paths in this document | Relative to `References/` |

## How to use this document

- Treat this document as a map of `References/`, not as a replacement for each subfolder's own `README.md`.
- Read the subfolder `README.md` named in each section before changing anything inside that subfolder.
- If a subfolder's code contradicts this document, the code wins. Report the contradiction.
- Status markers: `[from-config]` read from a manifest or CI file; `[from-docs]` claimed in project documentation and not confirmed against code; `[inferred]` deduced from structure and not executed; `[unknown]` open question.
- No command in this document was executed. Treat every command as unvalidated.

## Analysis method and its limits

This summary was produced by reading `README.md` files only, at the explicit request of the requester.

- No source file, manifest, workflow definition, or flow definition under `References/` was opened.
- Every factual claim below therefore carries `[from-docs]` at best, even where the source `README.md` labelled a claim `[from-config]`.
- Claims marked `[inferred]` were deduced from the folder listing or from agreement between two separate `README.md` files.
- Command tables reproduce commands as documented. No command was traced back to a `package.json`, `pom.xml`, or workflow file during this analysis.
- A reader needing config-level certainty must verify against the named manifest before acting.

## Subfolder map

| Path | Contains | Role |
| --- | --- | --- |
| `AIAgents/` | Agent packs, manifest schema, instruction and skill Markdown | Defines what each pipeline phase's AI agent does `[from-docs]` |
| `SDLC-Orchestration/` | Kestra flows, Dockerfile, compose files, helper scripts | Runs the five-phase pipeline and calls the agent packs `[from-docs]` |
| `SDLC-Artifacts/` | `Artifacts/` tree plus reusable GitHub Actions workflows | Persists every artifact a pipeline execution produces `[from-docs]` |
| `Test Repository/` | Maven project `Compass-Automation`, TestNG suites, page objects | Example target codebase the pipeline reads and writes `[inferred]` |
| `Manual Process/` | Two Atlassian Forge apps plus MatchaAI prompt Markdown | Standalone Jira-to-Xray manual test case workflow `[from-docs]` |

`Manual Process/` is the only subfolder that does not participate in the Kestra pipeline `[inferred]`.

## Cross-project relationships

The automated pipeline spans three subfolders that name each other explicitly.

1. `SDLC-Orchestration/` receives a `ticketId` and starts the flow `ai-dlc/master-sdlc` `[from-docs]`.
2. `SDLC-Orchestration/` sparse-checks out the matching agent subfolder from `AIAgents` and runs it in a copilotCLI container `[from-docs]`.
3. The agent pack reads inputs from `/work/resources/...` and writes a Markdown artifact to `/work/resources/output/` `[from-docs]`.
4. `SDLC-Orchestration/` commits that artifact into `SDLC-Artifacts` under `Artifacts/<TICKET>/<EXEC_ID>/` `[from-docs]`.
5. The next phase fetches the prior artifact from `SDLC-Artifacts` and repeats `[from-docs]`.

Boundaries that must not be crossed:

- Agent packs never orchestrate; sequencing lives only in `SDLC-Orchestration/_flows/` `[inferred]`.
- Flows never embed agent prompt text; flows reference an agent folder name `[from-docs]`.
- `Manual Process/` never writes to `SDLC-Artifacts`; `Manual Process/` writes to Xray Cloud `[inferred]`.

## Phase-to-agent mapping

| Pipeline phase | Agent pack in `AIAgents/` | Output artifact |
| --- | --- | --- |
| `JiraAnalysis` | `jira-reviewer` | `analysis.md` `[from-docs]` |
| `Design` | `solution-designer` | `design.md` and `implementation.md` `[from-docs]` |
| `CodeGeneration` | `code-generator` | `code.md` plus source edits `[from-docs]` |
| `TestGeneration` | `code-generator` | `[unknown]` — artifact name not stated in any `README.md` |
| `Review` | `[unknown]` — no agent listed | Pull request on the feature branch `[from-docs]` |

`ping-test` maps to no phase; `ping-test` is a smoke-test pack that returns exactly `pong` `[from-docs]`.

## AIAgents

Source of truth: [`AIAgents/README.md`](./AIAgents/README.md).

### AIAgents purpose

- Stores reusable, project-agnostic agent configurations: agent definitions, instructions, and skills `[from-docs]`.
- Excludes repository-specific agent configuration and personal MCP settings `[from-docs]`.
- Contains four agent packs: `jira-reviewer/`, `solution-designer/`, `code-generator/`, `ping-test/` `[from-docs]`.

### AIAgents pack structure

| Path | Required | Purpose |
| --- | --- | --- |
| `<agent-name>/manifest.json` | Yes | Runtime descriptor resolved by the orchestrator `[from-docs]` |
| `<agent-name>/agents/<agent-name>.agent.md` | Yes | Persona, responsibilities, constraints `[from-docs]` |
| `<agent-name>/instructions/<agent-name>.instructions.md` | No | Workflow and quality bar `[from-docs]` |
| `<agent-name>/instructions/<artifact>-contract.md` | No | Format contract for one input or output artifact `[from-docs]` |
| `<agent-name>/skills/<skill-name>/SKILL.md` | No | Reusable procedure the agent invokes `[from-docs]` |
| `<agent-name>/mcp-config.json` | No | MCP servers the pack depends on `[from-docs]` |

The minimum viable pack is `manifest.json` plus one `agents/*.agent.md`; `ping-test/` is that minimum `[from-docs]`.

### AIAgents manifest rules

- Every manifest validates against `AIAgents/schemas/agent-manifest-v1.schema.json` `[from-docs]`.
- The schema sets `additionalProperties: false` on every object, so an undeclared field fails validation `[from-docs]`.
- `schema_version` must be exactly `"1.0"` `[from-docs]`.
- `agent.name` matches `^[a-z0-9][a-z0-9-_/]*$` and equals the agent folder name `[from-docs]`.
- `files` must list every file in the agent folder and must include `manifest.json` `[from-docs]`.
- All four packs use the container image `ghcr.io/harris-smartworks/copilot-cli:latest` `[from-docs]`.

### AIAgents runtime paths

| Path | Used by | Role |
| --- | --- | --- |
| `/work/resources/Jira/input.json` | `jira-reviewer` | Ticket JSON input `[from-docs]` |
| `/work/resources/inputs/artifacts/<name>.md` | `solution-designer`, `code-generator` | Preferred upstream artifact `[from-docs]` |
| `/work/resources/artifacts/<name>.md` | `solution-designer`, `code-generator` | Fallback artifact location `[from-docs]` |
| `/work/resources/inputs/source` | `solution-designer`, `code-generator` | Target source repository `[from-docs]` |
| `/work/resources/output/<name>.md` | All artifact-producing packs | Final Markdown artifact `[from-docs]` |

### AIAgents output contract

- The first heading of every output file is `# Status`, followed by a `STATUS_CODE:MESSAGE` line `[from-docs]`.
- The Status section includes `Confidence: <percentage>%` `[from-docs]`.
- Orchestration parses the `# Status` heading and fails the phase when the heading is absent, renamed, or demoted `[from-docs]`.
- Output files are overwritten completely on each run and never appended to `[from-docs]`.
- A failure still writes the output file, containing only the `# Status` heading and the status line `[from-docs]`.
- Full reasoning goes to STDOUT and stays out of the output Markdown `[from-docs]`.

Documented status codes: `SUCCESS`, `JSON_NOT_FOUND`, `JSON_MALFORMED` for `jira-reviewer`; `SUCCESS`, `DESIGN_NOT_FOUND`, `IMPLEMENTATION_NOT_FOUND`, `SOURCE_NOT_FOUND` for `code-generator` `[from-docs]`.

## SDLC-Orchestration

Source of truth: [`SDLC-Orchestration/README.md`](./SDLC-Orchestration/README.md).

### Orchestration runtime

- Kestra is pinned to `v1.3.31` by digest in `SDLC-Orchestration/Dockerfile`; the image installs `plugin-docker` and `plugin-script-shell`. It also includes pinned `yq`, `git`, and `jq` `[from-config]`.
- Docker Compose maps host port `8080` by default (overridable with `KESTRA_PORT`) and the UI is at `http://localhost:8080/ui/main`; the REST API is tenant-scoped under `/api/v1/main/` `[from-config]`.
- Prerequisites: Docker or Podman, Git, and a `.env` file based on `SDLC-Orchestration/.env.example` `[from-docs]`.
- Development uses `FLOW_WATCH=true`; Kestra watches `_flows/` and reloads saved flow files. Flow filenames must begin with `main_` so the watcher assigns them to the `main` tenant `[from-config]`.
- Production uses `FLOW_WATCH=false`; `system.gitops-sync` deploys flows from the checked-out branch after validation and waits for in-flight pipeline executions to drain. Deploy the sync flow initially with `scripts/deploy/update.sh`; subsequent flow changes arrive through Git `[from-config]`.

### Orchestration flow layout

| Path (under `SDLC-Orchestration/`) | Contains |
| --- | --- |
| `_flows/main_master.yml` | The only API entry point; initializes the run and dispatches ready stages until the graph is complete `[from-config]` |
| `_flows/stages/` | Eleven stage flows, each defining its resources, deliverables, and `nextStages` `[from-config]` |
| `_flows/orchestration/` | Initializes executions, resolves ready stages, and wraps stage execution `[from-docs]` |
| `_flows/util/agents/` | Runs containerized agents and confidence checks `[from-config]` |
| `_flows/util/artifacts/` | Prepares, fetches, inherits, and publishes execution workspaces `[from-config]` |
| `_flows/util/git/`, `_flows/util/jira/` | Source-repository operations and Jira integrations `[from-config]` |
| `_flows/maintenance/` | Cleans up finished executions and stale working directories `[from-config]` |
| `_config/` | Kestra server settings, project configuration, and ticket-type-to-start-stage mapping `[from-config]` |
| `scripts/dev/`, `scripts/deploy/` | Local development helpers and server deployment scripts `[from-docs]` |

To run locally, create `.env` from `.env.example`, run `.\scripts\dev\setup.ps1`, then `docker compose build` and `docker compose up -d`. On Podman, use `podman-compose -f podman-compose.yml`; the production setup deploys flows with `scripts/deploy/update.sh` `[from-docs]`.

### Lifecycle graph and current stage status

The graph is compiled from stage `nextStages`, `resources`, and `deliverables`; it is not a fixed list of phases. A stage can start after multiple upstream stages have completed, and parallel ready stages are dispatched together `[from-config]`.

| Lifecycle path | Stages |
| --- | --- |
| Idea tickets | `Ideation` → `Requirements` → `Analysis` → `Design` → `FirstSecurityReview` |
| Other tickets | `Requirements` → `Analysis` → `Design` → `FirstSecurityReview` |
| Bug tickets | `Analysis` → `Design` → `FirstSecurityReview` |
| Parallel stage 6 | `CodeGeneration` and `TestCaseGeneration` |
| Converged stage 7 | `SecondSecurityReview` |
| Parallel stage 8 | `Review` and `Verification` |
| Final stage | `Release` |

The stage names and paths are defined in `_flows/stages/`; stage starts by ticket type are configured in `_config/pipeline.yml` `[from-config]`. `Ideation`, `Requirements`, both security reviews, `TestCaseGeneration`, `Verification`, and `Release` are currently marked as stubs; `Analysis`, `Design`, `CodeGeneration`, and `Review` are not `[from-config]`.

### Triggering a pipeline run

The pipeline endpoint is `POST /api/v1/main/executions/ai-dlc/master-sdlc`, using HTTP Basic authentication and `multipart/form-data` `[from-docs]`.

| Input | Default | Description |
| --- | --- | --- |
| `ticketId` | required | Jira ticket key, for example `ABC-123` `[from-config]` |
| `requestorEmail` | `""` | Requestor for traceability and outcome notification `[from-config]` |
| `initialStage` | `[]` | Stage(s) to start at; if empty, `Idea` starts at `Ideation`, `Bug` at `Analysis`, and other ticket types at `Requirements` `[from-config]` |
| `finalStages` | `["Release"]` | Last stage(s) to run; naming one stage of a parallel pair does not start its sibling `[from-config]` |
| `previousExecutionPath` | `""` | Earlier artifact execution to copy into this run `[from-config]` |
| `skipConfidenceCheck` | `"First"` | `False` skips none, `First` skips the starting stage, and `True` skips all stage confidence checks `[from-config]` |
| `dryRun` | `false` | Suppresses external notifications and Jira updates; it does not prevent branch pushes or artifact persistence `[from-config]` |
| `humanGatedStages` | `[]` | Stages to pause after for human approval `[from-config]` |

`initialStage` and `finalStages` accept `Ideation`, `Requirements`, `Analysis`, `Design`, `FirstSecurityReview`, `CodeGeneration`, `TestCaseGeneration`, `SecondSecurityReview`, `Review`, `Verification`, and `Release`; `humanGatedStages` accepts the same values except `Release` `[from-config]`.

### Artifact lifecycle

Each run is stored in `SDLC-Artifacts` under `Artifacts/<TICKET>/<EXEC_ID>/`. Stages fetch their own workspace, run their work, and publish their stage directory; a `previousExecutionPath` is copied into the new run rather than reused `[from-docs]`. The master flow compiles stage records into the execution manifest at completion `[from-config]`.

## SDLC-Artifacts

Source of truth: [`SDLC-Artifacts/README.md`](./SDLC-Artifacts/README.md).

### Artifact storage convention

- Artifacts live under `Artifacts/<JIRA_ID>/<YYYYMMDD_HH24MISS>/` `[from-docs]`.
- A documented example path is `Artifacts/SDLC-123/20260619_164300/` `[from-docs]`.
- Every execution directory must contain a `manifest.json` `[from-docs]`.
- Manifest details and evolution guidance live in `SDLC-Artifacts/Artifacts/README.md` `[from-docs]`.

### Artifact reusable workflows

| Workflow file (under `SDLC-Artifacts/.github/workflows/`) | Effect | Key outputs |
| --- | --- | --- |
| `prepare-execution-directory.yml` | Creates the execution folder and ensures a manifest exists; internal helper | `manifest_path` `[from-docs]` |
| `prepare-artifacts.yml` | Prepares directory and manifest without committing | `currdate_string`, `artifact_path`, `manifest_path` `[from-docs]` |
| `persist-artifacts.yml` | Commits and pushes changes under `Artifacts/` | `changed`, `commit_sha` `[from-docs]` |

Rules:

- `prepare-execution-directory.yml` requires at least one of `jira_id` or `artifact_path` `[from-docs]`.
- When only `jira_id` is supplied, the workflow generates `currdate_string` and resolves `artifact_path` to `<jira_id>/<currdate_string>` `[from-docs]`.
- Only `persist-artifacts.yml` commits or pushes; the prepare workflows never do `[from-docs]`.
- `persist-artifacts.yml` defaults `commit_message` to `chore(artifacts): persist workflow artifacts` `[from-docs]`.
- Pass `artifact_bundle_name` to download an uploaded workflow artifact before the commit step `[from-docs]`.

The documented composition order is prepare, then generate, then persist `[from-docs]`.

## Test Repository

Source of truth: [`Test Repository/README.md`](<./Test Repository/README.md>).

### Test Repository identity

- The Maven coordinates are `com.smartwork.compass:Compass-Automation:0.0.1-SNAPSHOT` `[from-docs]`.
- The project is a single Maven module with `24` TestNG test classes and `30` page objects `[from-docs]`.
- The project drives the Compass web application and contains no Compass application source code `[from-docs]`.
- The project ships no CI pipeline definition `[from-docs]`.

### Test Repository stack

| Layer | Technology | Version |
| --- | --- | --- |
| Language | Java | `21` `[from-docs]` |
| Browser automation | `com.microsoft.playwright:playwright` | `1.57.0` `[from-docs]` |
| Test runner | `org.testng:testng` | `7.12.0` `[from-docs]` |
| Reporting | `com.aventstack:extentreports` | `5.1.2` `[from-docs]` |
| JSON binding | `com.fasterxml.jackson.core:jackson-databind` | `2.18.6` `[from-docs]` |
| Databases | `org.postgresql:postgresql` `42.7.10`, `com.oracle.database.jdbc:ojdbc11` `23.26.2.0.0` | see versions in cell `[from-docs]` |

### Test Repository commands

| Purpose | Command | Documented in | Status |
| --- | --- | --- | --- |
| Install Playwright browsers | `mvn exec:java -D"exec.args=install"` | `Test Repository/README.md` | `[from-docs]` |
| Run the configured suite | `mvn test` | `Test Repository/README.md` | `[from-docs]` |
| Run the smoke suite | `mvn test -DsuiteXmlFile=smoketest.xml` | `Test Repository/README.md` | `[from-docs]` |
| Run the regression suite | `mvn test -DsuiteXmlFile=regressiontest.xml` | `Test Repository/README.md` | `[from-docs]` |
| Run one test class | `mvn test -Dtest=EditMeterTest` | `Test Repository/README.md` | `[from-docs]` |
| Select the Oracle environment | `mvn test -Dactive.env=env2` | `Test Repository/README.md` | `[from-docs]` |

Ordering requirement: install Playwright browsers once before the first `mvn test` run `[from-docs]`.

Generated output that must never be hand-edited: `Test Repository/test-output/` and `Test Repository/target/` `[from-docs]`.

## Manual Process

Source of truth: [`Manual Process/README.md`](<./Manual Process/README.md>).

### Manual Process components

| Path (under `Manual Process/`) | Contains | Workflow step |
| --- | --- | --- |
| `Test Case Generator/Forge-Test-Case-Generator/` | Forge app: Node backend plus React Custom UI | Collect ticket data, call the AI service, review, approve `[from-docs]` |
| `Forge-Xray-Import-MCP/` | Forge app: web trigger handler and Jest tests | Create Tests in Xray Cloud and read them back `[from-docs]` |
| `Test Generator Agents/Plan/` | `Requirement-Generator.md`, `Test-Step-Generator.md` | Prompt definitions for the two MatchaAI missions `[from-docs]` |

### Manual Process flow

1. A QA analyst opens the `Test Case Generator` issue panel on a Jira ticket `[from-docs]`.
2. The generator collects ticket and child-ticket data and builds an AI payload `[from-docs]`.
3. The generator calls the MatchaAI Requirement Generator mission, then the Steps Generator mission `[from-docs]`.
4. The React Review panel lets the analyst select, edit, reorder, and approve test cases `[from-docs]`.
5. The generator POSTs approved Xray JSON to the importer web trigger URL `[from-docs]`.
6. The importer creates each Test sequentially through the Xray GraphQL API `[from-docs]`.
7. The generator links each created Test to its requirements and Test Executions, then comments on the ticket `[from-docs]`.

### Manual Process key commands

| Purpose | Command | Run from | Status |
| --- | --- | --- | --- |
| Lint the generator backend | `npm run lint` | `Test Case Generator/Forge-Test-Case-Generator/` | `[from-docs]` |
| Build the generator UI bundle | `npm run build` | `Test Case Generator/Forge-Test-Case-Generator/static/` | `[from-docs]` |
| Run the generator UI tests | `npm test -- --watchAll=false` | `Test Case Generator/Forge-Test-Case-Generator/static/` | `[from-docs]` |
| Run the importer tests | `npm test` | `Forge-Xray-Import-MCP/` | `[from-docs]` |
| Deploy either Forge app | `npm run deploy` | The app root | `[from-docs]` |
| Create the importer web trigger | `forge webtrigger create --functionKey xray-import-trigger` | `Forge-Xray-Import-MCP/` | `[from-docs]` |

Ordering requirements: `forge login` precedes any `forge` subcommand; the generator UI build precedes `forge deploy`; the importer deploys and its web trigger exists before the generator is configured `[from-docs]`.

## External integrations across the folder

| Integration | Used by | Config or variable names |
| --- | --- | --- |
| Jira | `AIAgents/jira-reviewer`, `SDLC-Orchestration`, both Forge apps | Forge scopes; `enableJiraUpdates` flow input `[from-docs]` |
| GitHub Container Registry | `SDLC-Orchestration`, all agent packs | Image `ghcr.io/harris-smartworks/copilot-cli:latest` `[from-docs]` |
| NVD MCP server | `AIAgents/jira-reviewer` | `io.github.pipeworx-io/nvd` over streamable-http `[from-docs]` |
| MatchaAI completions API | `Manual Process` generator | `ai_endpoint_url`, `requirement_mission_id`, `steps_mission_id`, `matcha_api_key` `[from-docs]` |
| Xray Cloud GraphQL | `Manual Process` importer | `XRAY_CLIENT_ID`, `XRAY_CLIENT_SECRET`, `XRAY_REGION`, `JIRA_PROJECT_KEY`, `XRAY_LINK_TYPE_NAME` `[from-docs]` |
| PostgreSQL | `SDLC-Orchestration`, `Test Repository` | `docker-compose.postgres.yml`; `<env>.db.*` keys in `config.properties` `[from-docs]` |

Only variable names belong in documentation. Never commit a value for any variable named above.

## Conventions shared across the folder

- Update the affected subfolder's `README.md` in the same change set as a content change `[from-docs]`.
- Keep each `AIAgents` manifest `files` list synchronized with that agent folder's actual contents `[from-docs]`.
- Name Kestra flow files with the `main_` prefix, or Kestra never loads them `[from-docs]`.
- Keep prompt output format and the parser that consumes it in sync within `Manual Process` `[from-docs]`.
- Store secrets through the platform's secret mechanism, never in a committed file `[from-docs]`.
- Treat generated directories as read-only: `Test Repository/target/`, `Test Repository/test-output/`, and the generator's `static/build/` `[from-docs]`.

## Gotchas

- `AIAgents` manifests fail validation on any undeclared field, because the schema sets `additionalProperties: false` `[from-docs]`.
- A phase fails when an agent output file lacks a top-level `# Status` heading `[from-docs]`.
- A Kestra flow file without the `main_` filename prefix is silently ignored `[from-docs]`.
- `Forge-Xray-Import-MCP/README.md` embeds a live web trigger URL including its secret path segment; treat that file as sensitive `[from-docs]`.
- Parallel Xray imports trip rate limits, so the importer creates Tests sequentially `[from-docs]`.
- The Forge service account needs `Browse Projects` and `Edit Issues` in every involved project, or cross-project links fail `[from-docs]`.
- `Test Repository` requires a reachable Compass environment and database, because setup logs in through the UI and opens a JDBC connection `[from-docs]`.

## Where to change X

| Task | Start in | Then read |
| --- | --- | --- |
| Change what an agent produces | `AIAgents/<agent>/instructions/` | `AIAgents/README.md` output contract section |
| Add a new agent pack | `AIAgents/<new-agent>/manifest.json` | `AIAgents/schemas/agent-manifest-v1.schema.json` |
| Change phase sequencing or gating | `SDLC-Orchestration/_flows/main_master.yml` | `SDLC-Orchestration/README.md` trigger inputs |
| Change where artifacts are stored | `SDLC-Artifacts/.github/workflows/` | `SDLC-Artifacts/Artifacts/README.md` |
| Add a UI test scenario | `Test Repository/src/test/java/tests/` | `Test Repository/README.md` directory map |
| Change generated test case wording | `Manual Process/Test Generator Agents/Plan/` | `Manual Process/README.md` format contract section |

## Glossary

| Term | Definition |
| --- | --- |
| AI-DLC | The five-phase AI-assisted delivery lifecycle run by `SDLC-Orchestration` `[from-docs]` |
| Agent pack | A self-contained folder in `AIAgents` holding a manifest plus agent definition `[from-docs]` |
| copilotCLI container | The container image that executes an agent pack during a phase `[from-docs]` |
| Execution path | `Artifacts/<TICKET>/<EXEC_ID>/`, the folder holding one run's artifacts `[from-docs]` |
| Forge | Atlassian's hosted app platform used by both `Manual Process` apps `[from-docs]` |
| Kestra | The workflow engine that runs the pipeline flows `[from-docs]` |
| MatchaAI | The external AI service that drafts requirements and test steps `[from-docs]` |
| MCP | Model Context Protocol, the mechanism an agent pack uses to reach external tools `[from-docs]` |
| Human-gated phase | A phase after which the pipeline pauses for approval `[from-docs]` |
| Xray | The Jira test-management product the importer writes Test issues into `[from-docs]` |

## Further reading

- [`AIAgents/README.md`](./AIAgents/README.md): pack structure, manifest schema, runtime paths, and per-agent behavior.
- [`SDLC-Orchestration/README.md`](./SDLC-Orchestration/README.md): Kestra setup, flow layout, API trigger inputs, and artifact cycle.
- [`SDLC-Artifacts/README.md`](./SDLC-Artifacts/README.md): storage convention and the three reusable workflows with examples.
- [`Test Repository/README.md`](<./Test Repository/README.md>): Playwright and TestNG architecture, configuration keys, and commands.
- [`Manual Process/README.md`](<./Manual Process/README.md>): the two Forge apps, the prompt contract, and end-to-end Jira-to-Xray flow.

## Unknowns and unverified claims

| Claim or question | Status | How to resolve |
| --- | --- | --- |
| Every claim in this document derives from `README.md` files only | `[from-docs]` | Re-run the analysis with code and manifest reading enabled |
| Which agent, if any, performs the `Review` phase | `[unknown]` | Read `SDLC-Orchestration/_flows/phases/review/` |
| Artifact filename produced by the `TestGeneration` phase | `[unknown]` | Read `SDLC-Orchestration/_flows/phases/test-generation/` |
| Whether each subfolder is an independent Git repository | `[unknown]` | Check for a `.git` directory in each subfolder |
| Branch and commit each subfolder was analyzed at | `[unknown]` | Run `git -C <subfolder> rev-parse HEAD` in each repository |
| Whether `Manual Process/Test Generator Agents/Code/` is intentionally empty | `[unknown]` | Ask the `Manual Process` maintainer |
| Whether the four `AIAgents` packs are still the complete set | `[from-docs]` | List the top-level folders under `AIAgents/` |
| Whether any documented command still matches its manifest | `[from-docs]` | Compare each command against `package.json`, `pom.xml`, or the workflow file |
