# AIAgents

Central repository for reusable AI configuration content:
- Agent definitions
- Instruction markdowns
- Skill markdowns

This repository is for general-purpose AI configurations that are not tied to a single application codebase.

Each top-level folder other than `.github/` and `schemas/` is a self-contained **agent pack** that AIOrchestration resolves and runs in a container.
All paths in this document are relative to the repository root.

## Repository Scope

Use this repository for:
- Reusable and broadly applicable agent configurations
- Shared instruction and skill files that can be applied across projects
- Experiments and patterns that are portable between repositories

Do not use this repository for:
- Repository-specific agent configurations (store those in the target codebase)
- Personal or environment-specific MCP settings, endpoints, or credentials

Agent-scoped MCP configuration that a pack requires in order to function is committed alongside that pack and listed in its manifest (see `jira-reviewer/mcp-config.json`).
Never commit tokens, API keys, passwords, or credential-bearing URLs in any file.

## Agent Pack Structure

Use the table below as the structural template when creating a new agent pack.

| Path | Required | Purpose |
| --- | --- | --- |
| `<agent-name>/manifest.json` | Yes | Runtime descriptor resolved by AIOrchestration |
| `<agent-name>/agents/<agent-name>.agent.md` | Yes | Agent persona, responsibilities, and constraints |
| `<agent-name>/instructions/<agent-name>.instructions.md` | No | Operational workflow and quality bar for the agent |
| `<agent-name>/instructions/<artifact>-contract.md` | No | Canonical format contract for one input or output artifact |
| `<agent-name>/skills/<skill-name>/SKILL.md` | No | Reusable procedure the agent can invoke |
| `<agent-name>/mcp-config.json` | No | MCP servers the agent pack depends on |
| `<agent-name>/documentation/examples/<case>/...` | No | Sample input payloads for local runs and testing |

The minimum viable pack is `manifest.json` plus one `agents/*.agent.md` file.
`ping-test/` is that minimum and is the smallest reference to copy when replicating structure only.

### Naming Conventions

- Agent folder name is kebab-case and matches `agent.name` in `manifest.json`.
- Agent definition file is named `<agent-name>.agent.md` and lives under `agents/`.
- Agent definition frontmatter uses `name: <agent-name>-agent`.
- Instruction file is named `<agent-name>.instructions.md` and lives under `instructions/`.
- Contract files are named `<artifact>-contract.md`, where `<artifact>` matches the markdown file the contract governs (for example `design-contract.md` governs `design.md`).
- Skill folders are kebab-case and the skill file is always named exactly `SKILL.md`.

### Agent Definition File Shape

Every `agents/*.agent.md` file in this repository starts with YAML frontmatter containing `name` and `description`, followed by top-level headings.

Present in every agent definition:
- `# Role`
- `# Responsibilities`
- `# Constraints`

Present in the artifact-producing agent definitions (`jira-reviewer`, `solution-designer`, `code-generator`) and absent from `ping-test`:
- `# Required Output Structure`
- `# Behaviour Guidelines`

### Instruction File Shape

Every `instructions/<agent-name>.instructions.md` file starts with YAML frontmatter containing `applyTo: "**"`.
Contract files under `instructions/` carry no frontmatter.

## Manifest Requirements

Every agent folder must include a `manifest.json` at the fixed path `/<agent-name>/manifest.json`.
This is required by AIOrchestration, which resolves each agent at this exact location.

Schema definition:
- `schemas/agent-manifest-v1.schema.json`

All fields below are required by the schema. The schema sets `additionalProperties: false` on every object, so undeclared fields cause validation failure.

| Field | Type | Constraint |
| --- | --- | --- |
| `schema_version` | string | Must be exactly `"1.0"` |
| `agent.name` | string | Matches `^[a-z0-9][a-z0-9-_/]*$`, 1-128 characters |
| `agent.description` | string | 1-1000 characters |
| `container.image` | string | 1-512 characters |
| `files` | array of string | Non-empty, unique entries, must contain `manifest.json` |

`container.name` is the only optional field, and every pack in this repository sets it to the agent folder name.
All four packs currently use the image `ghcr.io/harris-smartworks/copilot-cli:latest`.

Template manifest, matching `ping-test/manifest.json`:

```json
{
  "schema_version": "1.0",
  "agent": {
    "name": "ping-test",
    "description": "Deterministic test agent that outputs exactly pong"
  },
  "container": {
    "name": "ping-test",
    "image": "ghcr.io/harris-smartworks/copilot-cli:latest"
  },
  "files": [
    "agents/ping-test.agent.md",
    "manifest.json"
  ]
}
```

The `files` array must list every file in that agent folder, using paths relative to the agent folder.
When files are added, renamed, or removed under an agent folder, update that agent's `manifest.json` so `files` stays accurate.

## Runtime Path Conventions

Agent packs that read and write artifacts use these container paths.

| Path | Used by | Role |
| --- | --- | --- |
| `/work/resources/Jira/input.json` | `jira-reviewer` | Ticket JSON input |
| `/work/resources/inputs/artifacts/<name>.md` | `solution-designer`, `code-generator` | Preferred upstream artifact input |
| `/work/resources/artifacts/<name>.md` | `solution-designer`, `code-generator` | Fallback artifact input |
| `/work/resources/inputs/source` | `solution-designer`, `code-generator` | Target source repository under review |
| `/work/resources/output/<name>.md` | All artifact-producing packs | Final markdown artifact |

## Output Contract Conventions

Artifact-producing packs in this repository share the following output rules, each defined in that pack's contract files.

- The first heading of every output file is `# Status`, followed by a `STATUS_CODE:MESSAGE` line.
- The Status section includes `Confidence: <percentage>%`.
- Downstream orchestration parses the `# Status` heading and fails the phase when the heading is absent, renamed, or demoted to a subsection.
- Output files are overwritten completely on each run and never appended to.
- Failure outcomes still write the output file, containing only the `# Status` heading and the `STATUS_CODE:MESSAGE` line.
- Full reasoning and traceability go to STDOUT and are excluded from the output markdown.

## Current Agent Packs

This repository contains four agent packs: `jira-reviewer/`, `solution-designer/`, `code-generator/`, and `ping-test/`.

### Jira Reviewer
Path: `jira-reviewer/`
Manifest: `jira-reviewer/manifest.json`

Purpose:
- Jira-focused ticket analysis workflows for implementation readiness reviews

Contents:
- `jira-reviewer/agents/jira-reviewer.agent.md`
- `jira-reviewer/instructions/jira-reviewer.instructions.md`
- `jira-reviewer/instructions/output-contract.md`
- `jira-reviewer/skills/ticket-analysis/SKILL.md`
- `jira-reviewer/mcp-config.json`
- `jira-reviewer/documentation/examples/{clean,compass,vuln}/Jira/input.json`

Primary behavior:
- Reads Jira ticket JSON data from `/work/resources/Jira/input.json`
- Incorporates description, comments, design, requirements, status, and ticket age
- Reads `attachments[]` metadata, and reads attachment bytes from `/work/resources/Jira/<localPath>` when `contentIncluded` is true
- Weighs `pullRequests[]` by status, treating MERGED as possible prior resolution, DECLINED as rejected prior art, and OPEN as unresolved concerns
- Produces a structured analysis report at `/work/resources/output/analysis.md`
- Uses `jira-reviewer/instructions/output-contract.md` as the canonical output contract
- Writes a top output status heading in STATUS_CODE:MESSAGE format
- Adds Status fields for Confidence (%) and Code Change Required
- Uses standardized output sections: Status, Summary, Details, Insights, and Risk Analysis
- Keeps the Status confidence value identical to the Risk Analysis confidence value
- Uses web search fallback when MCP evidence is insufficient
- Provides evidence and source links for external findings
- Emits full reasoning and traceability to STDOUT (not in output markdown)

Status codes:
- `SUCCESS`, `JSON_NOT_FOUND`, `JSON_MALFORMED`
- `JSON_NOT_FOUND` and `JSON_MALFORMED` outputs contain only the Status section

MCP integration:
- `jira-reviewer/mcp-config.json` includes `io.github.pipeworx-io/nvd` via streamable-http
- Endpoint: `https://gateway.pipeworx.io/nvd/mcp`

Sample payload:
- `jira-reviewer/documentation/examples/clean/Jira/input.json`
- `jira-reviewer/documentation/examples/compass/Jira/input.json`
- `jira-reviewer/documentation/examples/vuln/Jira/input.json`

### Solution Designer
Path: `solution-designer/`
Manifest: `solution-designer/manifest.json`

Purpose:
- Generates design and implementation documentation from an existing AI analysis and source code snapshot

Contents:
- `solution-designer/agents/solution-designer.agent.md`
- `solution-designer/instructions/solution-designer.instructions.md`
- `solution-designer/instructions/analysis-contract.md`
- `solution-designer/instructions/design-contract.md`
- `solution-designer/instructions/implementation-contract.md`

Primary behavior:
- Reads issue analysis from `/work/resources/inputs/artifacts/analysis.md` (fallback: `/work/resources/artifacts/analysis.md`)
- Validates analysis shape using `solution-designer/instructions/analysis-contract.md`
- Reviews source code under `/work/resources/inputs/source` and applies repository-level instructions/skills
- Writes design specification output to `/work/resources/output/design.md`
- Writes implementation guide output to `/work/resources/output/implementation.md`
- Uses `solution-designer/instructions/design-contract.md` and `solution-designer/instructions/implementation-contract.md` as canonical contracts
- Begins both output files with a mandatory `# Status` heading that a downstream orchestration gate parses
- Adds `Issue Confirmed In Code: Yes|No|Unknown` to the Status section of `design.md`
- Excludes analysis Status section from design derivation and emits full reasoning to STDOUT

### Code Generator
Path: `code-generator/`
Manifest: `code-generator/manifest.json`

Purpose:
- Generates source-code changes from an existing implementation guide, with optional design context when needed

Contents:
- `code-generator/agents/code-generator.agent.md`
- `code-generator/instructions/code-generator.instructions.md`
- `code-generator/instructions/code-summary-contract.md`
- `code-generator/instructions/design-contract.md`
- `code-generator/instructions/implementation-contract.md`

Primary behavior:
- Reads implementation input from `/work/resources/inputs/artifacts/implementation.md` (fallback: `/work/resources/artifacts/implementation.md`)
- Uses design input from `/work/resources/inputs/artifacts/design.md` only when more context is required (fallback: `/work/resources/artifacts/design.md`)
- Validates input structure using `code-generator/instructions/implementation-contract.md` and `code-generator/instructions/design-contract.md`
- Applies code changes only under `/work/resources/inputs/source`
- Applies repository-level instructions/skills from the target source repository before making edits
- Ignores Status sections when deriving or applying code changes
- Produces a structured code-change summary at `/work/resources/output/code.md`
- Uses `code-generator/instructions/code-summary-contract.md` as the canonical summary contract
- Begins `code.md` with a mandatory `# Status` heading that a downstream orchestration gate parses
- Emits full reasoning and traceability to STDOUT (not in output markdown)

Status codes:
- `SUCCESS`, `DESIGN_NOT_FOUND`, `IMPLEMENTATION_NOT_FOUND`, `SOURCE_NOT_FOUND`

### Ping Test
Path: `ping-test/`
Manifest: `ping-test/manifest.json`

Purpose:
- Deterministic smoke-test agent for orchestration and runtime wiring
- Smallest complete example of the required pack structure

Contents:
- `ping-test/agents/ping-test.agent.md`

Primary behavior:
- Returns exactly `pong` and nothing else

## Contributing

When adding or updating content:
- Keep configurations reusable and project-agnostic
- Place project-specific behavior in the destination repository instead
- Keep personal and environment-specific MCP settings out of this repository
- Include and maintain `manifest.json` for every agent folder
- Include at least one agent definition file at `agents/*.agent.md` for every agent folder
- Keep `manifest.json` compatible with `schemas/agent-manifest-v1.schema.json`
- Keep each manifest's `files` list synchronized with the actual files in that agent folder
- Update this `README.md` in the same change set whenever agent folder content changes

## AI Guidance Files

- Repository-wide policy: `AGENTS.md`
- Copilot instructions: `.github/copilot-instructions.md`
- VS Code instructions file: `.github/instructions/manifest-governance.instructions.md`
- Repository skill: `.github/skills/manifest-governance/SKILL.md`

These files enforce:
- Review `schemas/agent-manifest-v1.schema.json` before changing any agent folder content
- Keep manifest presence and schema compliance for every agent
- Keep each manifest `files` list synchronized with the agent folder contents
- Update impacted `README.md` files whenever agent content changes
