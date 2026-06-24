# Grocery Intelligence Agent

An agentic business-intelligence portfolio project for retail grocery inventory optimization. It identifies slow-moving SKUs, recommends reorder points, and suggests clearance actions, then improves future recommendations from human feedback.

## Project plan

The phased architecture, responsibilities, acceptance criteria, and zero-cost setup approach are in [PROJECT_BRIEF.md](PROJECT_BRIEF.md).

## Stack

- DynamoDB for operational inventory data
- PostgreSQL + pgvector for feedback retrieval
- LlamaIndex orchestration with local Ollama models
- MCP servers for inventory, feedback, and output integrations

The default path runs locally with Docker and Ollama; optional cloud and paid-model integrations remain disabled by default.
