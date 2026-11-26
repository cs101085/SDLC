# References — Folder Context

> `References` is a folder holding six independent reference projects that together
> describe an AI-assisted software delivery lifecycle. `SDLC-Orchestration` runs an eleven-stage
> Kestra pipeline plus an execution-record service. `AIAgents` supplies the containerized agent
> definitions the stages invoke. `SDLC-Artifacts` stores every artifact the stages emit.
> `metersense-release-5.1` is the Compass (MeterSense) product source tree the pipeline reads and
> changes. `Test Repository` is a Java Playwright automation codebase that drives the Compass web
> application. `Manual Process` is a separate, non-pipeline QA workflow that turns a Jira ticket
> into Xray test cases through two Atlassian Forge apps. `References` produces no combined build
> output; each subfolder builds, deploys, and runs on its own.

| Field | Value |
| --- | --- |
| Analysis root | `References/` |
| Analyzed on | `2026-11-24` |
| Branch / commit | `main` @ `946c0003729a` for the enclosing `SDLC` repository `[from-config]`; the two `Manual Process` Forge apps carry their own `.git` directories; uncommitted changes `[unknown]` |
| Shape | Folder containing 6 independent projects, no shared manifest or build |
| Primary languages | `Java 21`, `JavaScript` (ES modules and Node), `JSX` (React 18), `Bash`, `PowerShell`, `YAML`, `Markdown` |
| Runtimes | Kestra `v1.3.31`, Spring Boot `3.5.16`, PostgreSQL `15`, Atlassian Forge `nodejs22.x`, Node.js `>=18.0.0`, JVM `21` |
| Build / package managers | `Maven`, `Ant`, `npm`, `forge` CLI, `Flyway`, `docker compose` / `podman-compose` |
| Paths in this document | Relative to `References/` |

## How to use this document

- Treat this document as a map of `References/`; read the subfolder `README.md` before changing anything inside that subfolder.
- If a subfolder's code contradicts this document, the code wins. Report the contradiction.
- Status markers: `[from-config]` read from a manifest or CI file; `[from-docs]` claimed in project documentation and not confirmed against code; `[inferred]` deduced from structure and not executed; `[unknown]` open question.
- No command in this document was executed. Treat every command as unvalidated.

## Analysis method and its limits

- This summary was produced from manifests, compose files, flow definitions, workflow files, scripts, directory listings, and each subfolder's `README.md`.
- Java sources under `metersense-release-5.1/` and `Test Repository/src/` were listed, not read line by line; files under `SDLC-Artifacts/Artifacts/` were sampled.
- No command was executed. Each command carries the marker of the file that defines it.

## Subfolder map

| Path | Contains | Role |
| --- | --- | --- |
| `AIAgents/` | Four agent packs, manifest schema, `.github/` governance files | Defines the agents that stages run `[from-config]` |
| `SDLC-Orchestration/` | Kestra flows, compose files, scripts, `service/` Spring Boot app | Runs the stage pipeline and records each execution `[from-config]` |
| `SDLC-Artifacts/` | `Artifacts/` tree plus reusable GitHub Actions workflows | Persists every artifact a run produces `[from-config]` |
| `metersense-release-5.1/` | Ant and Maven build of the Compass product, `cust/` overlays, `webapps/`, `db/` | Product source the pipeline clones and edits `[inferred]` |
| `Test Repository/` | Maven project `Compass-Automation`, TestNG suites, page objects | Playwright UI tests for the Compass web application `[from-config]` |
| `Manual Process/` | Two Atlassian Forge apps plus MatchaAI prompt Markdown | Standalone Jira-to-Xray manual test case workflow `[from-config]` |

`Manual Process/` and `Test Repository/` do not participate in the Kestra pipeline: no flow or config file names either folder, and the `Verification` stage that would run Playwright suites is a stub `[from-config]`.

## Cross-project relationships

The automated pipeline spans four subfolders: `SDLC-Orchestration`, `AIAgents`, `SDLC-Artifacts`, and `metersense-release-5.1`.

1. `master-sdlc` in `SDLC-Orchestration/_flows/main_master.yml` receives a `ticketId` and calls `execution.initialize`, which fetches the Jira ticket, reads `_config/projects.yml`, and prepares the artifact workspace; `execution-record.*` flows record the run in `aidlc-service` `[from-config]`.
2. `master-sdlc` loops: `execution.resolve-next-stages` names the ready stages, and `stage.wrapper` runs each one, in parallel when several are ready `[from-config]`.
3. Each stage fetches input artifacts from `SDLC-Artifacts` through `_flows/util/artifacts/store/git.sh` and clones the product repository from Bitbucket when the stage needs a branch `[from-config]`.
4. Agent stages call `agents.run-container`, which loads the agent from `AGENT_CONFIGS_REPO` or `LOCAL_AI_AGENTS` and runs a container from `AI_CONTAINER_IMAGE` `[from-config]`; the agent writes Markdown to `/work/resources/output/` `[from-docs]`.
5. `stage.finalize` publishes the stage directory into `SDLC-Artifacts` under `Artifacts/<TICKET>/<EXEC_ID>/` `[from-config]`.
6. `CodeGeneration` pushes a feature branch to the Bitbucket repository named in `_config/projects.yml`; `Review` opens the pull request through the Bitbucket REST API in `_flows/stages/review/create-pr.sh` `[from-config]`.

Boundaries that must not be crossed:

- Agent packs never orchestrate; sequencing lives in each stage's `nextStages` and in `_flows/main_master.yml` `[from-config]`.
- Stage flows reference an agent by `agentName`; prompt text lives only in `AIAgents` `[from-config]`.
- `SDLC-Orchestration` reaches `SDLC-Artifacts` by git clone and push only; no flow calls the GitHub Actions workflows in `SDLC-Artifacts/.github/workflows/` `[from-config]`.

## Stage-to-agent mapping

| Pipeline stage | Agent path passed as `agentName` | Confidence agent suffix (threshold) | Output |
| --- | --- | --- | --- |
| `Analysis` | `AIDLC/Analysis/jira-reviewer` | `-confidence` (50) | `analysis.md` |
| `Design` | `AIDLC/Design/solution-designer` | `-confidence` (50) | `design.md`, `implementation.md` |
| `FirstSecurityReview` | `AIDLC/FirstSecurityReview/` `design-reviewer`, `finding-challenger`, `review-reporter` | `input-confidence` (80) | `security-review.md` |
| `CodeGeneration` | `AIDLC/CodeGeneration/code-generator` | `-confidence` (50) | `code.md` plus a feature branch |
| `Review` | none; Bitbucket REST call | none | Pull request link on the execution record |
| `Ideation`, `Requirements`, `TestCaseGeneration`, `SecondSecurityReview`, `Verification`, `Release` | none; stubs | none | none |

All rows `[from-config]`. A `-confidence` suffix is appended to the stage's agent path, for example `AIDLC/Analysis/jira-reviewer-confidence`.

## AIAgents

Source of truth: [`AIAgents/README.md`](./AIAgents/README.md).

### AIAgents purpose

- Contains four agent packs: `jira-reviewer/`, `solution-designer/`, `code-generator/`, `ping-test/` `[from-config]`.
- Holds governance files `AIAgents/AGENTS.md` and `AIAgents/.github/` `[from-config]`.
- Stage flows request agents by paths such as `AIDLC/Analysis/jira-reviewer`; the packs here are flat (`jira-reviewer/`), so the layout in `AGENT_CONFIGS_REPO` differs from this folder `[from-config]`.

### AIAgents pack structure

- Required per pack: `<agent-name>/manifest.json` and `<agent-name>/agents/<agent-name>.agent.md`; optional: `instructions/`, `skills/<skill-name>/SKILL.md`, `mcp-config.json` `[from-docs]`.
- Only `jira-reviewer` ships `documentation/examples/{clean,compass,vuln}/Jira/input.json` and an `mcp-config.json` `[from-config]`.
- The minimum viable pack is `ping-test/` (manifest plus one agent definition); the `system.ping-test` flow runs it and expects `pong` `[from-config]`.

### AIAgents manifest rules

- Every manifest validates against `AIAgents/schemas/agent-manifest-v1.schema.json` (`additionalProperties: false`); `schema_version` is `"1.0"`; `agent.name` equals the folder name; `files` lists every file in the folder including `manifest.json` `[from-docs]`.
- All four packs set `container.image` to `ghcr.io/harris-smartworks/copilot-cli:latest` `[from-config]`.
- `SDLC-Orchestration/.env.example` sets `AI_CONTAINER_IMAGE` to `ghcr.io/harris-smartworks/copilot-cli:v1.0.1`, which the flows use at runtime `[from-config]`.

### AIAgents runtime paths and output contract

- Container paths: `/work/resources/Jira/input.json` (ticket), `/work/resources/inputs/artifacts/<name>.md` with fallback `/work/resources/artifacts/<name>.md` (upstream artifacts), `/work/resources/inputs/source` (target repository), `/work/resources/output/<name>.md` (result) `[from-docs]`.
- The first heading of every output file is `# Status`, followed by a `STATUS_CODE:MESSAGE` line and `Confidence: <percentage>%`; Orchestration fails the stage when the heading is absent, renamed, or demoted `[from-docs]`.
- Output files are overwritten on each run; a failure still writes a file containing only the Status section; full reasoning goes to STDOUT `[from-docs]`.
- Status codes: `SUCCESS`, `JSON_NOT_FOUND`, `JSON_MALFORMED` for `jira-reviewer`; `SUCCESS`, `DESIGN_NOT_FOUND`, `IMPLEMENTATION_NOT_FOUND`, `SOURCE_NOT_FOUND` for `code-generator` `[from-docs]`.

## SDLC-Orchestration

Source of truth: [`SDLC-Orchestration/README.md`](./SDLC-Orchestration/README.md).

### Orchestration runtime

- Kestra is pinned to `v1.3.31` by digest in `SDLC-Orchestration/Dockerfile`, with `plugin-docker` `1.5.3`, `plugin-script-shell` `1.9.7`, `git`, `jq`, and checksum-pinned `yq` `v4.53.6` `[from-config]`.
- `docker-compose.yml` defines `postgres` (`postgres:15`), `kestra`, and `aidlc-service`; Kestra maps host port `8080` by default (`KESTRA_PORT`), with the UI at `http://localhost:8080/ui/main` and the REST API under `/api/v1/main/` `[from-config]`.
- `podman-compose.yml` comments and `SDLC-Orchestration/AGENTS.md` name production port `8082` `[from-config]`.
- Prerequisites: Docker or Podman, Git, and a `.env` based on `SDLC-Orchestration/.env.example` `[from-docs]`.
- Development uses `FLOW_WATCH=true`; Kestra reloads saved files under `_flows/`, and filenames must begin with `main_` `[from-config]`.
- Development start and stop go through `scripts/dev/start.sh` and `scripts/dev/stop.sh`, not `docker compose up` or `down` `[from-docs]`.
- Production uses `FLOW_WATCH=false` (the `podman-compose.yml` default); `system.gitops-sync` runs every five minutes, validates flows from the checked-out branch, and deploys after in-flight executions drain; seed it once with `scripts/deploy/update.sh` `[from-config]`.

### Orchestration flow layout

| Path (under `SDLC-Orchestration/`) | Contains |
| --- | --- |
| `_flows/main_master.yml` | The only API entry point, `master-sdlc`; dispatches ready stages, then records the outcome, reconciles the manifest, and sends the email in a `finally` block `[from-config]` |
| `_flows/stages/` | Eleven `stage.*` flows, one folder each, declaring `resources`, `deliverables`, `confidence`, `nextStages` `[from-config]` |
| `_flows/orchestration/` | `execution.initialize`, `execution.resolve-next-stages`, `stage.initialize`, `stage.wrapper`, `stage.finalize`, `stage.fail` `[from-config]` |
| `_flows/util/` | `agents.*`, `artifacts.*`, `execution-record.*` (calls `aidlc-service`), `git.*`, `jira.*`, `config.read-projects`, `notify.send-email` `[from-config]` |
| `_flows/maintenance/`, `_flows/diagnostics/` | `cleanup-run`, `sweep-working-dirs`, `system.ping-test` `[from-config]` |
| `_config/` | `kestra-application.yml`, `pipeline.yml` (start stage per ticket type), `projects.yml` (repository, branch, version rules per Jira project) `[from-config]` |
| `scripts/` | `dev/` helpers, `deploy/` scripts including `gitops-sync/`, and 14 `tests/test-*.sh` `[from-config]` |

To run locally, create `.env` from `.env.example`, run `.\scripts\dev\setup.ps1`, then `docker compose build` and `docker compose up -d`; on Podman use `podman-compose -f podman-compose.yml` `[from-docs]`.

### Execution service

- `service/` is the Spring Boot `3.5.16` application `aidlc-execution-service` `0.1.0` on Java `21` with JPA, Flyway, PostgreSQL, actuator, and springdoc `[from-config]`.
- Migration `service/src/main/resources/db/migration/V1__init.sql` creates `executions`, `stages`, `execution_gates`, `execution_branches`, `execution_links`, `execution_properties`, `release_notes` `[from-config]`.
- REST resources sit under `/api/executions`; `BearerTokenFilter` guards them with the token from `SECRET_AIDLC_SERVICE_TOKEN`, which the flows read as `AIDLC_SERVICE_TOKEN` `[from-config]`.
- Compose publishes the service on `AIDLC_SERVICE_PORT` (default `8086`); health is `/actuator/health`, docs are `/swagger-ui.html`; data lives in the `aidlc` database set by `AIDLC_DB_*` `[from-config]`.
- `gitops-sync` does not deploy `service/`; run `scripts/deploy/update.sh` after changing `service/`, the `Dockerfile`, or a compose file `[from-docs]`.

### Lifecycle graph and current stage status

The graph is compiled from stage `nextStages`, `resources`, and `deliverables`. A stage starts after all upstream stages complete, and parallel ready stages are dispatched together `[from-config]`.

- Idea tickets run `Ideation` → `Requirements` → `Analysis` → `Design` → `FirstSecurityReview`; other tickets start at `Requirements`; Bug tickets start at `Analysis` (`_config/pipeline.yml`) `[from-config]`.
- `FirstSecurityReview` forks to `CodeGeneration` and `TestCaseGeneration`, which both feed `SecondSecurityReview`; that stage forks to `Review` and `Verification`, which both feed `Release` `[from-config]`.
- Stubs (log and succeed, empty `resources` and `deliverables`): `Ideation`, `Requirements`, `TestCaseGeneration`, `SecondSecurityReview`, `Verification`, `Release` `[from-config]`.
- Non-stubs: `Analysis`, `Design`, `FirstSecurityReview`, `CodeGeneration`, `Review` `[from-config]`.

### Triggering a pipeline run

The pipeline endpoint is `POST /api/v1/main/executions/ai-dlc/master-sdlc`, using HTTP Basic authentication and `multipart/form-data` `[from-docs]`. Inputs, all `[from-config]`:

- `ticketId` (required): Jira ticket key, for example `ABC-123`.
- `requestorEmail` (default `""`): recipient of the outcome email.
- `initialStage` (default `[]`): stage(s) to start at; empty derives the stage from the ticket type.
- `finalStages` (default `["Release"]`): last stage(s) to run; naming one stage of a parallel pair does not start its sibling.
- `previousExecutionPath` (default `""`): earlier artifact execution to copy into this run.
- `skipConfidenceCheck` (default `"First"`): `False` skips none, `First` skips the starting stage, `True` skips all.
- `dryRun` (default `false`): suppresses Jira updates, the pull request, and the email; branch pushes and artifact commits still happen.
- `humanGatedStages` (default `[]`): stages to pause after for approval; accepts the eleven stage names except `Release`.

### Artifact lifecycle

Each run is stored in `SDLC-Artifacts` under `Artifacts/<TICKET>/<EXEC_ID>/`. Stages fetch their workspace, run, and publish their stage directory; a `previousExecutionPath` is copied into the new run `[from-docs]`. Setting `LOCAL_SDLC_ARTIFACTS` switches the store from `git.sh` to `local.sh`, which reads and writes a mounted folder and commits nothing `[from-config]`.

## SDLC-Artifacts

Source of truth: [`SDLC-Artifacts/README.md`](./SDLC-Artifacts/README.md).

### Artifact storage convention

- Artifacts live under `Artifacts/<JIRA_ID>/<YYYYMMDD_HH24MISS>/`, and every execution directory must contain a `manifest.json` `[from-docs]`.
- Each execution directory holds one subfolder per stage `artifactDir` (`Jira/`, `Requirements/`, `Analysis/`, `Design/`, `Implementation/`), with `<name>_reasoning.txt` beside agent-written artifacts `[inferred]`.
- Baseline manifest keys are documented in `SDLC-Artifacts/Artifacts/README.md` `[from-docs]`.
- Recent manifests use `schema_version` `1.7.0` and add `state`, `request`, `context`, `config`, `values`, `files`, `stage_registry`, and `plan` `[from-config]`.
- The `Artifacts/` tree holds seven ticket folders, from `IDEA-29` to `METERSENSE-11321` `[inferred]`.

### Artifact reusable workflows

Workflow files under `SDLC-Artifacts/.github/workflows/` `[from-config]`:

- `prepare-execution-directory.yml`: creates the execution folder and manifest; internal helper; output `manifest_path`.
- `prepare-artifacts.yml`: prepares directory and manifest without committing; outputs `currdate_string`, `artifact_path`, `manifest_path`, `bundle_name`.
- `persist-artifacts.yml`: the only workflow that commits and pushes under `Artifacts/`; outputs `changed`, `commit_sha`; default `commit_message` `chore(artifacts): persist workflow artifacts`; secret `CODEGEN_ACTION_PAT`; checks out `main`.
- `test-artifact-workflows.yml`: end-to-end test using ticket `ZZZTEST-WORKFLOW-E2E`; runs on `workflow_dispatch` and on pushes to `main` touching `.github/workflows/**`.

Rules:

- The prepare workflows require `jira_id` or `artifact_path`; with only `jira_id`, `artifact_path` resolves to `<jira_id>/<currdate_string>` `[from-docs]`.
- `artifact_bundle_name` downloads an uploaded workflow artifact before the commit and requires `artifact_path` `[from-docs]`.
- The documented composition order is prepare, then generate, then persist `[from-docs]`.
- No flow in `SDLC-Orchestration/` invokes these workflows `[from-config]`.

## Test Repository

Source of truth: [`Test Repository/README.md`](<./Test Repository/README.md>).

### Test Repository identity

- The Maven coordinates are `com.smartwork.compass:Compass-Automation:0.0.1-SNAPSHOT` `[from-config]`.
- The project is a single Maven module with `24` TestNG test classes under `src/test/java/tests/` plus a stray `test.java`, and `30` page objects under `src/main/java/pages/` `[inferred]`.
- The project drives the Compass web application and contains no Compass application source code `[from-docs]`.
- The project ships no CI pipeline definition; `.github/` holds only `copilot-instructions.md` `[from-config]`.

### Test Repository stack

- `Java 21`, `com.microsoft.playwright:playwright` `1.57.0`, `org.testng:testng` `7.12.0`, `com.aventstack:extentreports` `5.1.2`, `jackson-databind` `2.18.6`, `org.apache.poi:poi-ooxml` `5.2.3` `[from-config]`.
- Database drivers: `org.postgresql:postgresql` `42.7.10` and `com.oracle.database.jdbc:ojdbc11` `23.26.2.0.0` `[from-config]`.
- Plugins: `maven-surefire-plugin` `3.2.5`, `exec-maven-plugin` `3.6.3` (launches `com.microsoft.playwright.CLI`) `[from-config]`.

### Test Repository commands

- `mvn exec:java -D"exec.args=install"` installs Playwright browsers; run once before the first test run `[from-config]`.
- `mvn test` runs the suite set in `pom.xml` `[from-config]`.
- `mvn test -DsuiteXmlFile=smoketest.xml` and `mvn test -DsuiteXmlFile=regressiontest.xml` run the other suites `[from-docs]`.
- `mvn test -Dtest=EditMeterTest` runs one class; `mvn test -Dactive.env=env2` selects the Oracle environment `[from-docs]`.
- Run every command from `Test Repository/`.

Generated output that must never be hand-edited: `Test Repository/test-output/` and `Test Repository/target/` `[from-config]`.

Suite definitions: `testNg.xml` (`Cross Browser Suite`, `parallel="classes"`, `thread-count="3"`), `smoketest.xml`, `regressiontest.xml` `[from-config]`.

## Manual Process

Source of truth: [`Manual Process/README.md`](<./Manual Process/README.md>).

### Manual Process components

- `Test Case Generator/Forge-Test-Case-Generator/`: Forge app with a Node backend (`src/`, 14 helpers in `src/functions/`) and a React Custom UI (`static/`); on `main` @ `fe6e03ee501e` `[from-config]`.
- `Forge-Xray-Import-MCP/`: Forge app with the web trigger handler `src/forge.js`, importer, retriever, and Jest tests; on `main` @ `5b5230638086` `[from-config]`.
- `Test Generator Agents/Plan/`: `Requirement-Generator.md` and `Test-Step-Generator.md`, the prompts for the two MatchaAI missions `[from-docs]`.
- `Test Generator Agents/Code/` is an empty directory `[inferred]`.

### Manual Process flow

1. A QA analyst opens the `Test Case Generator` issue panel on a Jira ticket `[from-config]`.
2. The generator collects ticket data (`prepareGenerateTestCases`; child tickets only for Epic and Story) or takes user-supplied text (`prepareGenerateFromText`) `[from-config]`.
3. The generator calls the MatchaAI Requirement Generator mission, then the Steps Generator mission; `continueWithManual*` resolvers provide a manual fallback `[from-config]`.
4. The analyst selects, edits, reorders, and approves test cases in the React Review panel `[from-docs]`.
5. The generator POSTs approved Xray JSON to the importer web trigger (`submitToXray`, `submitTestCases`); the importer creates each Test sequentially through the Xray GraphQL API `[from-config]`.
6. `finalizeTestCases` links each Test to its requirements and Test Executions, then comments on the ticket `[from-config]`.

### Manual Process key commands

- `npm run lint` in `Test Case Generator/Forge-Test-Case-Generator/` `[from-config]`.
- `npm run build` in `Test Case Generator/Forge-Test-Case-Generator/static/` `[from-config]`.
- `npm test -- --watchAll=false` in `Test Case Generator/Forge-Test-Case-Generator/static/` `[from-docs]`.
- `npm test` in `Forge-Xray-Import-MCP/` `[from-config]`.
- `npm run deploy` in either app root `[from-config]`.
- `forge webtrigger create --functionKey xray-import-trigger` in `Forge-Xray-Import-MCP/` `[from-docs]`.
- Ordering: `forge login` precedes any `forge` subcommand; the importer deploys and its web trigger exists before the generator is configured `[from-docs]`.

## metersense-release-5.1

The folder has no top-level `README.md`; area docs are `VirtualMachine/README.md`, `RulesEditor_v2/README.md`, and `python/README.md` `[from-config]`.

### metersense-release-5.1 identity

- The product is SmartWorks Compass (formerly MeterSense); Maven coordinates are `com.harriscomputer.metersense:compass`, Java `21` `[from-config]`.
- `pom.xml` declares version `5.0` with `java/` as the source directory and `Testing/java` as test sources `[from-config]`.
- `pom2.xml` declares version `5.1`, `packaging` `pom`, and 15 modules under `MavenProject/Modules/` (`Security`, `CommonUtility`, `Billing`, `RulesEngine`, `VEE`, `WebApplication`, and 9 others); the `tests` profile adds the `Testing` module `[from-config]`.
- `SDLC-Orchestration/_config/projects.yml` maps Jira project `METERSENSE` to Bitbucket `MS/metersense`, base branch `release/{targetVersion}`, default version `5.1`; this folder is the checked-in counterpart `[inferred]`.

### metersense-release-5.1 directory map

| Path | Contains |
| --- | --- |
| `java/com/harriscomputer/metersense/` | Application Java sources `[inferred]` |
| `MavenProject/` | `Modules/` plus `bin/` Ant-to-Maven migration and deploy scripts `[inferred]` |
| `Testing/` | Test `java/`, `jsp/`, `lib/`, `testdata/`, `runMavenTests.sh` `[from-config]` |
| `webapps/` | `ROOT`, `admin`, `axis`, `compassAPI`, `visualizationAPI`, `workflowAPI` `[inferred]` |
| `cust/` | One folder per customer or environment with `metersense.properties`; `CUSTOMER_TEMPLATE/` is the template `[inferred]` |
| `db/`, `VirtualMachine/` | `oracle/` and `postgres/` scripts; VM build, hostname, and data import and export scripts `[from-docs]` |
| `RulesEditor/`, `RulesEditor_v2/` | Legacy GWT-style editor `[inferred]`; React 18 Create React App editor `[from-config]` |
| `build.sh`, `build.xml`, `ant.jar` | Ant build driver; `ant.jar` expands into `ant/` on first run `[from-config]` |

### metersense-release-5.1 commands

Run from `metersense-release-5.1/`:

- `./build.sh help` lists Ant targets; `./build.sh -c <custKey> hard.refresh` refreshes a customer install `[from-config]`.
- `Testing/runMavenTests.sh -c <custKey> -s <suite> -t <class>` runs unit tests; suites include `ALL`, `DB`, `UI`, `JAVA`, `Sanity`, `Slice`; the default customer is `Bamboo3` `[from-config]`.
- `VirtualMachine/maven_build.sh -c <customer>` builds on a VM with `pom2.xml` `[from-docs]`.
- `npm start` in `RulesEditor_v2/` starts the Rules Editor dev server `[from-config]`.

## External integrations across the folder

| Integration | Used by | Config or variable names |
| --- | --- | --- |
| Jira | `SDLC-Orchestration`, `jira-reviewer`, both Forge apps | `JIRA_URL`, `JIRA_API_URL`, `SECRET_JIRA_API_TOKEN` `[from-config]` |
| Bitbucket | `SDLC-Orchestration` clone, branch, and pull request flows | `BITBUCKET_URL`, `BITBUCKET_USERNAME`, `SECRET_BITBUCKET_TOKEN`, `BITBUCKET_PROJECT` `[from-config]` |
| GitHub and GHCR | `SDLC-Orchestration` artifact store, agent checkout, container image; `SDLC-Artifacts` workflows | `SECRET_GITHUB_TOKEN`, `AGENT_CONFIGS_REPO`, `SDLC_ARTIFACTS_REPO`, `AI_CONTAINER_IMAGE`, `SECRET_GHCR_TOKEN`; secret `CODEGEN_ACTION_PAT` `[from-config]` |
| MCP servers | Agent containers; `jira-reviewer` | `ATLASSIAN_ROVO_ENABLED`, `SECRET_ATLASSIAN_ROVO_API_KEY`; NVD server `io.github.pipeworx-io/nvd` `[from-config]` |
| SMTP | `SDLC-Orchestration` outcome email | `SMTP_HOST`, `SMTP_PORT`, `SMTP_FROM`, `ADMIN_EMAILS`, `ENABLE_EMAIL_UPDATES` `[from-config]` |
| PostgreSQL | `SDLC-Orchestration`, `Test Repository` | `KESTRA_DB_*`, `AIDLC_DB_*`; `<env>.db.*` in `config.properties` `[from-config]` |
| MatchaAI | `Manual Process` generator | `ai_endpoint_url`, `requirement_mission_id`, `steps_mission_id`, `matcha_api_key` `[from-docs]` |
| Xray Cloud GraphQL | `Manual Process` importer | `XRAY_CLIENT_ID`, `XRAY_CLIENT_SECRET`, `XRAY_REGION`, `JIRA_PROJECT_KEY`, `XRAY_LINK_TYPE_NAME` `[from-docs]` |

Only variable names belong in documentation. Never commit a value for any variable named above.

## Conventions shared across the folder

- Update the affected subfolder's `README.md` in the same change set as a content change `[from-docs]`.
- Keep each `AIAgents` manifest `files` list synchronized with that agent folder's contents `[from-config]`.
- Name Kestra flow files with the `main_` prefix; edit flows as files, never in the Kestra UI or by `PUT` to the API `[from-docs]`.
- Set `required: true` on every Kestra flow input that has `defaults:` `[from-docs]`.
- Keep prompt output format and the parser that consumes it in sync within `Manual Process` `[from-docs]`.
- Store secrets through the platform's secret mechanism, never in a committed file `[from-docs]`.

## Gotchas

- `AIAgents` manifests fail validation on any undeclared field `[from-docs]`.
- A Kestra flow file without the `main_` prefix is silently ignored `[from-config]`.
- A `master-sdlc` request is refused while `system.gitops-sync` holds the `gitops-drain` flag; resubmit after the deploy lands `[from-config]`.
- `Forge-Xray-Import-MCP/README.md` embeds a live web trigger URL with its secret path segment; treat the file as sensitive `[from-config]`.
- `Test Repository/pom.xml` sets `suiteXmlFiles` to `testng.xml`, but the file on disk is `testNg.xml`; the case differs `[from-config]`.
- `Test Repository/pom.xml` never reads a `suiteXmlFile` property, so `-DsuiteXmlFile=...` may not select a suite `[inferred]`.
- The three `Test Repository` suites list `tests.RateImpactAnalysisTest`, and `regressiontest.xml` lists `tests.IntelligentSegmentationTest`; neither class exists under `src/test/java/tests/` `[from-config]`.
- `metersense-release-5.1/build.sh -c <custKey>` deletes every other `cust/<name>` folder outside `customer.inheritance.list`, unless `-k` is passed or the key is `DemoVMLinux` `[from-config]`.
- `metersense-release-5.1/pom2.xml` sets `testFailureIgnore` to `true`; `Testing/runMavenTests.sh` derives the exit code from the reports `[from-config]`.

## Where to change X

| Task | Start in |
| --- | --- |
| Change what an agent produces | `AIAgents/<agent>/instructions/` |
| Add an agent pack | `AIAgents/<new-agent>/manifest.json`, checked against `AIAgents/schemas/agent-manifest-v1.schema.json` |
| Change stage sequencing or gating | `SDLC-Orchestration/_flows/stages/<stage>/main_<stage>.yml` (`nextStages`) and `_flows/main_master.yml` |
| Change the start stage per ticket type, or onboard a Jira project | `SDLC-Orchestration/_config/pipeline.yml`, `SDLC-Orchestration/_config/projects.yml` |
| Change where artifacts are stored | `SDLC-Orchestration/_flows/util/artifacts/store/` |
| Change the execution-record schema or API | `SDLC-Orchestration/service/src/main/resources/db/migration/`, `service/src/main/java/com/harris/sdlc/execution/` |
| Add a UI test scenario | `Test Repository/src/test/java/tests/` |
| Change generated test case wording | `Manual Process/Test Generator Agents/Plan/` |
| Change a Compass customer build setting | `metersense-release-5.1/cust/<custKey>/metersense.properties` |

## Glossary

| Term | Definition |
| --- | --- |
| AI-DLC | The AI-assisted delivery lifecycle of eleven stages run by `SDLC-Orchestration` |
| Agent pack | A folder in `AIAgents` holding a manifest plus agent definition |
| Compass | The SmartWorks product, formerly MeterSense, whose source is `metersense-release-5.1/` |
| Confidence check | A pre-stage agent score out of 100; below the stage threshold the run stops and is Blocked |
| Customer key | A folder name under `metersense-release-5.1/cust/` passed to `build.sh -c` |
| Execution record | The run, stage, gate, branch, link, property, and release-note rows stored by `aidlc-service` |
| Human-gated stage | A stage after which the pipeline pauses for approval |
| Stub stage | A stage flow that logs, succeeds, and declares no resources or deliverables |

## Further reading

- [`AIAgents/README.md`](./AIAgents/README.md) and [`AIAgents/AGENTS.md`](./AIAgents/AGENTS.md): pack structure, manifest schema, change rules.
- [`SDLC-Orchestration/README.md`](./SDLC-Orchestration/README.md) and [`SDLC-Orchestration/AGENTS.md`](./SDLC-Orchestration/AGENTS.md): Kestra setup, trigger inputs, local-override mounts, safe start, stop, and flow editing.
- [`SDLC-Artifacts/README.md`](./SDLC-Artifacts/README.md): storage convention and the reusable workflows.
- [`Test Repository/README.md`](<./Test Repository/README.md>): Playwright and TestNG architecture, configuration keys.
- [`Manual Process/README.md`](<./Manual Process/README.md>): the two Forge apps and the Jira-to-Xray flow.
- [`metersense-release-5.1/VirtualMachine/README.md`](<./metersense-release-5.1/VirtualMachine/README.md>), [`metersense-release-5.1/RulesEditor_v2/README.md`](<./metersense-release-5.1/RulesEditor_v2/README.md>): VM scripts, React Rules Editor.

## Unknowns and unverified claims

| Claim or question | Status | How to resolve |
| --- | --- | --- |
| Uncommitted working-tree changes under `References/` | `[unknown]` | Run `git status` in the `SDLC` repository |
| `AGENT_CONFIGS_REPO` layout: flows request `AIDLC/<Stage>/<agent>` paths (including `*-confidence` and `FirstSecurityReview` agents); `AIAgents/` here holds four flat packs | `[unknown]` | List the `AIAgents` repository folders at `AGENT_CONFIGS_REF` |
| Committed manifests such as `SDLC-Artifacts/Artifacts/IDEA-29/20261120_054125/manifest.json` record `Requirements` as a non-stub and `FirstSecurityReview` as a stub, opposite to the current flows | `[unknown]` | Compare the orchestration commit behind that run with `SDLC-Orchestration/_flows/stages/` |
| `service/README.md`, `_flows/stages/README.md`, and `_flows/stages/*/contract.yml` are referenced but absent | `[unknown]` | Ask the `SDLC-Orchestration` maintainer |
| Run and test commands for `SDLC-Orchestration/service/` | `[unknown]` | Read `service/Dockerfile` and `scripts/tests/test-execution-service.sh` |
| Production Kestra port: `8080` mapping default versus `8082` in `AGENTS.md` | `[unknown]` | Read `KESTRA_PORT` in the production `.env` |
| Authoritative Compass build: `metersense-release-5.1/pom.xml` (`5.0`) or `pom2.xml` (`5.1`) | `[unknown]` | Ask the Compass build owner |
| Whether `metersense-release-5.1/` equals Bitbucket `MS/metersense` branch `release/5.1` | `[unknown]` | Diff against a clone of that branch |
| Whether the `Test Repository` suites run despite the missing `RateImpactAnalysisTest` and `IntelligentSegmentationTest` classes, and whether `-DsuiteXmlFile` selects a suite | `[unknown]` | Run `mvn test` and `mvn test -DsuiteXmlFile=smoketest.xml`, then read the surefire log |
| Whether `Manual Process/Test Generator Agents/Code/` is intentionally empty | `[unknown]` | Ask the `Manual Process` maintainer |
