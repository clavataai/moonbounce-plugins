# Install and share

Evidence convention: **observed** means checked locally or against the server, with the scope stated. **from docs** means read in the linked documentation (2026-10-28). **not observed** marks setup guidance not run end to end. The public `skills/decisions/` folder is generated from this skill in Moonbounce's source repository.

## Claude Code plugin

**from docs** — Add the marketplace, then install by plugin and marketplace name ([marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)):

```sh
claude plugin marketplace add clavataai/moonbounce-plugins
claude plugin install moonbounce@moonbounce
```

**observed** — Claude Code 2.1.312 validated and installed the plugin from a local marketplace path on 2026-10-28; `claude mcp list` showed `plugin:moonbounce:moonbounce` at `https://app.moonbounce.ai/mcp` needing authentication. Public-repository installation was **not observed**.

**from docs** — In a session, run `/mcp` → **plugin:moonbounce:moonbounce** → **Authenticate** ([MCP](https://code.claude.com/docs/en/mcp)). Invoke `/moonbounce:decisions` ([skills](https://code.claude.com/docs/en/skills)); the plugin installs the skill too. No key is bundled.

## Claude Code sign-in without the plugin

**from docs** — Moonbounce sign-in is live. Add the server ([MCP](https://code.claude.com/docs/en/mcp)):

```sh
claude mcp add --transport http moonbounce https://app.moonbounce.ai/mcp
```

Then run `/mcp` → **moonbounce** → **Authenticate**.

**from docs** — Copy the complete `decisions/` folder to `.claude/skills/` (project) or `~/.claude/skills/` (personal) and invoke `/decisions`. Copying the source folder instead keeps the name `/moonbounce-decisions` ([skill locations](https://code.claude.com/docs/en/skills#where-skills-live)).

## Cursor

**from docs** — Add the server to `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` ([Cursor MCP](https://cursor.com/docs/mcp)):

```json
{"mcpServers":{"moonbounce":{"url":"https://app.moonbounce.ai/mcp"}}}
```

**from docs** — Sign in from Cursor's MCP settings (Customize → MCP) when the server needs login, or run `agent mcp login moonbounce` ([Cursor MCP](https://cursor.com/docs/mcp), [CLI MCP](https://cursor.com/docs/cli/mcp)). Sign-in from the Cursor agent CLI is checked up to the sign-in page and token exchange (2026-10-28); **not observed** — the Cursor editor's sign-in and a full Cursor session.

**from docs** — Copy `decisions/` to `.agents/skills/` or `.cursor/skills/` (project), or `~/.agents/skills/` or `~/.cursor/skills/` (personal). Invoke `/decisions` or let Cursor select it automatically ([Cursor skills](https://cursor.com/docs/skills)). Alternatively use the installer below.

## Codex CLI

**from docs** — Add the HTTP server, then sign in ([Codex MCP](https://developers.openai.com/codex/mcp)):

```sh
codex mcp add moonbounce --url https://app.moonbounce.ai/mcp
codex mcp login moonbounce
```

**from docs** — Copy `decisions/` to `.agents/skills/` (project) or `~/.agents/skills/` (personal). Invoke `$decisions` or select it through `/skills` ([Codex skills](https://developers.openai.com/codex/skills)). Alternatively use the installer below. **not observed** — The Codex route has not yet been tested with Moonbounce.

## Any agent: install or copy the skill

Claude Code plugin users should not also install the skill with the CLI: that adds `/decisions` beside `/moonbounce:decisions`.

**from docs** — With Node.js 22.20 or later, use the [skills installer](https://github.com/vercel-labs/skills), which supports many agents:

```sh
npx skills add clavataai/moonbounce-plugins --skill decisions
```

**from docs** — Add `-a cursor -a codex` to select agents, `-g` for global installation, or `-y` for non-interactive installation ([installer options](https://github.com/vercel-labs/skills)). **not observed** — This route has not yet been tested with Moonbounce.

**not observed** — Or share the complete `skills/decisions/` folder by copy or zip and place it in the agent's skills folder (locations above). Connect MCP separately; Claude Code plugin users already have both.

## Other MCP clients

**not observed** — Connect with Streamable HTTP at `https://app.moonbounce.ai/mcp`. Moonbounce supports OAuth sign-in through MCP authorization with dynamic client registration and client ID metadata documents, or `Authorization: Bearer <key>`. Other clients have not been tested end to end.

**not observed** — Moonbounce sign-in opens AuthKit and a team picker when needed. Admins and members get `decisions:develop` and `decisions:evaluate` in Production, including real and dry-run evaluations and reads, attributed to the person. Test (`env:test`), unattended agents, backends and REST need an API key.

## API keys, per agent

**not observed** — No key is bundled. Give the agent `decisions:develop` and/or `decisions:evaluate` as needed, with `env:test` for Test. Never print or commit the key. When a key-based and a signed-in server are both connected, disable the signed-in one so the agent uses the key.

### Claude Code

**from docs** — Add this to `.mcp.json` and export `MOONBOUNCE_API_KEY` before starting `claude` ([MCP environment variables](https://code.claude.com/docs/en/mcp)):

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

**not observed** — If the plugin is installed, disable `plugin:moonbounce:moonbounce` in `/mcp` so the key-based server is used.

### Cursor

**from docs** — In `.cursor/mcp.json` or `~/.cursor/mcp.json`, add the header below to the `moonbounce` server. The variable must be in Cursor's environment ([Cursor MCP](https://cursor.com/docs/mcp)):

```json
"headers": { "Authorization": "Bearer ${env:MOONBOUNCE_API_KEY}" }
```

### Codex CLI

**from docs** — Export `MOONBOUNCE_API_KEY` before starting `codex` and configure the bearer token environment variable ([Codex MCP](https://developers.openai.com/codex/mcp)):

```sh
codex mcp add moonbounce --url https://app.moonbounce.ai/mcp --bearer-token-env-var MOONBOUNCE_API_KEY
```

**from docs** — Or set it in `~/.codex/config.toml` ([Codex MCP](https://developers.openai.com/codex/mcp)):

```toml
[mcp_servers.moonbounce]
url = "https://app.moonbounce.ai/mcp"
bearer_token_env_var = "MOONBOUNCE_API_KEY"
```

### Other clients and REST

**not observed** — Set `Authorization: Bearer <key>` through the client's secret or environment configuration. REST uses the same header at `https://app.moonbounce.ai/v1`.

## Claude Code DCR fallback

**not observed** — If Authenticate fails with "Dynamic Client Registration rejected", for the plugin route, disable the plugin's server in `/mcp`; for the plain route, run `claude mcp remove moonbounce`. Then run:

```sh
claude mcp add-json moonbounce '{"type":"http","url":"https://app.moonbounce.ai/mcp","oauth":{"authServerMetadataUrl":"https://essential-sparrow-47.authkit.app/.well-known/oauth-authorization-server"}}'
```

**not observed** — Authenticate the replacement server through `/mcp`.

## Updates and completion

**from docs** — Run `claude plugin update moonbounce@moonbounce` or `/plugin marketplace update moonbounce`, or enable auto-update for **moonbounce** in `/plugin` → **Marketplaces** ([updates](https://code.claude.com/docs/en/plugins/host-marketplace#keep-users-up-to-date)). Moonbounce's plugin has no pinned version.

**from docs** — For Skills CLI installs, run `npx skills update` ([installer updates](https://github.com/vercel-labs/skills)).

**not observed** — For a copied skill, replace the complete folder from the public repository when updating.

**Done when:** The skill is available, the agent has an MCP sign-in or API-key connection, and discovery identifies the connected team and environment.
