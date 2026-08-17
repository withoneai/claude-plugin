---
name: integration-code
description: >-
  Write integration code against a third-party API — Gmail, Stripe, Shopify, HubSpot, QuickBooks,
  Slack, Salesforce, Notion, Linear and 500+ more — using the API's real schema from One instead of
  a guessed one. Use when building a feature that calls an external SaaS API, generating a client,
  route handler, or webhook handler, adding a connection to an app, or debugging an integration
  that returns 400/401/422. Uses One's read tools (search + knowledge) and then writes code; does
  not execute live calls.
license: MIT
allowed-tools:
  - mcp__plugin_one_one__list_one_integrations
  - mcp__plugin_one_one__search_one_platform_actions
  - mcp__plugin_one_one__get_one_action_knowledge
---

# Writing integration code with real API schemas

Integration code fails on details a model can't recall: the exact field name, whether a value
goes in the body or the query string, which enum the API accepts *this year*. One's action
knowledge carries all of it — so look the API up rather than writing from memory.

This is what One's **knowledge-only mode** is for: on the consent screen the user can drop
`execute_one_action`, so you can read every API's real schema while coding and can't fire a
live request at production data by accident. The workflow below is the same either way — you
just never call execute when the deliverable is code.

## Before you write the call

1. `list_one_integrations` — confirm the platform slug (and, if you're going to run through One,
   grab the connection `key`).
2. `search_one_platform_actions` on that `platform`, `query` described by outcome ("create a
   customer", "list orders since a date"). Pass `agent_type: "knowledge"` if you want to bias
   toward documentation-rich results.
3. `get_one_action_knowledge` with `action_id` + `platform` for the action you picked. Read
   the whole thing: required and optional parameters, types, enums, auth, the request shape,
   the response shape, and the gotchas section.
4. Write the code **from that schema**. Field names, casing, nesting, and types come from the
   knowledge, not from what the API probably looks like.

If you write a request body with a field the knowledge doesn't list, you invented it. Go back
and check.

## Two ways to ship the call — pick one and say which

**Through One (Passthrough API).** Keep One in the runtime. The user's One connection handles
the platform's auth, so your code holds no per-platform tokens and no refresh logic. In
knowledge-only mode the knowledge response ends with an *Integration Code Guide* that spells
this out: `POST/GET https://api.withone.ai/v1/passthrough<action path>` with headers
`x-one-secret` (from `ONE_SECRET`), `x-one-connection-key` (from an env var like
`ONE_GMAIL_CONNECTION_KEY`), and `x-one-action-id` (the action's id). Follow that guide
verbatim — path variables into the URL, query params into the query string, body fields into
JSON. Good when the app already uses One or integrates several platforms.

**Direct to the platform.** Use the knowledge purely as documentation and write a normal HTTP
client against the vendor's own base URL. You then own OAuth, token refresh, and rotation. Good
when the integration is one platform deep or the runtime can't take another dependency.

State which you're doing and why in one sentence before you write the file. Don't mix them in
the same module.

## Types come from the response shape

The knowledge includes the response shape. Generate types from it instead of hand-writing an
interface that drifts. Model optional fields as optional. When the API returns a union or a
nullable field, represent that rather than asserting the happy path.

## Auth belongs in the environment

Never write a key, token, or secret into source, a config file, a test fixture, or a committed
`.env`. Read from the environment and add the variable name to `.env.example` so the next
person knows what to set. If the code is a client-side component, the call goes through a
server route — a browser bundle can't hold a platform credential or a One secret. When you
deliver the code, tell the user explicitly which env vars to set and where (Vercel/Netlify env,
Supabase secrets, etc.).

## Pagination and rate limits are not optional

The knowledge names the pagination parameters and the platform's limits. A list call that
ignores them works on a test account with twelve records and fails on a real one. Write the
loop with a cursor and a bound, and handle 429 with a backoff in the first version, not the
second.

## Errors

Handle the four that actually happen: **400/422** (payload doesn't match the schema), **401/403**
(auth expired or scope missing), **404** (record isn't on this account), **429** (slow down).
Surface the platform's own error message to the caller. Swallowing it into "something went
wrong" makes the integration undebuggable.

## Webhooks

If the feature reacts to platform events rather than polling, look up the platform's webhook
action the same way. Verify the signature, respond fast, and do the work asynchronously. An
unverified webhook endpoint is an open write path into your system.

## Debugging an integration that's already broken

Re-read the knowledge for the action before changing anything. Most 400s are a renamed field or
a value in the wrong place. Compare the failing request against the schema field by field. If
auth is failing, check the scope granted on the connection before touching the code.

## Don't execute

The deliverable here is source code. Even if `execute_one_action` is available, don't call it
to "test" — the user's connections point at real accounts. If they want a live smoke test, ask
first and use the `integrations` skill's confirm-before-write discipline.

Full docs: https://www.withone.ai/docs/mcp
