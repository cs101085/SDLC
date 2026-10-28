# SDLC-Orchestration

Kestra-based orchestration for the AI-DLC pipeline. Runs the SDLC stages as Kestra flows, coordinating copilotCLI containers and persisting artifacts to [SDLC-Artifacts](https://github.com/Harris-SmartWorks/SDLC-Artifacts).

## Prerequisites

- Docker/Podman
- Git
- A `.env` file (copy from `.env.example` and fill in all required values and tokens)

## Kestra version

Pinned to **v1.3.31** by digest in the `Dockerfile`, with only two plugins:
`plugin-docker`
`plugin-script-shell`

The image also adds `git`, `jq` and `yq`. `yq` is pinned by version and by a SHA-256 per architecture (`amd64` and `arm64`), so the same `Dockerfile` builds natively on Windows, Linux and Apple Silicon Macs. When bumping `YQ_VERSION`, update both checksums from the release's `checksums` file.

The UI is at `http://localhost:8080/ui/main` and the REST API at `http://localhost:8080/api/v1/main/...`.

## Quick Start

### Windows + Docker

```powershell
# 1. Create .env from the template and fill in all values
Copy-Item .env.example .env
# ...edit .env...

# 2. Log into GHCR and pull the AI container image
.\scripts\dev\setup.ps1

# 3. Build the custom Kestra image (adds git + jq)
docker compose build

# 4. Start Kestra + Postgres
docker compose up -d

# 5. Open the Kestra UI
Start-Process http://localhost:8080
```

### Linux + Podman

On Podman-based hosts, use `podman-compose.yml` instead:

```bash
sudo systemctl start podman.socket
podman-compose -f podman-compose.yml build
podman-compose -f podman-compose.yml up -d
```

Then deploy the flows once:

```bash
scripts/deploy/update.sh
```

From then on the server deploys itself from git (see below). `update.sh` is only needed again when a change touches the `Dockerfile` or a compose file.

## Loading flows

How flows reach Kestra depends on `FLOW_WATCH`, which each compose file defaults differently.

**Development** (`FLOW_WATCH=true`): flows load automatically from `_flows/` on startup and reload whenever saved, without restarting Kestra.

**Production** (`FLOW_WATCH=false`): flows are loaded as they are updated in git. Every five minutes `system.gitops-sync` (`scripts/deploy/gitops-sync/`) checks the branch the checkout is on (or `GITOPS_BRANCH`) and, when it has moved:

1. validates every flow in the new commit, and deploys nothing if one fails
2. raises a drain flag, which makes `master-sdlc` refuse new runs
3. waits until no `ai-dlc` execution is still in flight, including paused ones
4. fast-forwards the checkout and replaces every flow in one update per namespace
5. lowers the flag to allow runs to continue

A run that fails needs a person, and its log says why: local edits on the server, a rewritten branch, an invalid flow, or a change to the image or compose file, which `update.sh` applies. A deploy that fails part-way is rolled back to the previous commit. Merge to the branch; do not edit files on the server or flows in the UI.

## Triggering flows by API

**Endpoint:** `POST /api/v1/main/executions/ai-dlc/master-sdlc`  
**Auth:** HTTP Basic — username and password from `.env`  
**Content-Type:** `multipart/form-data`

```bash
# Full pipeline run
curl -s -u "$USER:$PASSWORD" -X POST \
  "$KESTRA_URL/api/v1/main/executions/ai-dlc/master-sdlc" \
  -F ticketId=ABC-123

# Run a stage range, continuing from a prior execution's artifacts
curl -s -u "$USER:$PASSWORD" -X POST \
  "$KESTRA_URL/api/v1/main/executions/ai-dlc/master-sdlc" \
  -F ticketId=ABC-123 -F 'initialStage=["Design"]' -F 'finalStages=["Review"]' \
  -F previousExecutionPath=Artifacts/ABC-123/20260901_120000

# Stop after both halves of a parallel pair — stage lists are JSON arrays
curl -s -u "$USER:$PASSWORD" -X POST \
  "$KESTRA_URL/api/v1/main/executions/ai-dlc/master-sdlc" \
  -F ticketId=ABC-123 -F 'finalStages=["Review","Verification"]'

# Pause for human approval after specific stages
curl -s -u "$USER:$PASSWORD" -X POST \
  "$KESTRA_URL/api/v1/main/executions/ai-dlc/master-sdlc" \
  -F ticketId=ABC-123 -F 'humanGatedStages=["Analysis","Design"]'

```

| Input | Default | Description |
|---|---|---|
| `ticketId` | _(required)_ | Jira ticket key, e.g. `ABC-123` |
| `initialStage` | _(derived)_ | Stage(s) to begin at; empty derives it from the ticket type |
| `finalStages` | `Release` | Last stage(s) to run. Nothing downstream of a named stage runs, so naming one half of a parallel pair ends that branch and never starts its sibling — name both to run both |
| `previousExecutionPath` | _(none)_ | An earlier run of the same ticket to continue from, e.g. `Artifacts/ABC-123/20260901_120000`. Its artifacts are copied into the new run |
| `skipConfidenceCheck` | `First` | Which stages skip their confidence check: `False` none, `First` only the starting stage, `True` all |
| `humanGatedStages` | `[]` | Stages to pause after for human approval |
| `requestorEmail` | `""` | Recipient of the outcome email; also recorded with the run |
| `dryRun` | `false` | Run in full without publishing anything externally. See [Dry runs](#dry-runs) |

Stage values are the stage names in [_flows/stages/README.md](_flows/stages/README.md).

A stage named in `humanGatedStages` pauses once it finishes. Find the paused execution and resume it to continue the run:

```bash
# List paused executions
curl -s -u "$USER:$PASSWORD" \
  "$KESTRA_URL/api/v1/main/executions/search?filters[namespace][EQUALS]=ai-dlc&filters[state][EQUALS]=PAUSED"

# Resume one by its id
curl -s -u "$USER:$PASSWORD" -X POST \
  "$KESTRA_URL/api/v1/main/executions/<executionId>/resume"
```

### Starting mid-pipeline

To start a run mid-pipeline (ie. to test a single stage in isolation), you must supply its required artifacts as Jira attachments.

```bash
curl -s -u "$USER:$PASSWORD" -X POST \
  "$KESTRA_URL/api/v1/main/executions/ai-dlc/master-sdlc" \
  -F ticketId=ABC-123 \
  -F 'initialStage=["CodeGeneration"]' -F 'finalStages=["CodeGeneration"]' \
  -F skipConfidenceCheck=False -F dryRun=true
```

- Attach one file per `as:` entry in the stage's `resources`, named as the `as:` value —
  `Design`: `analysis.md`; `CodeGeneration`: `design.md`, `implementation.md`.
- Text files and PDFs only, 1 MiB each, ten per ticket.
- Omit `previousExecutionPath`.
- An attachment is ignored if the stage that produces a same-named artifact is itself in the run, so an attachment cannot displace a real stage output. It will not be available to earlier stages as a result.
- Uptake is logged by `resolve-inputs`: "taken from Jira attachments".

### Dry runs

`dryRun=true` runs the pipeline in full and publishes nothing externally.

Note, that operations required for a subsequent stage to perform are still run.

Dry runs skip the following:

- Jira comments, status transitions and field updates.
- The pull request Review would open.
- The outcome email.

Dry runs still execute the following:

- The source is cloned and every agent runs.
- Code Generation pushes its feature branch.
- Artifacts are committed to SDLC-Artifacts.
- Human gates pause.

## Notifications

Emailed to `requestorEmail` and `ADMIN_EMAILS` when a run ends:

- **Success**
- **Blocked** — a confidence check stopped the run; names the stage and its rating.
- **Failure** — names the stage and the underlying error.

Configured by `SMTP_HOST`, `SMTP_PORT`, `SMTP_FROM`, `ADMIN_EMAILS` and `ENABLE_EMAIL_UPDATES` in
`.env`. Empty `SMTP_HOST` disables email.

## Repository Structure

```
_flows/
  main_master.yml               # orchestrates the full pipeline; dispatches stages as they become ready
  orchestration/                # sets up an execution and frames every stage
  stages/                       # one folder per lifecycle stage, owned by its SDLC champion
  util/
    agents/                     # fetch agent config and launch copilotCLI containers
    artifacts/                  # workspace lifecycle: prepare, fetch, update, and publish
    git/                        # low-level git primitives (clone, sparse-checkout)
    bitbucket/                  # resolve Bitbucket project variables
    jira/                       # post status updates back to Jira
  maintenance/                  # system flows that remove finished runs' working files
_config/                        # per-project Bitbucket settings and the Kestra server config
scripts/
  dev/                          # local dev helper scripts
  deploy/                       # server deployment scripts to clean update and restart
Dockerfile
docker-compose.yml
podman-compose.yml
.env.example
```

Each stage folder holds a `main_*.yml` flow plus shell scripts to separate orchestration from specific logical operations. Flow files are prefixed `main_` so that Kestra loads them automatically (see [Loading flows](#loading-flows)). Utility subflows extract repeated or distinct operations from the stages.

## Artifact Storage

Artifacts are persisted to the [SDLC-Artifacts](https://github.com/Harris-SmartWorks/SDLC-Artifacts) repo under `Artifacts/<TICKET>/<EXEC_ID>/`. Each stage follows the same pattern:

1. **Fetch** — sparse-checkout the execution path into a local working directory
2. **Work** — run the stage, write outputs locally
3. **Publish** — commit and push the local directory back to SDLC-Artifacts

This makes every stage independently runnable and resumable. When a run names a `previousExecutionPath`, that run's artifacts are copied into the new one before the first stage begins.

## Agents

Agents are defined in [AIAgents](https://github.com/Harris-SmartWorks/AIAgents). Each stage maps to a named agent subfolder:

| Phase | Agent |
|---|---|
| Jira Analysis | `jira-reviewer` |
| Design | `solution-designer` |
| Code Generation / Test Generation | `code-generator` |

At runtime, `agents.run-container` sparse-checks out the relevant agent subfolder from AIAgents so the copilotCLI container can read it.
