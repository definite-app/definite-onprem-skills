# Definite on-prem skills for Claude Code

A [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) of
skills for using a self-hosted (on-prem) [Definite](https://www.definite.app)
deployment from Claude Code, Claude Cowork, Claude Desktop, and other AI agents.

> Running Definite Cloud instead? Use [`definite-app/claude-skills`](https://github.com/definite-app/claude-skills).

## What's in here

| Plugin | What it does |
|--------|--------------|
| `definite-onprem` | Adds the `definite-onprem` skill: query data, search the ontology and semantic layer, build data apps, inspect integrations, and run automations through your deployment's MCP server. The skill loads automatically when a request looks data-related, so you never have to say "use the Definite connector." |

## Install (individual)

```bash
# 1. Add this marketplace
/plugin marketplace add definite-app/definite-onprem-skills

# 2. Install the definite-onprem plugin
/plugin install definite-onprem@definite-onprem
```

## Install (whole organization)

Team and Enterprise admins can distribute these plugins to everyone automatically:

1. In Claude Cowork, connect this repository (`definite-app/definite-onprem-skills`)
   as a marketplace via **GitHub sync**.
2. Set the `definite-onprem` plugin's installation preference (auto-install,
   available, or restricted) per group.
3. Updates roll out to your team on their next session.

See [Manage Claude Cowork plugins for your organization](https://support.claude.com/en/articles/13837433-manage-claude-cowork-plugins-for-your-organization).

## Prerequisite: connect your deployment's MCP server

The skill drives your deployment's MCP server (mounted at `/mcp` on the same host
as your Definite app), so each user needs it connected once. Replace
`https://your-deployment/mcp` with your deployment's URL.

### Option A: OAuth connector (Claude Cowork, Claude Desktop, claude.ai)

1. In Claude, add a custom connector with the URL `https://your-deployment/mcp`.
2. Claude registers itself, then opens a Definite sign-in page in the browser.
3. Sign in with your Definite email and password, then approve the request.
4. Claude is returned an access token automatically and the connector is live.

The token is a normal Definite session (14-day lifetime). When it expires,
reconnect the connector to sign in again.

### Option B: static bearer token (Claude Code, scripts, MCP Inspector)

Mint a session token from your Definite login and pass it directly:

```bash
claude mcp add definite --transport http \
  https://your-deployment/mcp \
  --header "Authorization: Bearer <session-token>"
```

For Claude Desktop's config file, the `mcp-remote` bridge does the same:

```json
{
  "mcpServers": {
    "definite": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote", "https://your-deployment/mcp",
        "--header", "Authorization:Bearer <session-token>"
      ]
    }
  }
}
```

## License

MIT
