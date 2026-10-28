# Moonbounce for coding agents

Moonbounce decisions are JSON specs your code owns, evaluated by an engine and improved through feedback.

What's here:

- Hosted MCP server: `https://app.moonbounce.ai/mcp`.
- `skills/decisions/`: a skill for building and improving decisions with any agent that supports skills.
- A Claude Code plugin that installs both the MCP connection and skill.

The root `.mcp.json` and `.claude-plugin/` are the Claude Code plugin's config.

## Connect the MCP server

### Claude Code

```sh
claude plugin marketplace add clavataai/moonbounce-plugins
claude plugin install moonbounce@moonbounce
```

In Claude Code, run `/mcp` → **plugin:moonbounce:moonbounce** → **Authenticate**. Invoke the skill with `/moonbounce:decisions`; the plugin already installs it.

### Cursor

Add to your project's `.cursor/mcp.json` or `~/.cursor/mcp.json`:

```json
{"mcpServers":{"moonbounce":{"url":"https://app.moonbounce.ai/mcp"}}}
```

Sign in from Cursor's MCP settings (Customize → MCP) when the server needs login, or run:

```sh
agent mcp login moonbounce
```

### Codex CLI

```sh
codex mcp add moonbounce --url https://app.moonbounce.ai/mcp
codex mcp login moonbounce
```

### Other MCP clients

Connect to `https://app.moonbounce.ai/mcp` using Streamable HTTP. Sign in through MCP authorization; the server supports dynamic client registration and client ID metadata documents. You can also send `Authorization: Bearer <key>`.

Sign-in gives team admins and members `decisions:develop` and `decisions:evaluate` in Production, including real and dry-run evaluations attributed to them. Test (`env:test`) and unattended agents need an API key.

### Verification

Claude Code plugin installation was observed from a local marketplace path with 2.1.312; installation from GitHub has not yet been tested. Cursor sign-in is new: the Cursor agent CLI is checked up to the sign-in page and token exchange, and the Cursor editor is not tested yet. Codex and the `npx skills` route below have not yet been tested.

## Add the skill

Claude Code plugin users should not also install the skill with the CLI: that adds `/decisions` beside `/moonbounce:decisions`. For other agents, with Node.js 22.20 or later:

```sh
npx skills add clavataai/moonbounce-plugins --skill decisions
```

Optionally add `-a cursor -a codex`, `-g` for global installation, or `-y` for non-interactive installation. This installs the skill; connect MCP separately above.

Or copy `skills/decisions/` into your agent's skills folder:

| Agent | Project folder | Personal folder | Invoke |
| --- | --- | --- | --- |
| Cursor | `.agents/skills/` or `.cursor/skills/` | `~/.agents/skills/` or `~/.cursor/skills/` | `/decisions` or automatically |
| Codex | `.agents/skills/` | `~/.agents/skills/` | `$decisions` or `/skills` |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` | `/decisions` |

## API keys

Use a key with `decisions:develop` and/or `decisions:evaluate` as needed, and `env:test` for Test. Keep `MOONBOUNCE_API_KEY` in the agent's environment; never print or commit it. When both a signed-in server and a key-based server are connected, disable the signed-in one so the agent uses the key.

### Claude Code

Add to your project's `.mcp.json` and export `MOONBOUNCE_API_KEY` before starting `claude`:

```json
{
  "mcpServers": {
    "moonbounce": {
      "type": "http",
      "url": "https://app.moonbounce.ai/mcp",
      "headers": { "Authorization": "Bearer ${MOONBOUNCE_API_KEY}" }
    }
  }
}
```

If the plugin is installed, disable **plugin:moonbounce:moonbounce** in `/mcp`.

### Cursor

In `.cursor/mcp.json` or `~/.cursor/mcp.json`, use this configuration. The variable must be available in Cursor's environment:

```json
{
  "mcpServers": {
    "moonbounce": {
      "url": "https://app.moonbounce.ai/mcp",
      "headers": { "Authorization": "Bearer ${env:MOONBOUNCE_API_KEY}" }
    }
  }
}
```

### Codex CLI

Export `MOONBOUNCE_API_KEY` before starting `codex`:

```sh
codex mcp add moonbounce --url https://app.moonbounce.ai/mcp --bearer-token-env-var MOONBOUNCE_API_KEY
```

### Other MCP clients

Set the server's `Authorization` header to `Bearer <key>` using the client's secret or environment configuration.

## Updates

```sh
claude plugin update moonbounce@moonbounce
```

For Skills CLI installs, run `npx skills update`.

This repository is generated from Moonbounce's source repository. Changes made here are overwritten on the next sync; contact Moonbounce with issues.
