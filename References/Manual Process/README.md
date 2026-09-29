# Manual Process — Repository Context

> `Manual Process` is a folder containing three cooperating parts of one QA workflow that turns a Jira ticket into manual Xray test cases. `Test Case Generator/Forge-Test-Case-Generator` is an Atlassian Forge app (Jira issue panel + admin page) that collects ticket data, calls an external MatchaAI endpoint to draft test cases, and lets a QA analyst review and edit them. `Forge-Xray-Import-MCP` is a second Forge app exposing a web trigger that imports those approved test cases into Xray Cloud through Xray's GraphQL API and reads them back. `Test Generator Agents` holds the plain-Markdown prompt specifications for the two MatchaAI missions the generator calls. The folder produces no shared build artifact; each Forge app deploys independently with the Forge CLI.

| Field | Value |
| --- | --- |
| Analysis root | `Manual Process/` |
| Analyzed on | `2026-09-25` |
| Shape | Folder holding 2 independent Git repositories plus 1 documentation-only folder |
| Primary languages | `JavaScript` (ES modules in the generator, CommonJS-style Node in the importer), `JSX` (React 18) |
| Runtimes | Atlassian Forge `nodejs22.x`; local Node.js `>=18.0.0` |
| Build / package manager | `npm`; Atlassian `forge` CLI |
| Branch / commit | `Forge-Test-Case-Generator`: `main` @ `fe6e03ee` — `Forge-Xray-Import-MCP`: `main` @ `5b523063` |
| Paths in this document | Relative to `Manual Process/` |

## How to use this document

- Trust the facts below. Search the subfolders only when a fact is missing, marked `[unknown]`, or contradicted by code.
- If code contradicts this document, the code wins. Report the contradiction.
- Status markers: `[from-config]` read from a manifest, `package.json`, or `.env.example`; `[from-docs]` claimed in project documentation and not confirmed against code; `[inferred]` deduced from file structure and not executed; `[unknown]` open question.
- No command in this document was executed. Treat every command as unvalidated.
- `Manual Process/` itself is not a Git repository; the two Forge apps each carry their own `.git` directory `[inferred]`.

## Purpose and scope

- Does: generate draft manual test cases from Jira ticket content using an external AI service `[from-docs]`.
- Does: present a review and edit UI inside the Jira issue panel before anything is written to Xray `[from-docs]`.
- Does: create Xray Test issues and link them back to requirement tickets and Test Executions `[from-docs]`.
- Does: store the MatchaAI mission prompts as version-controlled Markdown in `Test Generator Agents/Plan/`.
- Does not: run automated test execution, CI pipelines, or scheduled jobs — no workflow files exist under either app `[inferred]`.
- Does not: contain the MatchaAI platform itself; `Test Generator Agents/` holds only prompt text `[inferred]`.
- Consumers: QA analysts working in the Jira projects `METERSENSE`, `COMPTEST`, and `CONNECT` `[from-docs]`.

## Component map

| Path | Contains | Role in the workflow |
| --- | --- | --- |
| `Test Case Generator/Forge-Test-Case-Generator/` | Forge app: Node backend (`src/`) + React Custom UI (`static/`) | Step 1 — collect ticket data, call AI, review and edit, POST approved JSON |
| `Forge-Xray-Import-MCP/` | Forge app: Node backend (`src/`), web trigger handler, Jest tests | Step 2 — receive JSON, create Tests in Xray Cloud, retrieve Tests |
| `Test Generator Agents/Plan/` | `Requirement-Generator.md`, `Test-Step-Generator.md` | Step 0 — prompt definitions for the two MatchaAI missions |
| `Test Generator Agents/Code/` | Empty directory | `[unknown]` — see the unknowns section |

## End-to-end data flow

Numbered flow, from Jira ticket to Xray Test issue `[from-docs]`:

1. A QA analyst opens a Jira ticket and expands the `Test Case Generator` issue panel (`jira:issuePanel` key `epic-panel`).
2. Resolver `prepareGenerateTestCases` authorizes the user, reads the ticket and — for Epic and Story types only — its child tickets, then builds the AI payload.
3. Resolver `generateRequirements` POSTs `{ mission_id, input }` to the configured MatchaAI endpoint using the Requirement Generator mission and receives plain-text requirements.
4. Resolver `generateStepTestCases` POSTs the requirements to the Steps Generator mission and parses the returned Markdown into structured test case objects.
5. The React Review panel lets the analyst select, edit, reorder steps, and approve test cases.
6. Resolvers `prepareSubmission` then `submitToXray` build Xray JSON and POST the payload to the `Forge-Xray-Import-MCP` web trigger URL stored in admin settings.
7. `Forge-Xray-Import-MCP/src/forge.js` receives the POST, and `src/importer.js` creates each Test sequentially through the Xray GraphQL API.
8. Resolver `finalizeTestCases` links each created Test to its requirement tickets and Test Executions, then adds a completion comment to the source ticket.

Boundaries that must not be crossed:

- The generator never calls Xray directly; every Xray write goes through the importer web trigger `[inferred]`.
- The importer never calls MatchaAI; the importer's manifest allowlists only Xray Cloud addresses `[from-config]`.
- Prompt wording lives only in `Test Generator Agents/Plan/`; the generator sends a `mission_id`, not prompt text `[from-docs]`.

## Component: Forge Test Case Generator

### Generator entry points

| Entry point | File and symbol | Invoked by |
| --- | --- | --- |
| Backend resolver handler | `Test Case Generator/Forge-Test-Case-Generator/src/index.js` re-exports `index.handler` | Forge `function` key `resolver` in `manifest.yml` `[from-config]` |
| Resolver definitions | `Test Case Generator/Forge-Test-Case-Generator/src/resolvers/index.js` | The Custom UI over `@forge/bridge` `[from-docs]` |
| React root | `Test Case Generator/Forge-Test-Case-Generator/static/src/index.jsx` | Forge resources `main` and `admin`, both served from `static/build` `[from-config]` |
| Issue panel module | `jira:issuePanel` key `epic-panel`, title `Test Case Generator` | A QA analyst opening a Jira ticket `[from-config]` |
| Admin page module | `jira:adminPage` key `admin-settings`, title `Matcha Settings` | A Jira administrator `[from-config]` |

### Generator directory layout

| Path (under `Test Case Generator/Forge-Test-Case-Generator/`) | Contains |
| --- | --- |
| `src/resolvers/index.js` | Every resolver exposed to the UI |
| `src/functions/` | 13 side-effect-light helpers, including `collectIssueData.js`, `callMatchaAI.js`, `parseTestCases.js`, `buildXrayJson.js`, `postToTrigger.js`, `createIssueLinks.js`, `buildFinalizeSummary.js` |
| `static/src/panels/` | `EpicPanel.jsx`, `ReviewPanel.jsx`, `AdminPanel.jsx` |
| `static/src/components/` | `Breadcrumb.jsx`, `TestCaseCard.jsx`, `StepsTable.jsx`, `FinalizeSummary.jsx`, `ManualFallback.jsx`, `Spinner.jsx`, `StatusBanner.jsx` |
| `static/src/functions/` | Frontend copies of `parseTestCases.js`, `buildXrayJson.js`, `applyWorkflowLabels.js`, each with a sibling `.test.js` |
| `static/src/local-dev/fixtures/` | Mock resolver responses used when the Forge bridge is unavailable |
| `static/build/` | Generated production bundle — never hand-edit `[inferred]` |

### Generator permissions and external fetch

- Scopes: `read:jira-work`, `write:jira-work`, `read:jira-user`, `storage:app` `[from-config]`.
- Allowlisted backend fetch addresses: `https://matcha.harriscomputer.com` and `https://*.webtrigger.atlassian.app` `[from-config]`.
- App ID and runtime are declared in `Test Case Generator/Forge-Test-Case-Generator/manifest.yml` `[from-config]`.

### Generator storage keys

Storage uses `@forge/kvs`; the API key uses `kvs.setSecret` and is never returned to the UI `[from-docs]`.

| Key | Purpose |
| --- | --- |
| `ai_endpoint_url` | MatchaAI completions endpoint |
| `requirement_mission_id` | Mission ID for the requirements pass |
| `steps_mission_id` | Mission ID for the steps pass |
| `matcha_api_key` | Secret sent as the `MATCHA-API-KEY` header; write-only |
| `endpoint_url` | Web trigger URL of the importer app |
| `allowed_projects`, `allowed_groups`, `allowed_issue_types` | Access control lists shown in Matcha Settings |
| `tcg_generate_state_<runId>`, `tcg_submit_state_<runId>`, `tcg_finalize_plan_<runId>`, `tcg_progress_<runId>` | Transient per-run state, cleaned up after each phase |

## Component: Forge Xray Import MCP

### Importer entry points

| Entry point | File and symbol | Invoked by |
| --- | --- | --- |
| Web trigger handler | `Forge-Xray-Import-MCP/src/forge.js`, exported `handler` | Forge `webtrigger` key `xray-import-trigger`, `urlFormat: v2` `[from-config]` |
| Local CLI runner | `Forge-Xray-Import-MCP/src/index.js` | `node src/index.js <file.json>` `[from-docs]` |
| Domain logic | `Forge-Xray-Import-MCP/src/importer.js` | Both entry points `[from-docs]` |

### Importer module chain

- `src/forge.js` and `src/index.js` both delegate to `src/importer.js` `[from-docs]`.
- `src/importer.js` builds Xray GraphQL mutations and calls `src/graphql.js` `[from-docs]`.
- `src/graphql.js` obtains a bearer token from `src/auth.js`, which caches the token for 55 minutes `[from-docs]`.
- `src/config.js` reads environment variables and fails fast on missing required values `[from-docs]`.
- `src/retriever.js` serves the GET path that reads Tests back out of Xray `[inferred]`.

### Importer HTTP contract

One web trigger URL serves both operations, routed by HTTP method `[from-docs]`:

- `POST` — body is a JSON array, or a single object, of test definitions; creates Xray Tests.
- `GET` — `?issueIds=<comma-separated>` returns matching Tests; numeric values use the Xray `issueIds` filter, non-numeric values are converted to JQL; maximum 100 IDs per request.
- `GET` — optional `?fields=<comma-separated>` narrows the response; `issueId` and `issueKey` are always returned.

### Importer payload shapes

- Flat shape keys: `summary`, `projectKey`, `testType`, `description`, `priority`, `labels`, `fixVersions`, `linkedRequirements`, `folderPath`, `preconditionIssueIds`, `steps`, `gherkin`, `unstructured` `[from-docs]`.
- App-payload shape uses a Jira-style `fields` object plus `testtype`, `update.issuelinks`, and `steps` `[from-docs]`.
- Both shapes may be mixed in one batch `[from-docs]`.
- Supported test types: `Manual` (steps), `Generic` (unstructured), `Cucumber` (gherkin) `[from-docs]`.
- A worked example lives in `Forge-Xray-Import-MCP/sample-tests.json`; the full field table lives in `Forge-Xray-Import-MCP/README.md`.

### Importer permissions and external fetch

- Scope: `write:jira-work` `[from-config]`.
- Allowlisted backend fetch addresses: `https://us.xray.cloud.getxray.app`, `https://xray.cloud.getxray.app`, `https://eu.xray.cloud.getxray.app`, `https://au.xray.cloud.getxray.app` `[from-config]`.

## Component: Test Generator Agents

| File (under `Test Generator Agents/Plan/`) | Defines |
| --- | --- |
| `Requirement-Generator.md` | The Front-End Technical Requirements Analyst mission: ingests an Epic and its child Stories, emits a plain-text front-end Scope of Work with `(Source: [Ticket ID])` traceability and an omitted back-end list |
| `Test-Step-Generator.md` | The QA Test Case Logic Author mission: converts requirements into Markdown test case blocks with `TCID`, `Test Summary`, `Folder`, `Components`, `Linked Requirements`, `Test Priority`, `Test Type`, `Fix Version`, `Label`, `Description`, and a steps table |

Format contract that couples the prompts to the code:

- `Test-Step-Generator.md` emits `**TCID:**`-prefixed blocks with an `| Action | Data | Expected Result |` table; `src/functions/parseTestCases.js` parses exactly that shape `[inferred]`.
- Changing either prompt's output format without updating `parseTestCases.js` breaks generation `[inferred]`.
- `Test-Step-Generator.md` fixes `Test Priority` to `2`, `Test Type` to `Manual`, and always starts `Label` with `AI` `[from-docs]`.
- `Requirement-Generator.md` forbids Markdown styling and replaces quotation marks with asterisks `[from-docs]`.

## Commands

| Purpose | Command | Run from | Defined in | Status |
| --- | --- | --- | --- | --- |
| Lint generator backend | `npm run lint` | `Test Case Generator/Forge-Test-Case-Generator/` | `package.json` | `[from-config]` |
| Build generator backend | `npm run build` (`forge build`) | `Test Case Generator/Forge-Test-Case-Generator/` | `package.json` | `[from-config]` |
| Deploy generator | `npm run deploy` (`forge deploy`) | `Test Case Generator/Forge-Test-Case-Generator/` | `package.json` | `[from-config]` |
| Tunnel generator | `npm run tunnel` (`forge tunnel`) | `Test Case Generator/Forge-Test-Case-Generator/` | `package.json` | `[from-config]` |
| Run generator UI dev server | `npm start` | `Test Case Generator/Forge-Test-Case-Generator/static/` | `static/package.json` | `[from-config]` |
| Build generator UI bundle | `npm run build` | `Test Case Generator/Forge-Test-Case-Generator/static/` | `static/package.json` | `[from-config]` |
| Run generator UI tests | `npm test -- --watchAll=false` | `Test Case Generator/Forge-Test-Case-Generator/static/` | `static/package.json`, `CLAUDE.md` | `[from-docs]` |
| Run importer tests | `npm test` (Jest, `tests/**/*.test.js`) | `Forge-Xray-Import-MCP/` | `package.json` | `[from-config]` |
| Import a JSON file locally | `npm start <file.json>` or `node src/index.js <file.json>` | `Forge-Xray-Import-MCP/` | `package.json`, `AGENTS.md` | `[from-docs]` |
| Deploy importer | `npm run deploy` (`forge deploy`) | `Forge-Xray-Import-MCP/` | `package.json` | `[from-config]` |
| Install a Forge app on a site | `forge install -s <site>.atlassian.net -p jira` | Either app root | `Forge-Xray-Import-MCP/README.md` | `[from-docs]` |
| Create the importer web trigger URL | `forge webtrigger create --functionKey xray-import-trigger` | `Forge-Xray-Import-MCP/` | `Forge-Xray-Import-MCP/README.md` | `[from-docs]` |

Ordering requirements:

- Run the generator UI build before `forge deploy`, because the manifest serves `static/build` `[from-config]`.
- Deploy and install `Forge-Xray-Import-MCP` and create the web trigger before configuring the generator, because Matcha Settings stores that trigger URL `[from-docs]`.
- `forge login` must succeed before any `forge` subcommand `[from-docs]`.

## External dependencies and integrations

| Integration | Purpose | Config or env var names | Where configured |
| --- | --- | --- | --- |
| MatchaAI completions API | Generates requirements and test steps | `ai_endpoint_url`, `requirement_mission_id`, `steps_mission_id`, `matcha_api_key` (sent as the `MATCHA-API-KEY` header) | Matcha Settings admin page; env fallbacks `MATCHA_AI_ENDPOINT_URL`, `MATCHA_REQUIREMENT_MISSION_ID`, `MATCHA_STEPS_MISSION_ID`, `MATCHA_API_KEY` |
| Importer web trigger | Receives approved test case JSON | `endpoint_url`; env fallback `XRAY_IMPORT_ENDPOINT_URL` | Matcha Settings admin page |
| Xray Cloud GraphQL and auth APIs | Creates and reads Test issues | `XRAY_CLIENT_ID`, `XRAY_CLIENT_SECRET`, `XRAY_REGION` (`us`, `eu`, `au`), `JIRA_PROJECT_KEY`, `XRAY_LINK_TYPE_NAME` | `forge variables set --encrypt ...`; local `.env` modelled on `Forge-Xray-Import-MCP/.env.example` |
| Jira REST API | Reads ticket data, writes comments and issue links | Forge scopes only, no separate credential | `manifest.yml` of each app |

Never commit values for any variable named above; only the names belong in documentation.

## Conventions

- Register a Forge function handler by module basename: `manifest.yml` uses `forge.handler`, not `src/forge.handler` `[from-docs]`.
- Split long backend work across multiple resolver calls; a single Forge function invocation is capped near 25 seconds `[from-docs]`.
- Import Xray tests sequentially in `src/importer.js`; parallel calls trip Xray rate limits `[from-docs]`.
- Map source projects to test projects with `SOURCE_TO_TEST_PROJECT_MAP` in the generator resolver: `METERSENSE` → `COMPTEST`, `CONNECT` → `CONNECTTEST`, otherwise unchanged `[from-docs]`.
- Keep the AI Markdown contract and `parseTestCases.js` in sync whenever a prompt in `Test Generator Agents/Plan/` changes `[inferred]`.
- Store secrets with `kvs.setSecret` and expose only a `hasApiKey` boolean to the UI `[from-docs]`.
- Both apps ship agent guidance in their own `AGENTS.md`; the generator adds `CLAUDE.md`, `USER_GUIDE.md`, `Examples.md`, and `XrayInputFormat.md`.

## Definition of done

1. `npm run lint` passes in `Test Case Generator/Forge-Test-Case-Generator/` `[from-config]`.
2. `npm test -- --watchAll=false` passes in `Test Case Generator/Forge-Test-Case-Generator/static/` when frontend functions change `[from-docs]`.
3. `npm test` passes in `Forge-Xray-Import-MCP/` when importer logic changes `[from-config]`.
4. `manifest.yml` still allowlists every external address the changed code calls `[from-config]`.
5. Documentation in the changed app's `AGENTS.md` and `README.md` matches the new behavior `[inferred]`.

CI equivalent: none. Neither app contains `.github/workflows/`; `Forge-Xray-Import-MCP/.github/` holds only `agents/xray-graphql-expert.agent.md` `[inferred]`.

## Gotchas

- The Forge app service account needs `Browse Projects` and `Edit Issues` in every project involved, or cross-project links fail with "issue does not exist" `[from-docs]`.
- The requirement link type is hardcoded to `Tests`; override with `XRAY_LINK_TYPE_NAME` on sites that renamed the link type `[from-docs]`.
- `Forge-Xray-Import-MCP/README.md` embeds a live web trigger URL including its secret path segment; treat that file as sensitive and rotate the trigger if the repository is shared `[inferred]`.
- `Forge-Xray-Import-MCP/README.md` comments out the `forge register` and `forge variables set` steps, so a fresh clone has no documented uncommented setup path before `forge deploy` `[from-config]`.
- `Forge-Xray-Import-MCP/mcp.json` is an empty file and configures nothing `[inferred]`.
- `Test Case Generator/Forge-Test-Case-Generator/AGENTS.md` describes the breadcrumb as 4 steps while `README.md` describes 3 numbered stops; the two documents disagree `[inferred]`.
- Both apps keep an installed `node_modules/` directory on disk; never edit files inside it `[inferred]`.
- Access to the generator panel is restricted by an explicit user allowlist, a Jira group check, and an allowed-project check; a correctly deployed app still shows nothing to an unlisted user `[from-docs]`.
- `(None)` values produced by the AI in a step's data column are normalized to an empty string by `parseTestCases.js` `[from-docs]`.

## Where to change X

| Task | Touch these files | Then run |
| --- | --- | --- |
| Change AI prompt wording or output shape | `Test Generator Agents/Plan/*.md` plus `Test Case Generator/Forge-Test-Case-Generator/src/functions/parseTestCases.js` | `npm test -- --watchAll=false` in `static/` |
| Add a field to the generated Xray payload | `src/functions/buildXrayJson.js` and `static/src/functions/buildXrayJson.js` | `npm test -- --watchAll=false` in `static/` |
| Change the review or edit UI | `static/src/panels/ReviewPanel.jsx`, `static/src/components/TestCaseCard.jsx`, `static/src/components/StepsTable.jsx` | `npm start` in `static/`, then `npm run build` |
| Add or change an admin setting | `static/src/panels/AdminPanel.jsx` and `src/resolvers/index.js` | `npm run lint`, then `forge deploy` |
| Change how Tests are created in Xray | `Forge-Xray-Import-MCP/src/importer.js`, `src/graphql.js` | `npm test` in `Forge-Xray-Import-MCP/` |
| Change the retrieval endpoint | `Forge-Xray-Import-MCP/src/retriever.js`, `src/forge.js` | `npm test` in `Forge-Xray-Import-MCP/` |
| Allow a new external host | the relevant `manifest.yml` under `permissions.external.fetch.backend` | `forge deploy` |

## Glossary

| Term | Definition |
| --- | --- |
| Forge | Atlassian's serverless app platform; hosts both apps in this folder |
| Custom UI | A Forge UI mode that renders a bundled React app inside a Jira iframe |
| Resolver | A named backend function a Forge Custom UI calls over `@forge/bridge` |
| KVS | Forge key-value store accessed through `@forge/kvs` |
| Web trigger | A Forge module exposing a public HTTPS URL that invokes a function |
| Xray | Jira test management app whose Cloud GraphQL API stores Test issues |
| MatchaAI | Internal AI platform at `matcha.harriscomputer.com` that runs the two missions |
| Mission ID | Numeric identifier of a stored MatchaAI prompt configuration |
| TCID | Sequential test case identifier emitted by the Steps Generator mission |
| `runId` | Per-run identifier used to key transient KVS state across pipeline stages |
| SOW | Scope of Work; the plain-text requirements document the Requirement Generator emits |

## Further reading

- [`Test Case Generator/Forge-Test-Case-Generator/AGENTS.md`](<./Test Case Generator/Forge-Test-Case-Generator/AGENTS.md>): full resolver API, KVS keys, workflow stages, and validation rules.
- [`Test Case Generator/Forge-Test-Case-Generator/CLAUDE.md`](<./Test Case Generator/Forge-Test-Case-Generator/CLAUDE.md>): condensed command list and backend/frontend split.
- [`Test Case Generator/Forge-Test-Case-Generator/USER_GUIDE.md`](<./Test Case Generator/Forge-Test-Case-Generator/USER_GUIDE.md>): end-user walkthrough of every phase and the manual fallback.
- [`Test Case Generator/Forge-Test-Case-Generator/XrayInputFormat.md`](<./Test Case Generator/Forge-Test-Case-Generator/XrayInputFormat.md>): the exact JSON the generator sends to the importer.
- [`Test Case Generator/Forge-Test-Case-Generator/Matcha_API_Documentation_2026-07-29.md`](<./Test Case Generator/Forge-Test-Case-Generator/Matcha_API_Documentation_2026-07-29.md>): MatchaAI authentication, completions, and file APIs.
- [`Forge-Xray-Import-MCP/README.md`](./Forge-Xray-Import-MCP/README.md): deployment steps, request examples, and the full field reference.
- [`Forge-Xray-Import-MCP/AGENTS.md`](./Forge-Xray-Import-MCP/AGENTS.md): module chain, environment variables, and Xray pitfalls.
- [`Forge-Xray-Import-MCP/sample-tests.json`](./Forge-Xray-Import-MCP/sample-tests.json): canonical import payload example.

## Unknowns and unverified claims

| Claim or question | Status | How to resolve |
| --- | --- | --- |
| Purpose of the empty `Test Generator Agents/Code/` directory | `[unknown]` | Ask the folder owner whether MatchaAI mission code belongs there or the directory is obsolete |
| Whether `Forge-Xray-Import-MCP/mcp.json` should hold MCP server configuration | `[unknown]` | Ask the importer maintainer; the file is currently zero bytes |
| Current MatchaAI mission IDs for requirements and steps | `[unknown]` | Read the deployed Matcha Settings admin page; values are not stored in the repositories |
| Whether the deployed apps are installed on more sites than `harris-smartworks.atlassian.net` | `[unknown]` | Run `forge install list` for each app |
| Whether `static/` tests currently pass | `[unknown]` | Run `npm test -- --watchAll=false` in `Test Case Generator/Forge-Test-Case-Generator/static/` |
| Whether `Forge-Xray-Import-MCP/tests/importer.test.js` covers the GET retrieval path | `[unknown]` | Read the test file and compare against `src/retriever.js` |
| Breadcrumb step count disagreement between the generator's `README.md` and `AGENTS.md` | `[unknown]` | Read `static/src/components/Breadcrumb.jsx` and correct the stale document |
| Release and versioning process for either Forge app | `[unknown]` | Ask the maintainers; no CI workflow or release script exists in either repository |
| Which model backs the MatchaAI missions — the previous version of this file stated Gemini | `[unknown]` | Check the mission configuration in the MatchaAI dashboard; no file in this folder names a model |