# Install and share

How to connect each agent to Moonbounce and install this skill. The public `skills/decisions/` folder is generated from this skill in Moonbounce's source repository.

## Claude Code plugin

One command adds the marketplace and installs (Claude Code 2.1.292 or later) ([install](https://code.claude.com/docs/en/plugins/install.md#add-a-marketplace-and-install-in-one-command)):

```sh
claude plugin install moonbounce --marketplace clavataai/moonbounce-plugins
```

On an older Claude Code, add the marketplace, then install by plugin and marketplace name:

```sh
claude plugin marketplace add clavataai/moonbounce-plugins
claude plugin install moonbounce@moonbounce
```

The skill loads as `decisions`, and the MCP server shows as `plugin:moonbounce:moonbounce` at `https://api.moonbounce.ai/mcp`, needing sign-in. An older install pointed at `https://app.moonbounce.ai/mcp`, which still works; moving to the api URL means signing in once more.

Claude Code shows a startup notice when a server needs sign-in; the person signs in, the agent can't. In a session, run `/mcp` → **plugin:moonbounce:moonbounce** → **Authenticate** ([MCP](https://code.claude.com/docs/en/mcp)). For a server added by hand, `claude mcp login <name>` in a shell does the same. Invoke `/moonbounce:decisions` ([skills](https://code.claude.com/docs/en/skills)); the plugin installs the skill too. No key is bundled.

## Claude Code sign-in without the plugin

Moonbounce sign-in is live. Add the server ([MCP](https://code.claude.com/docs/en/mcp)):

```sh
claude mcp add --transport http moonbounce https://api.moonbounce.ai/mcp
```

Then run `/mcp` → **moonbounce** → **Authenticate**.

Copy the complete `decisions/` folder to `.claude/skills/` (project) or `~/.claude/skills/` (personal) and invoke `/decisions`. Copying the source folder instead keeps the name `/moonbounce-decisions` ([skill locations](https://code.claude.com/docs/en/skills#where-skills-live)).

## Cursor

Add the server to `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` ([Cursor MCP](https://cursor.com/docs/mcp)):

```json
{"mcpServers":{"moonbounce":{"url":"https://api.moonbounce.ai/mcp"}}}
```

Sign in from Cursor's MCP settings (Customize → MCP) when the server needs login, or run `agent mcp login moonbounce` ([Cursor MCP](https://cursor.com/docs/mcp), [CLI MCP](https://cursor.com/docs/cli/mcp)).

Copy `decisions/` to `.agents/skills/` or `.cursor/skills/` (project), or `~/.agents/skills/` or `~/.cursor/skills/` (personal). Invoke `/decisions` or let Cursor select it automatically ([Cursor skills](https://cursor.com/docs/skills)). Alternatively use the installer below.

## Codex app and CLI

Install the plugin using a CLI with `codex plugin` support ([Codex plugins](https://developers.openai.com/plugins/build/plugins)):

```sh
codex plugin marketplace add https://github.com/clavataai/moonbounce-plugins.git
codex plugin add moonbounce@moonbounce
codex mcp login moonbounce
```

This bundles the MCP connection and the shared `decisions` skill. Invoke `$moonbounce:decisions` or select it through the skill picker. Describe whether to build, improve, assess or resume; the same conversational intake handles each. A decision ID is optional when the agent can locate it with `list_decisions`. Restart a running Codex app if the new plugin or connection is absent. Do not also install the standalone skill in the same host.

### Without the plugin

Add the HTTP server, then sign in ([Codex MCP](https://developers.openai.com/codex/mcp)):

```sh
codex mcp add moonbounce --url https://api.moonbounce.ai/mcp
codex mcp login moonbounce
```

Copy `decisions/` to `.agents/skills/` (project) or `~/.agents/skills/` (personal). Invoke `$decisions` or select it through `/skills` ([Codex skills](https://developers.openai.com/codex/skills)). Alternatively use the installer below.

## Any agent: install or copy the skill

Plugin users should not also install a standalone copy in the same host; that duplicates the entry point.

With Node.js 22.20 or later, use the [skills installer](https://github.com/vercel-labs/skills), which supports many agents:

```sh
npx skills add clavataai/moonbounce-plugins --skill decisions
```

Add `-a cursor -a codex` to select agents, `-g` for global installation, or `-y` for non-interactive installation ([installer options](https://github.com/vercel-labs/skills)).

Or share the complete `skills/decisions/` folder by copy or zip and place it in the agent's skills folder (locations above). Connect MCP separately; Claude Code plugin users already have both.

## Other MCP clients

Connect with Streamable HTTP at `https://api.moonbounce.ai/mcp`. Moonbounce supports OAuth sign-in through MCP authorization with dynamic client registration and client ID metadata documents, or `Authorization: Bearer <key>`. Other clients have not been tested end to end.

Moonbounce sign-in opens AuthKit and a team picker when needed. Admins and members get `decisions:develop` and `decisions:evaluate` in Production, including real and dry-run evaluations and reads, attributed to the person. Test (`env:test`), unattended agents, backends and REST need an API key.

## API keys, per agent

No key is bundled. Give the agent `decisions:develop` and/or `decisions:evaluate` as needed, with `env:test` for Test. Never print or commit the key. When a key-based and a signed-in server are both connected, disable the signed-in one so the agent uses the key.

### Claude Code

Add this to `.mcp.json` and export `MOONBOUNCE_API_KEY` before starting `claude` ([MCP environment variables](https://code.claude.com/docs/en/mcp)):

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

If the plugin is installed, disable `plugin:moonbounce:moonbounce` in `/mcp` so the key-based server is used.

### Cursor

In `.cursor/mcp.json` or `~/.cursor/mcp.json`, add the header below to the `moonbounce` server. The variable must be in Cursor's environment ([Cursor MCP](https://cursor.com/docs/mcp)):

```json
"headers": { "Authorization": "Bearer ${env:MOONBOUNCE_API_KEY}" }
```

### Codex CLI

Export `MOONBOUNCE_API_KEY` before starting `codex` and configure the bearer token environment variable ([Codex MCP](https://developers.openai.com/codex/mcp)):

```sh
codex mcp add moonbounce --url https://api.moonbounce.ai/mcp --bearer-token-env-var MOONBOUNCE_API_KEY
```

Or set it in `~/.codex/config.toml` ([Codex MCP](https://developers.openai.com/codex/mcp)):

```toml
[mcp_servers.moonbounce]
url = "https://api.moonbounce.ai/mcp"
bearer_token_env_var = "MOONBOUNCE_API_KEY"
```

### Other clients and REST

Set `Authorization: Bearer <key>` through the client's secret or environment configuration. REST uses the same header at `https://api.moonbounce.ai/v1`.

## Claude Code DCR fallback

If Authenticate fails with "Dynamic Client Registration rejected", for the plugin route, disable the plugin's server in `/mcp`; for the plain route, run `claude mcp remove moonbounce`. Then run:

```sh
claude mcp add-json moonbounce '{"type":"http","url":"https://api.moonbounce.ai/mcp","oauth":{"authServerMetadataUrl":"https://auth.moonbounce.ai/.well-known/oauth-authorization-server"}}'
```

Authenticate the replacement server through `/mcp`.

If the server was added this way earlier with an `authkit.app` metadata URL, remove it and add it again with the command above.

## Updates and completion

Run `claude plugin update moonbounce@moonbounce` or `/plugin marketplace update moonbounce`, or enable auto-update for **moonbounce** in `/plugin` → **Marketplaces** ([updates](https://code.claude.com/docs/en/plugins/host-marketplace#keep-users-up-to-date)). Moonbounce's plugin has no pinned version.

For Skills CLI installs, run `npx skills update` ([installer updates](https://github.com/vercel-labs/skills)).

For a copied skill, replace the complete folder from the public repository when updating.

**Done when:** The skill is available, the agent has an MCP sign-in or API-key connection, and discovery identifies the connected team and environment.
