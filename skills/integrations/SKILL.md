---
name: integrations
description: >-
  Read and write real data in third-party apps (Gmail, Slack, Stripe, Shopify, HubSpot,
  QuickBooks, Linear, Notion, Salesforce, Google Calendar and 700+ more) through the One MCP
  server. Use whenever the user wants to send an email, post a message, look up a customer, pull
  invoices, create or update a record (order, contact, task, issue, deal), or take ANY action in an
  external or connected SaaS app, even if they never say "One". Workflow: list_one_integrations,
  then find_one_actions (every operation the task needs, in one call, with each action's
  documentation), then execute_one_action. Never execute without reading the documentation first.
license: MIT
allowed-tools:
  - mcp__plugin_one_one__list_one_integrations
  - mcp__plugin_one_one__find_one_actions
---

# Working with third-party apps through One

One exposes every app the user has connected through **three tools**. No matter how many apps
or actions they connect, it stays three tools, so *finding* is how you get to an action, not a
giant tool list.

| Tool | What it does |
| --- | --- |
| `list_one_integrations` | Lists the user's active connections, each with its `key` and the `access` it allows |
| `find_one_actions` | Finds the action for every operation a task needs, across platforms, with its real documentation: parameters, types, request shape, gotchas |
| `execute_one_action` | Runs the action against the live account |

**Golden rule: never guess an action's parameters. Always read its documentation first.**

## The loop

Follow it in order. Skipping a step is where these calls fail.

### 1. `list_one_integrations`: always start here

No input. Returns `connectedCount` and a `connections` array. Each connection has:

- `platform`: kebab-case slug (`gmail`, `slack`, `google-calendar`, `quickbooks`)
- `key`: the **connection key** you pass to `execute_one_action` as `connection_key`
- `access`: what you may run on it (see "Read the access field" below)
- `tags`: optional labels the user gave the connection (`["work"]`, `["acme"]`)

Use it to confirm the platform the user wants is actually connected and to grab its `key`.
If the same platform appears more than once, use `tags` (or ask) to pick the right account.
Only use keys returned here. Never invent one.

If the platform is missing, the user hasn't connected it. Say which platform and point them at
https://app.withone.ai to add it. Do **not** fall back to a raw HTTP request, a scraped page, or
a different platform that happens to be connected.

### 2. `find_one_actions`: find every action the task needs, with its documentation

Send **one call for the whole task**, one `requests` entry per operation, on any platforms:

```json
{
  "task": "email a weather report to a contact",
  "requests": [
    { "platform": "wttr-in", "intent": "get the weather for a city" },
    { "platform": "gmail", "intent": "send an email" }
  ]
}
```

- `intent` names the **operation alone**, in a few words: "send a message to a channel", not
  "post 'deploy done' in #eng". IDs, names and message text in the intent make the search miss.
- `task` (optional) is the whole job in one line, **in general terms**: what it does, without
  names, addresses, IDs or message text. It helps choose between similar actions.
- `platform` is the slug from step 1. Up to 10 requests per call.

For each intent the answer says why it chose what it did, then gives:

- **The action to use**, headed `Title · METHOD path · actionId: ...`, with its documentation:
  required and optional parameters, exact names and casing, enums, where each value belongs
  (path / query / body / header), the response shape, and gotchas. Large documents come back as
  a digest that names what it left out.
- Sometimes **actions also chosen**: needed beside the pick for the same intent (look up, then
  update). Use them together.
- Sometimes **"use this one instead: one or the other, never both"**: a substitute for when the
  pick doesn't fit. Never run both.
- **Alternatives**, one line each, undocumented.

If it says **no action fits**, rephrase the intent by outcome or check the platform slug. If it
says the chosen action **is not allowed**, the user's access settings refused it and it offered
the next allowed one; check it fits.

**Need more of a document?** Call `find_one_actions` again with `load` instead of `requests`:
`load: [{ "action_id": "...", "section": "Response" }]` for a section the digest left out (the
digest names the exact call), `"full": true` for the whole document, `"toc": true` for its
contents, or just the `action_id` of an alternative for its documentation.

For **action-scoped** connections, `access.actions` already names exactly what may run: read
the one you need with `load: [{ "action_id": "..." }]` instead of sending `requests`.

### 3. `execute_one_action`: perform the operation

Parameters (snake_case; these are the real names on the tool):

| Parameter | Required | What to pass |
| --- | --- | --- |
| `action_id` | yes | The `actionId` from step 2 (or from `access.actions`) |
| `connection_key` | yes | The connection `key` from step 1 |
| `data` | as needed | Request body (JSON by default) |
| `path_variables` | as needed | Values for `{var}` / `{{var}}` placeholders in the action path |
| `query_params` | as needed | Query-string parameters |
| `headers` | rarely | Extra headers to forward to the platform |
| `is_form_url_encoded` | rarely | `true` to send `data` as `application/x-www-form-urlencoded` |
| `is_form_data` | no | Not supported yet. Leave unset |

There is no `platform` parameter on execute; the connection key already identifies it. Put
values where the documentation says they go: path variables in `path_variables`, query params in
`query_params`, body fields in `data`. Don't hand-build URLs or stuff path values into the body.

This makes a **live call** to the real platform, so Claude Code asks the user to approve it.
Summarize what you're about to do first.

## Read the access field before you plan

Each connection's `access` tells you exactly what you may run:

- `{"policy": "full"}`: every action on that connection.
- `{"policy": "methods", "methods": ["GET"]}`: only those HTTP methods (`["GET"]` = read-only).
  Plan a read-only answer and say so, rather than attempting a write that will be refused.
- `{"policy": "actions", "actions": [{"actionId", "title", "method"}, ...]}`: exactly those
  actions. Use one of them directly; `find_one_actions` with `load` gets its documentation.

If `execute_one_action` isn't in your tool list at all, the user chose **knowledge-only mode**
on the One consent screen. You can list and find actions with their documentation, but not perform
live operations: `find_one_actions` then returns each action's whole document with how to call it
from code. Switch to writing code (see the `integration-code` skill) or tell the user they
need to re-authorize with execution enabled (`/mcp`, One, Clear authentication, sign in again).

## Never guess parameters

The documentation returns the actual schema: required fields, exact names, types, enum values, and
where each value belongs. Guessing a field name that looks obvious produces a 400 from the
platform, or worse, a 200 that wrote the wrong thing.

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

- **Doing** ("send the email", "create the order", "post to Slack"): run all three steps.
- **Building** ("write a script that syncs Shopify orders", "add a Stripe webhook handler"):
  run steps 1 and 2, then **stop and write code** from the returned documentation. Don't call
  `execute_one_action`; the user wants source, not a live call. The `integration-code` skill
  covers this in depth.

## Pagination

List actions are paginated. The documentation names the parameters (`limit`, `cursor`, `page`,
`starting_after`, `pageToken`; it varies by platform). Fetch a bounded page, summarize it, and
tell the user it was a page rather than everything. Don't page a whole account into context.

## When a call fails

The error comes from the platform, not from One. Read it.

- **400 / 422**: your parameters don't match the schema. Re-read the documentation (`load` the
  section if the digest left it out), fix the field, retry once.
- **401 / 403**: the connection lacks permission or needs re-authorizing on One's side. Tell the
  user which platform and stop; retrying won't help.
- **404**: the id doesn't exist on that account. Verify with a read before assuming the action
  is wrong.
- **429**: rate limited. Back off; if you were looping, batch instead.
- **"Missing required OAuth scope"**: the One grant itself lacks the scope. Re-authorize via
  `/mcp`.

Report the failure with the platform's own message. Never retry a write more than once. The
first attempt may have succeeded.

## Multiple platforms in one task

Find every action in one `find_one_actions` call, then chain reads before writes. Pull from every
source first, reconcile, then write once per target.
A per-record read-then-write loop across two platforms is slow and leaves half-finished state
when it breaks.

## Report results, not just "done"

Return the created record's id or link, the count of things read, or the platform's response:
whatever lets the user verify the outcome.

## Setup and auth

The One server is remote (`https://mcp.withone.ai/mcp`) and uses OAuth. On first use Claude
Code prompts the user to sign in via `/mcp`, **One**, **Authenticate**; the browser opens One's
consent screen, where they scope this client: which connections, read or read-write, or
knowledge-only. There's no API key. If tools are missing or every call returns 401, that's the
fix: `/mcp` and (re)authenticate.

Full docs: https://www.withone.ai/docs/mcp
