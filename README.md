# Extend Agent Plugin

[![smithery badge](https://smithery.ai/badge/extend/extend)](https://smithery.ai/servers/extend/extend)

Document processing for agents, packaged as a plugin. [Extend](https://extend.ai)
turns unstructured documents (PDFs, images, Office files, spreadsheets) into
structured data: parsing, field extraction, classification, splitting,
PDF form filling, and multi-step document workflows.

This plugin bundles:

- **The hosted Extend MCP server** (`https://mcp.extend.ai/mcp`): 80+ tools
  covering extraction, parsing, classification, splitting, PDF editing,
  workflows, files, webhooks, and evaluations. Sign-in is OAuth: your client
  opens a browser window where you choose the workspace and environment the
  connection may use. The server teaches its own cross-tool usage patterns
  through MCP server instructions and per-response guidance, so no companion
  skill is needed for it.
- **`extend-api` skill**: a compact brief for agents writing integration
  code against the [Extend API and SDKs](https://docs.extend.ai): endpoints,
  schema rules, sync-vs-async patterns, webhooks, and error handling.
- **`extend-cli` skill**: how to drive the [`extend` CLI](https://docs.extend.ai/cli)
  for shell, script, and CI work against local files.

The skills also install standalone, without the plugin:

```bash
npx skills add extend-hq/extend-agent-plugin
```

## Install

- **Cursor**: install "Extend" from the [Cursor Marketplace](https://cursor.com/marketplace),
  or load this repository from `~/.cursor/plugins/local/` for development.
- **Claude Code**: `claude plugin install extend` once listed in the plugin
  directory, or add this repository as a [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces).
- **Any MCP client**: you don't need the plugin to use Extend; connect
  directly to `https://mcp.extend.ai/mcp` (Streamable HTTP + OAuth). See the
  [connection guide](https://docs.extend.ai/mcp) for per-client instructions.

## Repository layout

Two manifest formats coexist so one repository serves both ecosystems:

| Path | Purpose |
| --- | --- |
| `plugin.json`, `mcp.json` | [Agent Plugins](https://agent-plugins.org) standard (Cursor and other conformant clients) |
| `.claude-plugin/plugin.json`, `.mcp.json` | [Claude plugin](https://code.claude.com/docs/en/plugins-reference) manifest |
| `skills/` | [Agent Skills](https://agentskills.io) shared by both formats |

## Privacy and terms

The MCP server calls the Extend API on your behalf under the workspace and
environment you approve at sign-in. See Extend's
[privacy policy](https://www.extend.ai/privacy),
[terms](https://www.extend.ai/terms-conditions), and
[security overview](https://docs.extend.ai/security_and_privacy). Support:
support@extend.ai.

## License

MIT; see [LICENSE](LICENSE). The license covers this plugin packaging;
the Extend platform and hosted MCP server are Extend's commercial services.
