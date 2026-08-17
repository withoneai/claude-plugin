---
name: integrations
description: >-
  Read and write real data in third-party apps — Gmail, Slack, Stripe, Shopify, HubSpot,
  QuickBooks, Linear, Notion, Salesforce, Google Calendar and 500+ more — through the One MCP
  server. Use whenever the user wants to send an email, post a message, look up a customer, pull
  invoices, create or update a record (order, contact, task, issue, deal), or take ANY action in an
  external or connected SaaS app, even if they never say "One". Workflow: list_one_integrations →
  search_one_platform_actions → get_one_action_knowledge → execute_one_action. Never execute
  without reading the knowledge first.
license: MIT
allowed-tools:
  - mcp__plugin_one_one__list_one_integrations
  - mcp__plugin_one_one__search_one_platform_actions
  - mcp__plugin_one_one__get_one_action_knowledge
---

# Working with third-party apps through One

One exposes every app the user has connected through **four tools**. No matter how many apps
or actions they connect, it stays four tools — so *search* is how you find things, not a giant
tool list.

| Tool | What it does |
| --- | --- |
| `list_one_integrations` | Lists the user's active connections, each with its `key` and the `access` it allows |
| `search_one_platform_actions` | Searches the action catalog of one platform |
| `get_one_action_knowledge` | Returns the real documentation for one action: parameters, types, request shape, gotchas |
| `execute_one_action` | Runs the action against the live account |

**Golden rule: never guess an action's parameters. Always read its knowledge first.**

## The loop

Follow it in order. Skipping a step is where these calls fail.

### 1. `list_one_integrations` — always start here

No input. Returns `connectedCount` and a `connections` array. Each connection has:

- `platform` — kebab-case slug (`gmail`, `slack`, `google-calendar`, `quickbooks`)
- `key` — the **connection key** you pass to `execute_one_action` as `connection_key`
- `access` — what you may run on it (see "Read the access field" below)
- `tags` — optional labels the user gave the connection (`["work"]`, `["acme"]`)

Use it to confirm the platform the user wants is actually connected and to grab its `key`.
If the same platform appears more than once, use `tags` (or ask) to pick the right account.
Only use keys returned here — never invent one.

If the platform is missing, the user hasn't connected it. Say which platform and point them at
https://app.withone.ai to add it. Do **not** fall back to a raw HTTP request, a scraped page, or
a different platform that happens to be connected.

### 2. `search_one_platform_actions` — find the action

Input: `platform` (the slug from step 1), `query` (plain language describing the *outcome* —
"send an email", "list paid invoices", "create a customer"), and optional `agent_type`
(`"execute"` when the user wants to *perform* something, `"knowledge"` when they want to
read/learn/generate code; omit to search all). Returns up to 5 candidates, each with an
`actionId`, `title`, HTTP `method`, and `path`. Pick the best match. If nothing fits, rephrase
by outcome or re-check the platform slug.

Skip this step for **action-scoped** connections — their `access.actions` list already names
exactly what may run.

### 3. `get_one_action_knowledge` — MANDATORY before executing

Input: `action_id` and `platform`. Returns the full action doc — required and optional
parameters, exact names and casing, enums, where each value belongs (path / query / body /
header), the response shape, and platform-specific gotchas. **Always call this before
`execute_one_action`**, for the *specific* `action_id` you're about to run.

> The knowledge text ends with a "How to execute this action" block written in camelCase
> (`actionId`, `connectionKey`, `pathVariables`, `queryParams`, `platform`). That block describes
> the *concepts*; the actual tool parameters are the snake_case names in the next step. Follow the
> tool schema.

### 4. `execute_one_action` — perform the operation

Parameters (snake_case — these are the real names on the tool):

| Parameter | Required | What to pass |
| --- | --- | --- |
| `action_id` | yes | The `actionId` from step 2 (or from `access.actions`) |
| `connection_key` | yes | The connection `key` from step 1 |
| `data` | as needed | Request body (JSON by default) |
| `path_variables` | as needed | Values for `{var}` / `{{var}}` placeholders in the action path |
| `query_params` | as needed | Query-string parameters |
| `headers` | rarely | Extra headers to forward to the platform |
| `is_form_url_encoded` | rarely | `true` to send `data` as `application/x-www-form-urlencoded` |
| `is_form_data` | no | Not supported yet — leave unset |

There is no `platform` parameter on execute; the connection key already identifies it. Put
values where the knowledge says they go — path variables in `path_variables`, query params in
`query_params`, body fields in `data`. Don't hand-build URLs or stuff path values into the body.

This makes a **live call** to the real platform, so Claude Code asks the user to approve it.
Summarize what you're about to do first.

## Read the access field before you plan

Each connection's `access` tells you exactly what you may run:

- `{"policy": "full"}` — every action on that connection.
- `{"policy": "methods", "methods": ["GET"]}` — only those HTTP methods (`["GET"]` = read-only).
  Plan a read-only answer and say so, rather than attempting a write that will be refused.
- `{"policy": "actions", "actions": [{"actionId", "title", "method"}, …]}` — exactly those actions.
  Use one of them directly; no need to search.

If `execute_one_action` isn't in your tool list at all, the user chose **knowledge-only mode**
on the One consent screen. You can list, search, and read documentation, but not perform live
operations — switch to writing code (see the `integration-code` skill) or tell the user they
need to re-authorize with execution enabled (`/mcp` → One → Clear authentication → sign in again).

## Never guess parameters

The knowledge returns the actual schema: required fields, exact names, types, enum values, and
where each value belongs. Guessing a field name that looks obvious produces a 400 from the
platform — or worse, a 200 that wrote the wrong thing.

If a required value is missing and you can't derive it from the conversation or a previous
read, **ask the user**. Do not invent an id, an email address, an amount, or a date.

## Before a write, say what you are about to do

Sends, payments, deletions, and status changes land on real accounts and real people and can't
be recalled. Before the first write in a task, state the platform, the action, and the specific
target in one line ("Sending to jane@acme.com via Gmail (work)") and let the user stop you.
Reads need no confirmation.

Never write to a platform the user didn't ask you to touch. Pulling a contact from HubSpot is
not permission to update it.

## Branch: doing vs building

- **Doing** ("send the email", "create the order", "post to Slack") → run all four steps.
- **Building** ("write a script that syncs Shopify orders", "add a Stripe webhook handler") →
  run steps 1–3, then **stop and write code** from the returned knowledge. Don't call
  `execute_one_action`; the user wants source, not a live call. The `integration-code` skill
  covers this in depth.

## Pagination

List actions are paginated. The knowledge names the parameters (`limit`, `cursor`, `page`,
`starting_after`, `pageToken` — it varies by platform). Fetch a bounded page, summarize it, and
tell the user it was a page rather than everything. Don't page a whole account into context.

## When a call fails

The error comes from the platform, not from One. Read it.

- **400 / 422** — your parameters don't match the schema. Re-read the knowledge, fix the field,
  retry once.
- **401 / 403** — the connection lacks permission or needs re-authorizing on One's side. Tell the
  user which platform and stop; retrying won't help.
- **404** — the id doesn't exist on that account. Verify with a read before assuming the action
  is wrong.
- **429** — rate limited. Back off; if you were looping, batch instead.
- **"Missing required OAuth scope"** — the One grant itself lacks the scope. Re-authorize via
  `/mcp`.

Report the failure with the platform's own message. Never retry a write more than once — the
first attempt may have succeeded.

## Multiple platforms in one task

Chain reads before writes. Pull from every source first, reconcile, then write once per target.
A per-record read-then-write loop across two platforms is slow and leaves half-finished state
when it breaks.

## Report results, not just "done"

Return the created record's id or link, the count of things read, or the platform's response —
whatever lets the user verify the outcome.

## Setup and auth

The One server is remote (`https://mcp.withone.ai/mcp`) and uses OAuth. On first use Claude
Code prompts the user to sign in via `/mcp` → **One** → Authenticate; the browser opens One's
consent screen, where they scope this client — which connections, read or read-write, or
knowledge-only. There's no API key. If tools are missing or every call returns 401, that's the
fix: `/mcp` and (re)authenticate.

Full docs: https://www.withone.ai/docs/mcp
