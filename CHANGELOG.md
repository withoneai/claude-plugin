# Changelog

All notable changes to the One plugin are documented here. This project follows
[Semantic Versioning](https://semver.org).

## [1.1.0] - 2026-09-30

### Changed

- The server now finds actions with one tool: `find_one_actions` replaces
  `search_one_platform_actions` and `get_one_action_knowledge`. It finds the action for every
  operation a task needs, across platforms, with its documentation, in one call; `load` fetches
  more of a document or an alternative's.
- `integrations` skill: the loop is now list, find, execute, with how to phrase `intent` (the
  operation alone) and `task` (the job in general terms), how to read a find answer (the pick,
  actions also chosen, a substitute to use instead, alternatives), and `load` for more.
- `integration-code` skill: looks every API a feature touches up in one find call, and loads the
  whole document before generating types from a digest.

## [1.0.1] - 2026-08-20

### Changed

- Updated the app count from 500+ to 700+ across the manifest, marketplace catalog, README,
  and both skills.

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
