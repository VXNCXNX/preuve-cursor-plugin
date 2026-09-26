# Preuve

Cursor plugin that connects agents to [Preuve](https://preuve.ai) through Preuve's official remote [Model Context Protocol](https://modelcontextprotocol.io/) server.

Score a startup idea against live market evidence without leaving Cursor. The agent gets the same analysis as preuve.ai: a viability score, competitors, risks, and source-linked evidence.

## Install

1. Open **Cursor Settings → Plugins**.
2. Search for **Preuve**.
3. Click **Install**, then complete the Preuve sign-in prompt.

Or run `/add-plugin preuve` in chat.

## MCP

```json
{
  "mcpServers": {
    "preuve": {
      "type": "http",
      "url": "https://mcp.preuve.ai/mcp"
    }
  }
}
```

Auth is OAuth. Cursor opens Preuve sign-in the first time the plugin connects, and the consent screen creates an API key for this connection. There is no API key or client ID to configure.

## Before you connect

You need a Preuve account. Start free: connect and run your first starter scans.

## What agents can do

| Tool              | What it does                                                                                                                               |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `start_analysis`  | Validate a startup idea. A starter scan is the default. A deep scan runs only when you ask for one                                         |
| `get_analysis`    | Check whether a run has finished, with links to the report                                                                                 |
| `enrich_analysis` | Deepen a deep report with founder fit, proof of demand, a launch playbook, trends, or community signals                                    |
| `export_analysis` | Get a finished report as structured data: scores, verdict, risks, and competitors. Deep reports add market sizing, sections, and citations |
| `generate_ideas`  | Generate startup ideas from your interests                                                                                                 |
| `get_agency`      | Read a Consultant or Agency workspace: its project slots and client reports                                                                |

The server also offers two prompts, `validate_idea` and `generate_ideas`, and the plugin bundles the `preuve-agent-api` skill. The skill teaches the agent the workflow: which scan type to start, how to poll a run, and how to retry without starting a second run. The hosted server is the source of truth for tool names and schemas.

## Notes

- Tool calls run as the Preuve account that approved the connection.
- A deep scan draws on that account's scan quota. The agent starts a starter scan unless you ask for a deep one.
- The key appears as **Cursor (OAuth)** in [Account → API Keys](https://preuve.ai/app?settings=apiKeys), where you can revoke it. Reconnecting replaces it.
- If you already have a Preuve API key, you can skip OAuth and send it as `"headers": {"Authorization": "Bearer prv_..."}` instead.

## Docs

- Preuve MCP server: https://preuve.ai/mcp
- Setup guide: https://docs.preuve.ai/mcp-server
- Agent skill: https://docs.preuve.ai/agent-skill
- Privacy policy: https://preuve.ai/privacy
- Support: support@preuve.ai

## License

MIT
