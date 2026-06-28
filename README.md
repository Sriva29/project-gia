# Grocery Intelligence Agent

An agentic business-intelligence portfolio project for retail grocery inventory optimization. It identifies slow-moving SKUs, recommends reorder points, and suggests clearance actions, then improves future recommendations from human feedback.

## Team

- **Srivatsan Rangarajan** ([@Sriva29](https://github.com/Sriva29)) — agent logic, LlamaIndex orchestration, MCP servers, and recommendation quality
- **Beula** ([@BeulaEvangelin](https://github.com/BeulaEvangelin)) — data infrastructure, DynamoDB and PostgreSQL/pgvector, parallel ETL, Slack, and dashboard integrations

## Project plan

The phased architecture, named ownership, acceptance criteria, and zero-cost setup approach are in [PROJECT_BRIEF.md](PROJECT_BRIEF.md). Phase 1 tasks are maintained as GitHub Issues under the **Phase 1 — Core infrastructure and agent MVP** milestone; the brief provides the durable technical context behind those issues.

## How we manage the work

- **GitHub Issues** are the source of truth for individual, reviewable tasks. Each issue has one accountable assignee, a phase milestone, area labels, acceptance criteria, and a linked pull request.
- **GitHub Milestones** group issues by delivery phase and make the Phase 1 scope visible.
- **GitHub Project** visualizes issue flow: Backlog → Ready → In progress → In review → Done. It is a planning view, not a second task list.
- **PROJECT_BRIEF.md** holds architecture decisions, phase goals, and the larger delivery plan. Update it through a pull request when the project direction changes.

Beula’s Phase 1 issues are assigned directly to her GitHub account, so ownership lives in GitHub Issues rather than in duplicated README checklists.

## Stack

- DynamoDB for operational inventory data
- PostgreSQL + pgvector for feedback retrieval
- LlamaIndex orchestration with local Ollama models
- MCP servers for inventory, feedback, and output integrations

The default path runs locally with Docker and Ollama; optional cloud and paid-model integrations remain disabled by default.
