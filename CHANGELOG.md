# Changelog

All notable changes to the One plugin are documented here. This project follows
[Semantic Versioning](https://semver.org).

## [1.0.0] - 2026-08-17

### Added

- Initial release.
- Connects Claude Code and Claude Cowork to One's remote MCP server (`https://mcp.withone.ai/mcp`) over
  Streamable HTTP with OAuth 2.1 sign-in (dynamic client registration + PKCE). Nothing to
  install locally and no API keys to manage.
- Four tools: `list_one_integrations`, `search_one_platform_actions`,
  `get_one_action_knowledge`, and `execute_one_action`.
- `integrations` skill: the discover, search, read, execute funnel, access-policy handling,
  confirm-before-write discipline, and error handling.
- `integration-code` skill: writing integration code from real API schemas (through One's
  Passthrough API or direct to the vendor) without executing live calls.
- Self-hosted marketplace catalog for `/plugin marketplace add withoneai/claude-plugin`.
