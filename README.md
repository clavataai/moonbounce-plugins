# Moonbounce for coding agents

Moonbounce decisions are JSON structures your code owns, evaluated by an engine and improved through feedback.

What's here:

- Hosted MCP server: `https://api.moonbounce.ai/mcp`.
- `skills/build/`: create and refine decision definitions.
- `skills/decide/`: find and evaluate existing decisions.
- `skills/decisions/`: the compatibility entry point.
- Claude Code and Codex plugin installation, bundling the same MCP connection and skill.

The two hosts share `skills/` and `.mcp.json`. `.claude-plugin/` configures Claude Code; `.codex-plugin/` and `.agents/plugins/marketplace.json` configure Codex.

## Connect the MCP server

### Claude Code

```sh
claude plugin install moonbounce --marketplace moonbounce-ai/moonbounce-plugins
```

That needs Claude Code 2.1.292 or later; on an older version, run `claude plugin marketplace add moonbounce-ai/moonbounce-plugins`, then `claude plugin install moonbounce@moonbounce`.

Then sign in: run `/mcp` → **plugin:moonbounce:moonbounce** → **Authenticate** (Claude Code also shows a notice at startup when it's needed; your agent can't start sign-in for you). Invoke `/moonbounce:build` and describe what you want to build, improve, assess or resume. Include a decision ID if known; the agent can list registered decisions to find it, and retrieve a server-only decision without an existing local file. The agent reuses the conversation and materials, proposes the needed work and follows the requested scope and stopping point. Use `/moonbounce:decide` to apply an existing decision to inputs; `/moonbounce:decisions` remains available for compatibility. Other MCP clients can use the `build_decision` and `decide` prompts through server-owned guidance. The plugin is a thin entry point: the returned plan identifies each stage's guidance, and `get_decision_guidance` resolves its pinned version and related role instructions.

For example: `/moonbounce:build refund-exception Build and register a quick, scored first version, then stop. No tuning or app changes yet.` This asks for a usable measured result, not an untested draft or a fixed Quick preset. The agent reports what was tested, the score, cost and remaining limitations; further work follows the user's request.

### Decide with an existing decision

Invoke `/moonbounce:decide` in Claude Code or `$moonbounce:decide` in Codex and describe the task, optionally naming a decision and version. The agent discovers a suitable decision, fetches its definition, gathers missing inputs and evaluates. It honors an explicit version; otherwise it selects the newest registered version and pins its hash for this task. Newest does not imply approval or measured quality. If nothing fits, it directs you to build. No repository, build plan or generated dataset is needed. Ordinary evaluations are recorded for history and feedback; an explicit dry run spends model calls without an evaluation record.

### Cursor

Add to your project's `.cursor/mcp.json` or `~/.cursor/mcp.json`:

```json
{"mcpServers":{"moonbounce":{"url":"https://api.moonbounce.ai/mcp"}}}
```

Sign in from Cursor's MCP settings (Customize → MCP) when the server needs login, or run:

```sh
agent mcp login moonbounce
```

### Codex app and CLI

Install the plugin with a Codex CLI that supports `codex plugin`:

```sh
codex plugin marketplace add https://github.com/moonbounce-ai/moonbounce-plugins.git
codex plugin add moonbounce@moonbounce
codex mcp login moonbounce
```

The app and CLI use the same local Codex configuration. Restart an already running app after installation if it has not picked up the connection. Invoke `$moonbounce:build` or select it in the skill picker; describe the requested work, for example `Improve refund-exception. Focus on false denials.` Use `$moonbounce:decide` to apply an existing decision to inputs. `$moonbounce:decisions` remains available for compatibility. Claude's slash command syntax is not a Codex entry point.

For MCP only, without installing the plugin:

```sh
codex mcp add moonbounce --url https://api.moonbounce.ai/mcp
codex mcp login moonbounce
```

### Other MCP clients

Connect to `https://api.moonbounce.ai/mcp` using Streamable HTTP. Sign in through MCP authorization; the server supports dynamic client registration and client ID metadata documents. You can also send `Authorization: Bearer <key>`.

Sign-in gives team admins and members `decisions:develop` and `decisions:evaluate` in Production, including real and dry-run evaluations attributed to them. Test (`env:test`) and unattended agents need an API key.

### Verification

Claude Code plugin installation from GitHub was observed with 2.1.312 in a clean profile. Codex 0.167.0 was checked for native plugin installation, discovery of the enabled Moonbounce skills and the bundled MCP declaration using an isolated configuration; production OAuth sign-in was not tested end to end. Cursor's agent CLI is checked up to the sign-in page and token exchange; its editor is not tested yet. The Skills CLI route is checked in an empty project.

## Add the skill

Plugin users should not also install a standalone copy of the same skill in that host. Choose one installation route per host. For other agents, with Node.js 22.20 or later:

```sh
npx skills add moonbounce-ai/moonbounce-plugins --skill build decide
```

Optionally add `-a cursor -a codex`, `-g` for global installation, or `-y` for non-interactive installation. This installs the skill; connect MCP separately above.

Or copy the complete `skills/build/` and `skills/decide/` folders into your agent's skills folder:

| Agent | Project folder | Personal folder | Invoke |
| --- | --- | --- | --- |
| Cursor | `.agents/skills/` or `.cursor/skills/` | `~/.agents/skills/` or `~/.cursor/skills/` | `/build`, `/decide` or automatically |
| Codex | `.agents/skills/` | `~/.agents/skills/` | `$build`, `$decide` or `/skills` |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` | `/build` or `/decide` |

## API keys

Use a key with both `decisions:develop` (hosted guidance) and `decisions:evaluate` (evaluation) for decide, and `env:test` for Test. Keep `MOONBOUNCE_API_KEY` in the agent's environment; never print or commit it. When both a signed-in server and a key-based server are connected, disable the signed-in one so the agent uses the key.

### Claude Code

Add to your project's `.mcp.json` and export `MOONBOUNCE_API_KEY` before starting `claude`:

```json
{
  "mcpServers": {
    "moonbounce": {
      "type": "http",
      "url": "https://api.moonbounce.ai/mcp",
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
      "url": "https://api.moonbounce.ai/mcp",
      "headers": { "Authorization": "Bearer ${env:MOONBOUNCE_API_KEY}" }
    }
  }
}
```

### Codex CLI

Export `MOONBOUNCE_API_KEY` before starting `codex`:

```sh
codex mcp add moonbounce --url https://api.moonbounce.ai/mcp --bearer-token-env-var MOONBOUNCE_API_KEY
```

### Other MCP clients

Set the server's `Authorization` header to `Bearer <key>` using the client's secret or environment configuration.

## Updates

```sh
claude plugin update moonbounce@moonbounce
```

For Skills CLI installs, run `npx skills update`.

The MCP server moved to `https://api.moonbounce.ai/mcp`. `https://app.moonbounce.ai/mcp` keeps working, but the new URL is a separate sign-in, so if you signed in before, run **Authenticate** once more after updating. API keys work on both.

This repository is generated from Moonbounce's source repository. Changes made here are overwritten on the next sync; contact Moonbounce with issues.
