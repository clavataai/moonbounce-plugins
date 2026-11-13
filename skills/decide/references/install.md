# Install and share

How to connect each agent to Moonbounce and install this skill. This shared reference is included in each exported entry skill.

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

The skills load as `build`, `decide` and the compatibility `decisions` entry, and the MCP server shows as `plugin:moonbounce:tools` at `https://api.moonbounce.ai/mcp`, needing sign-in. An older install pointed at `https://app.moonbounce.ai/mcp`, which still works; moving to the api URL means signing in once more. The server was named `plugin:moonbounce:moonbounce` before; after an update it needs **Authenticate** once under its new name, and permission rules or hooks naming `mcp__plugin_moonbounce_moonbounce__` tools must use `mcp__plugin_moonbounce_tools__`. In Codex the server is still `moonbounce`.

Claude Code shows a startup notice when a server needs sign-in; the person signs in, the agent can't. In a session, run `/mcp` → **plugin:moonbounce:tools** → **Authenticate** ([MCP](https://code.claude.com/docs/en/mcp)). For a server added by hand, `claude mcp login <name>` in a shell does the same. Invoke `/moonbounce:build` ([skills](https://code.claude.com/docs/en/skills)); the plugin installs the skill too. No key is bundled.

## Claude Code sign-in without the plugin

Moonbounce sign-in is live. Add the server ([MCP](https://code.claude.com/docs/en/mcp)):

```sh
claude mcp add --transport http moonbounce https://api.moonbounce.ai/mcp
```

Then run `/mcp` → **moonbounce** → **Authenticate**.

Copy the complete exported `build/` and `decide/` folders to `.claude/skills/` (project) or `~/.claude/skills/` (personal) and invoke `/build` or `/decide` ([skill locations](https://code.claude.com/docs/en/skills#where-skills-live)).

## Cursor

Add the server to `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` ([Cursor MCP](https://cursor.com/docs/mcp)):

```json
{"mcpServers":{"moonbounce":{"url":"https://api.moonbounce.ai/mcp"}}}
```

Sign in from Cursor's MCP settings (Customize → MCP) when the server needs login, or run `agent mcp login moonbounce` ([Cursor MCP](https://cursor.com/docs/mcp), [CLI MCP](https://cursor.com/docs/cli/mcp)).

Copy `build/` and `decide/` to `.agents/skills/` or `.cursor/skills/` (project), or `~/.agents/skills/` or `~/.cursor/skills/` (personal). Invoke `/build` or `/decide` or let Cursor select it automatically ([Cursor skills](https://cursor.com/docs/skills)). Alternatively use the installer below.

## Codex app and CLI

Install the plugin using a CLI with `codex plugin` support ([Codex plugins](https://developers.openai.com/plugins/build/plugins)):

```sh
codex plugin marketplace add https://github.com/clavataai/moonbounce-plugins.git
codex plugin add moonbounce@moonbounce
codex mcp login moonbounce
```

This bundles the MCP connection with `build`, `decide` and the compatibility `decisions` skill. Invoke `$moonbounce:build` or select it through the skill picker. Describe whether to build, improve, assess or resume; build intake handles each. To use an existing decision, invoke `$moonbounce:decide`; it needs no build intake or local files. A decision ID is optional when the agent can locate it with `list_decisions`. Restart a running Codex app if the new plugin or connection is absent. Do not also install the standalone skill in the same host.

### Without the plugin

Add the HTTP server, then sign in ([Codex MCP](https://developers.openai.com/codex/mcp)):

```sh
codex mcp add moonbounce --url https://api.moonbounce.ai/mcp
codex mcp login moonbounce
```

Copy `build/` and `decide/` to `.agents/skills/` (project) or `~/.agents/skills/` (personal). Invoke `$build` or `$decide` or select it through `/skills` ([Codex skills](https://developers.openai.com/codex/skills)). Alternatively use the installer below.

## Any agent: install or copy the skill

Plugin users should not also install a standalone copy in the same host; that duplicates the entry point.

With Node.js 22.20 or later, use the [skills installer](https://github.com/vercel-labs/skills), which supports many agents:

```sh
npx skills add clavataai/moonbounce-plugins --skill build decide
```

Add `-a cursor -a codex` to select agents, `-g` for global installation, or `-y` for non-interactive installation ([installer options](https://github.com/vercel-labs/skills)).

Or share the complete exported skill folders by copy or zip and place them in the agent's skills folder (locations above). Connect MCP separately; Claude Code plugin users already have both.

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

If the plugin is installed, disable `plugin:moonbounce:tools` in `/mcp` so the key-based server is used.

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
