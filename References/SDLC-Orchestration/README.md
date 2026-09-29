# SDLC-Orchestration

Kestra-based orchestration for the AI-DLC pipeline. Runs the five SDLC phases (Jira Analysis → Design → Code Generation → Test Generation → Review) as Kestra flows, coordinating copilotCLI containers and persisting artifacts to [SDLC-Artifacts](https://github.com/Harris-SmartWorks/SDLC-Artifacts).

## Prerequisites

- Docker/Podman
- Git
- A `.env` file (copy from `.env.example` and fill in all required values and tokens)

## Kestra version

Pinned to **v1.3.31** by digest in the `Dockerfile`, with only two plugins:
`plugin-docker`
`plugin-script-shell`

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

To pull the latest code onto a running server:

```bash
scripts/deploy/update.sh
```

## Loading flows

Flows load automatically from `_flows/` on startup and reload when saved, without restarting Kestra. A watcher reads files prefixed **`main_*.yml`**, so any flow without this prefix won't load.

## Triggering flows by API

**Endpoint:** `POST /api/v1/main/executions/ai-dlc/master-sdlc`  
**Auth:** HTTP Basic — username and password from `.env`  
**Content-Type:** `multipart/form-data`

```bash
# Full pipeline run
curl -s -u "$USER:$PASSWORD" -X POST \
  "$KESTRA_URL/api/v1/main/executions/ai-dlc/master-sdlc" \
  -F ticketId=ABC-123

# Run a phase range, or resume from a phase (prior artifacts are inherited automatically)
curl -s -u "$USER:$PASSWORD" -X POST \
  "$KESTRA_URL/api/v1/main/executions/ai-dlc/master-sdlc" \
  -F ticketId=ABC-123 -F startPhase=Design -F endPhase=Review

# Pause for human approval after specific phases — repeat the field once per phase
curl -s -u "$USER:$PASSWORD" -X POST \
  "$KESTRA_URL/api/v1/main/executions/ai-dlc/master-sdlc" \
  -F ticketId=ABC-123 -F humanGatedPhases=JiraAnalysis -F humanGatedPhases=Design

```

| Input | Default | Description |
|---|---|---|
| `ticketId` | _(required)_ | Jira ticket key, e.g. `ABC-123` |
| `startPhase` | `JiraAnalysis` | First phase to run; prior-phase artifacts are inherited |
| `endPhase` | `Review` | Last phase to run; later phases are skipped |
| `previousExecutionPath` | _(auto)_ | Override the specific prior execution to inherit from |
| `enableJiraUpdates` | `false` | Post comments and status transitions to Jira |
| `enableTestGeneration` | `true` | Set to `false` to skip test generation |
| `humanGatedPhases` | `[]` | Phases to pause after for human approval; repeat `-F` once per phase |

Valid phase values: `JiraAnalysis` · `Design` · `CodeGeneration` · `TestGeneration` · `Review`

## Repository Structure

```
_flows/
  main_master.yml               # orchestrates the full pipeline; triggers phases in sequence
  phases/
    jira-analysis/              # analyze ticket; score confidence and extract requirements
    design/                     # produce a technical design document
    code-generation/            # generate code and commit it to a feature branch
    test-generation/            # generate tests for the produced code
    review/                     # validate the branch and open a pull request
  util/
    agents/                     # fetch agent config and launch copilotCLI containers
    artifacts/                  # workspace lifecycle: prepare, fetch, update, and publish
    git/                        # low-level git primitives (clone, sparse-checkout)
    bitbucket/                  # resolve Bitbucket project variables
    jira/                       # post status updates back to Jira
scripts/
  dev/                          # local dev helper scripts
  deploy/                       # server deployment scripts to clean update and restart
Dockerfile
docker-compose.yml
podman-compose.yml
.env.example
```

Each phase folder holds a `main_*.yml` flow plus shell scripts to separate orchestration from specific logical operations. Flow files are prefixed `main_` so that Kestra loads them automatically (see [Loading flows](#loading-flows)). Utility subflows extract repeated or distinct operations from the main phases.

## Artifact Storage

Artifacts are persisted to the [SDLC-Artifacts](https://github.com/Harris-SmartWorks/SDLC-Artifacts) repo under `Artifacts/<TICKET>/<EXEC_ID>/`. Each phase follows the same pattern:

1. **Fetch** — sparse-checkout the execution path into a local working directory
2. **Work** — run the phase, write outputs locally
3. **Publish** — commit and push the local directory back to SDLC-Artifacts

This makes every phase independently runnable and resumable. On a resume, `artifacts.prepare-workspace` copies artifacts from a prior execution before the new phase begins.

## Agents

Agents are defined in [AIAgents](https://github.com/Harris-SmartWorks/AIAgents). Each phase maps to a named agent subfolder:

| Phase | Agent |
|---|---|
| Jira Analysis | `jira-reviewer` |
| Design | `solution-designer` |
| Code Generation / Test Generation | `code-generator` |

At runtime, `agents.run-container` sparse-checks out the relevant agent subfolder from AIAgents so the copilotCLI container can read it.
