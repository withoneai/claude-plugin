# One for Claude Code

Connect Claude Code to **500+ apps** (Gmail, Slack, Shopify, HubSpot, Stripe, Notion, Linear,
Salesforce, QuickBooks, Google Calendar, and more) through [One](https://www.withone.ai)'s
remote MCP server.

Four tools. Real API documentation for 80,000+ actions. OAuth sign-in in your browser. Nothing to
install and no API keys to manage: One handles every platform's auth server-side.

```text
Discover -> Search -> Read the docs -> Execute
list_one_integrations -> search_one_platform_actions -> get_one_action_knowledge -> execute_one_action
```

## What you can do

- "Send an email to alex@acme.com with the Q3 summary" (Gmail)
- "Post 'deploy is live' to #engineering" (Slack)
- "Create a Stripe customer for this signup" (Stripe)
- "List my open Linear issues" (Linear)
- "Write a Next.js route that creates a HubSpot contact" (Claude writes code from the live API schema)

## Install

In Claude Code:

```text
/plugin marketplace add withoneai/claude-plugin
/plugin install one@one
```

> Once approved in Anthropic's community marketplace, you will also be able to run
> `/plugin marketplace add anthropics/claude-plugins-community` then `/plugin install one@claude-community`.

Then sign in:

```text
/mcp
```

Pick **One**, then **Authenticate**. Your browser opens One's consent screen, where you choose
which connections this Claude Code client may use, whether it can write or only read, and
whether to enable knowledge-only mode. That's it. Tokens are stored securely and refreshed
automatically.

Or from your shell: `claude mcp login plugin:one:one`.

### Prerequisites

- A One account with the apps you want to use connected: [app.withone.ai](https://app.withone.ai).
- Claude Code v2.1.196 or later (for the OAuth scope handling the server relies on).

## Usage

Just ask. The bundled skills teach Claude the discover, search, read, execute funnel, so you
don't have to name tools:

| Skill | When Claude uses it |
| --- | --- |
| `/one:integrations` | Doing things in apps: send, create, update, look up, sync. |
| `/one:integration-code` | Writing code that calls a third-party API, using the real schema instead of a guessed one. |

`execute_one_action` performs a **live** call (sending email, creating records, and so on), so
Claude Code asks you to approve it. The three discovery tools are read-only.

## Tools

| Tool | What it does |
| --- | --- |
| `list_one_integrations` | Your connected accounts, each with its connection `key` and the `access` it allows. |
| `search_one_platform_actions` | Find actions on a platform from a natural-language query. |
| `get_one_action_knowledge` | An action's full documentation: parameters, types, request/response shape, gotchas. |
| `execute_one_action` | Perform the live API call through One. |

Tool names inside Claude Code are prefixed `mcp__plugin_one_one__` (for example
`mcp__plugin_one_one__execute_one_action`). Use those in permission rules or hooks.

## Scoping and safety

Access is decided on One's OAuth consent screen when you authenticate, not in a config file:

| On the consent screen | Effect |
| --- | --- |
| **Which connections** | Only the selected accounts appear in `list_one_integrations`. |
| **Read / full / specific actions** per connection | Reflected in each connection's `access` field; disallowed calls are refused server-side. |
| **Knowledge-only mode** | Removes `execute_one_action` entirely. Claude can read every API's real schema and write code, but can never perform a live operation. |

To change scoping later: `/mcp`, select **One**, choose **Clear authentication**, then authenticate again.

Every third-party call is proxied by One, so no platform tokens ever touch your machine.

## How it works

The plugin's `.mcp.json` points Claude Code at One's Streamable-HTTP MCP endpoint
(`https://mcp.withone.ai/mcp`). Claude Code discovers the OAuth server via RFC 9728 protected-resource
metadata, registers itself dynamically (RFC 7591), and runs a PKCE authorization-code flow in
your browser. Nothing runs locally.

## Troubleshooting

- **Tools don't appear / "needs authentication"**: run `/mcp`, select **One**, and authenticate.
- **`execute_one_action` is missing**: you chose knowledge-only mode on the consent screen.
  Clear authentication in `/mcp` and sign in again with execution enabled.
- **A platform isn't listed**: it's not connected (or wasn't selected on the consent screen).
  Connect it at [app.withone.ai](https://app.withone.ai), or re-authenticate to widen the scope.
- **"Missing required OAuth scope"**: re-authenticate via `/mcp`.
- **You also use the One connector on claude.ai**: Claude Code deduplicates servers by endpoint, so
  a session shows either `plugin:one:one` or `claude.ai One` (same server, same four tools). If the
  connector wins, tool names appear as `mcp__claude_ai_One__*` instead. Everything else is identical.
- **Working on this repo itself**: its `.mcp.json` doubles as a project-scoped server when you open
  Claude Code here. Add `"disabledMcpjsonServers": ["one"]` to `.claude/settings.local.json` to avoid
  a duplicate alongside `--plugin-dir`.

## Links

- One: https://www.withone.ai
- MCP docs: https://www.withone.ai/docs/mcp
- Dashboard: https://app.withone.ai
- This plugin: https://github.com/withoneai/claude-plugin

## License

MIT. See [LICENSE](LICENSE).
