---
layout: post
title:  "Agents Plugin API"
date:   2026-06-11 10:37:00 +0100
categories: mattermost ai agents plugin api
---

# Provisioning Mattermost AI Agents by API: A Field Guide to the Agents Plugin's Undocumented REST Interface

If you automate Mattermost deployments, you may have recently had the same unpleasant surprise I did: a build pipeline that had configured the AI plugin perfectly happily for months suddenly produced environments with no AI configured at all — and not an error message in sight.

The cause is a significant (and sensible, but breaking) architectural change in the **Mattermost Agents plugin** (`mattermost-plugin-agents`). As of v2.x, the plugin no longer reads its configuration from `config.json`. Everything — LLM service connections *and* the agents themselves — now lives in the database. For an existing installation being upgraded, one-time migrations handle the transition invisibly. But if you build environments from scratch, as any self-respecting demo, test, or ephemeral-environment pipeline does, `mmctl config patch` silently stops being the answer. The plugin sees an empty configuration on first activation, saves it as the active one, and from that moment on your carefully patched `config.json` is decorative.

The replacement is the plugin's own REST API. The catch: it isn't publicly documented yet. The good news: it's small, sensible, and stable enough to build on — it's the same API the plugin's admin console calls. What follows is a field guide assembled by reading the plugin source (verified against tag `v2.1.0`), covering everything you need to provision AI services and agents programmatically.

> **A word of caution before we start.** The Agents plugin exposes its own HTTP API — the same one its React admin webapp calls — and that route contract isn't versioned or stability-guaranteed: Mattermost can change it between plugin releases without notice. There's no alternative interface for this job, so it's entirely appropriate for provisioning and automation — but treat it as "the Agents plugin API", pin your plugin version, and re-verify when you upgrade.

## The shape of the thing

All endpoints are served by the plugin under:

```
https://<your-mattermost>/plugins/mattermost-ai
```

(The plugin ID remains `mattermost-ai` even though the product is now branded "Agents" — a small archaeological detail that will save you a confused half-hour.)

Authentication is a normal Mattermost session: a Personal Access Token or session token in the `Authorization: Bearer <token>` header. Mattermost validates the session and injects a trusted `Mattermost-User-Id` header that the plugin reads — clients can't set or spoof it. There are two authorisation tiers: the agent endpoints accept any authenticated user (with per-agent access checks), while the configuration endpoints demand a system admin. For provisioning, use a system-admin token and both tiers are satisfied.

Conceptually, the API splits into two surfaces, mirroring the new storage model:

1. **Configuration** (`/admin/config`) — LLM service connections and global settings, stored as a versioned configuration history in the database with exactly one active row.
2. **Agents** (`/agents`) — the bots themselves, stored in their own table, each backed by a real Mattermost bot account.

This split matters: agents are no longer part of the configuration blob at all. If your tooling writes bots into the config, it's time to migrate it to the agents endpoints.

## Surface one: services and global settings

### `PUT /plugins/mattermost-ai/admin/config`

This is the spiritual successor to `mmctl config patch` for everything except the agents. The body is a full `Config` object — services, default agent, MCP settings, web search, embedding search, telemetry, and the various global toggles. A successful call returns `200 OK` with an empty body, writes a new active configuration row (history is preserved), refreshes the plugin's in-memory state, and notifies cluster nodes.

```bash
curl -sS -X PUT "$MM/plugins/mattermost-ai/admin/config" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "services": [{
      "id": "svc-openai-1",
      "name": "OpenAI",
      "type": "openai",
      "apiKey": "sk-...",
      "defaultModel": "gpt-4o",
      "tokenLimit": 128000,
      "outputTokenLimit": 4096
    }],
    "bots": [],
    "allowUnsafeLinks": false,
    "enableChannelMentionToolCalling": true
  }'
```

A few details worth knowing about the `services` entries:

- `type` accepts `openai`, `openaicompatible`, `azure`, `anthropic`, `cohere`, `bedrock`, `mistral`, `scale`, `gemini`, or `vertex`, with provider-specific credential fields (`region` and AWS keys for Bedrock, project details for Vertex, and so on).
- `tokenLimit` is the **input** token limit — the JSON key keeps its old name for backwards compatibility, with `outputTokenLimit` alongside it.
- Each service's `id` is the stable identifier that every agent will reference via `serviceID`. Choose it deliberately.
- Leave `bots` as an empty array. The field still exists for legacy reasons, but agents belong to the other surface now.

### `GET /plugins/mattermost-ai/admin/config`

Reads the active configuration back — useful as a verification gate in a pipeline. One behaviour to be aware of: the response passes through a normalisation step that **force-sets `useResponsesAPI: true` on every OpenAI-type service**. A freshly read configuration may therefore differ from what you wrote, by design. If your OpenAI-compatible endpoint can't speak the Responses API, define the service as `openaicompatible` instead, where the toggle is respected.

### `GET /plugins/mattermost-ai/services`

Lists the configured services with all secret fields stripped — handy for discovering the real `serviceID` values before creating agents, and available to ordinary authenticated users with agent-configuration permission rather than admins only.

## Surface two: agents

### `POST /plugins/mattermost-ai/agents`

Creates an agent — and, crucially, **provisions a real Mattermost bot account behind it**. You don't create the bot separately; the plugin does it, and rolls it back if persistence fails. The response is `201 Created` with the full agent object, including the `id` you'll need for every subsequent per-agent call.

```bash
curl -sS -X POST "$MM/plugins/mattermost-ai/agents" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "displayName": "YodaAI",
    "username": "yoda-ai",
    "serviceID": "svc-openai-1",
    "customInstructions": "Respond with wisdom, like a Jedi Master. Reverse your sentences, you must.",
    "teamIDs": ["<team-id>"],
    "userAccessLevel": 0,
    "channelAccessLevel": 0,
    "enableVision": true
  }'
```

The request shape is rich but logical. The essentials: `displayName`, `username` (must match `^[a-z][a-z0-9._-]*$`, becomes the bot's @-handle, and is **immutable after creation**), and `serviceID` (which must reference an existing service, or you'll get a `400`). Beyond those, you can scope the agent with `teamIDs`, plus channel and user access controls (`channelAccessLevel`/`userAccessLevel` as integer enums — `0` All, `1` Allow, `2` Block, `3` None — paired with `channelIDs`/`userIDs` lists), set a per-agent `model` override, `customInstructions` as the system prompt, vision and tool toggles, reasoning controls (`reasoningEnabled`, `reasoningEffort`, `thinkingBudget`), per-agent MCP tool allow-lists, and `adminUserIDs` to delegate management of the agent.

### The rest of the agent surface

- `GET /agents` — lists the agents the caller may access; `customInstructions` is redacted for agents the caller can't manage.
- `GET /agents/:agentid` — fetches one agent, `404` if absent or inaccessible.
- `PUT /agents/:agentid` — full replace, same shape as create, `username` excepted. Bodies are capped at 512 KiB.
- `DELETE /agents/:agentid` — soft-deletes the agent and deactivates its bot account.
- `POST /agents/:agentid/avatar` — sets the agent's icon via `multipart/form-data` with an `image` field (10 MB cap). Internally this sets the bot account's profile image, so standard Mattermost image handling applies.
- `POST /agents/models/fetch` — given a `serviceID`, queries the provider with its stored credentials and returns the available models. A nice touch for building tooling.

One identifier pitfall: each agent object carries both its own `id` *and* a `botUserID` for the backing bot account. The per-agent routes want the **agent's `id`** — mixing the two up produces confusing `404`s.

## Licensing: the quota that bites at provisioning time

Agent creation is licence-gated. Without at least an Enterprise-tier licence, the API permits exactly **one** agent server-wide; the second create returns `403` with a message telling you that more requires an E20 or Enterprise licence. With the licence in place, the check short-circuits to unlimited.

For interactive use, that's a clear error at an obvious moment. For automation, it's a sequencing constraint: **apply your licence before you create agents**. A pipeline that seeds agents before licensing will create one agent, then fail twelve times in a row — and if it doesn't fail fast, you'll be diagnosing a half-seeded server instead of reading one honest error.

## The end-to-end provisioning recipe

Putting it together, the build-from-scratch order is strict and worth committing to memory:

**licence → configuration → agents → avatars**

```bash
MM="https://your-mattermost.example.com"
TOKEN="<system-admin token>"

# 1. Seed services + global settings (the mmctl config patch replacement)
curl -sS -X PUT "$MM/plugins/mattermost-ai/admin/config" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d @ai-services.json

# 2. Discover the serviceID
SVC_ID=$(curl -sS "$MM/plugins/mattermost-ai/services" \
  -H "Authorization: Bearer $TOKEN" | jq -r '.[] | select(.name=="OpenAI") | .id')

# 3. Create each agent
curl -sS -X POST "$MM/plugins/mattermost-ai/agents" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"displayName\":\"Assistant\",\"username\":\"ai\",\"serviceID\":\"$SVC_ID\",\"teamIDs\":[\"$TEAM_ID\"]}"

# 4. Optionally, give it a face
curl -sS -X POST "$MM/plugins/mattermost-ai/agents/$AGENT_ID/avatar" \
  -H "Authorization: Bearer $TOKEN" -F "image=@assistant.png"
```

Two behaviours shape any robust implementation of this flow. First, because each agent is a real bot account, usernames are globally unique and a collision returns `409` — which makes **create-or-update by username** the natural idempotent pattern: list the agents, create the absent, `PUT` the present. Second, `serviceID` is validated at agent-creation time, so configuration must land before agents, always.

For error handling, lean on status codes: `400` for validation failures (create/update validation errors do return a JSON `{"error": "..."}` body), `401` for missing sessions, `403` for admin or licence problems, `404` for missing agents, `409` for username collisions, `413` for oversized bodies or avatars. Several endpoints return empty bodies on failure, so the status code is the reliable signal.

## Closing thoughts

The move from a configuration blob to a proper API is, breaking change aside, a genuine improvement: versioned configuration history, validated writes, agents as first-class objects with real lifecycle semantics, and bot account management handled for you. The transitional pain is concentrated entirely on those of us who provision from scratch — and the fix is a pleasantly small amount of code against a pleasantly small API.

Until official documentation arrives, treat this guide as a snapshot: verified against plugin v2.1.0 by reading the source, accurate today, and worth re-checking against the [plugin repository](https://github.com/mattermost/mattermost-plugin-agents) whenever you take a plugin upgrade. The API your automation depends on is the same one the admin console depends on — which is about the best stability guarantee an undocumented API can offer.