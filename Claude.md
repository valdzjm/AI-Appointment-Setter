# Crypto Automation — Claude Code Project

## Project Purpose

This project is a prompt-driven n8n workflow builder. The user describes an automation in natural language, and Claude uses the available n8n skills and n8n MCP tools to build, configure, and deploy workflows directly into the user's n8n instance. The focus is on powerful, production-quality crypto and financial automations.

## How to Work in This Project

- **Always consult the relevant n8n skill before calling any n8n MCP tool.** The skills prevent common mistakes (wrong node types, bad parameter formats, expression errors).
- **Skill usage order for building workflows:**
  1. `n8n-mcp-tools-expert` — before any MCP tool call
  2. `n8n-workflow-patterns` — when designing a new workflow
  3. `n8n-node-configuration` — when configuring individual nodes
  4. `n8n-expression-syntax` — when writing expressions to pass data between nodes
  5. `n8n-code-javascript` — when a Code node is needed
  6. `n8n-validation-expert` — when interpreting validation errors
- **Build iteratively:** create the workflow skeleton first, then configure nodes, then validate, then activate.
- **Validate before activating** any workflow using the MCP validation tools.
- **Never guess node types** — search for them via MCP tools first.

## Workflow Design Principles

- Prefer webhook triggers for event-driven crypto automations (price alerts, trade signals).
- Use scheduled triggers (cron) for periodic tasks (portfolio snapshots, rebalancing checks).
- Keep credentials referenced by name, never hard-coded in node parameters.
- Add error handling branches on critical paths (exchange API calls, trade execution).
- Use Code nodes only when no built-in node can accomplish the task.

## Domain Context

- Primary focus: cryptocurrency trading, portfolio tracking, price monitoring, and signal automation.
- Exchange integrations will typically go through HTTP Request nodes hitting exchange APIs (Binance, Coinbase, Kraken, etc.) or dedicated community nodes if available.
- Sensitive values (API keys, secrets) must always be stored as n8n credentials, never as plain text in workflow parameters.

## n8n Instance

- **URL:** `https://your-n8n-instance.com` (set in `.env`)
- **API key:** stored in `.env` and mirrored into `.mcp.json` (do not hard-code in workflows or node parameters)

## MCP & Skills Available

- **n8n MCP tools** — direct access to the user's n8n instance (create, update, activate workflows; manage credentials; search nodes)
- **Skills:** `n8n-mcp-tools-expert`, `n8n-workflow-patterns`, `n8n-node-configuration`, `n8n-expression-syntax`, `n8n-code-javascript`, `n8n-code-python`, `n8n-validation-expert`
