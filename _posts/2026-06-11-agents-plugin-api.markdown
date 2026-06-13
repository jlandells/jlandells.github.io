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

Conceptually, the API splits into three surfaces, mirroring the new storage model:

1. **Configuration** (`/admin/config`) — LLM service connections and global settings, stored as a versioned configuration history in the database with exactly one active row.
2. **Agents** (`/agents`) — the bots themselves, stored in their own table, each backed by a real Mattermost bot account.
3. **Custom prompts** (`/custom-prompts`) — saved, optionally shared prompt templates that users run or pin as one-click buttons.

The split matters: agents are no longer part of the configuration blob at all. If your tooling writes bots into the config, it's time to migrate it to the agents endpoints.

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

### MCP tools: the default that will surprise you

If you use MCP (Model Context Protocol) tools, the `mcp` block in the configuration is where they're governed — a master `enabled` switch, an idle timeout, a list of external `servers`, and the built-in `embeddedServer` that exposes Mattermost's own tools. Within each server, individual tools carry a `tool_configs` list that sets a per-tool approval policy. There are three policies: `ask` (prompt the user before every call), `auto_run_in_dm` (run without prompting, but only in direct messages), and `auto_run_everywhere` (run without prompting in both DMs and channels).

Here's the part that catches people out: **an unconfigured tool defaults to `ask` with `enabled: true`** — the agent stops and asks for approval before each call. For an interactive user that's a reasonable safety default. For a demo or an automated workflow where you want tools to *just work*, it's a wall of approval prompts. Making a tool run silently requires an explicit `tool_configs` entry setting its policy.

That, in turn, means you need the tool names — and you must not hard-code them. The embedded tool set is **dynamic**: automation tools appear only when the channel-automation plugin is present, developer tools only in dev mode, and the set drifts between plugin versions. So discover it live:

### `GET /plugins/mattermost-ai/admin/mcp/tools`

Returns every configured MCP server and the tools it currently exposes, discovered at call time (system admin). Enumerate from here before writing any `tool_configs` policy.

```json
{
  "servers": [
    {
      "name": "Mattermost",
      "url": "embedded://mattermost",
      "tools": [
        { "name": "read_post", "description": "...", "inputSchema": {} },
        { "name": "search_users", "description": "...", "inputSchema": {} }
      ],
      "needsOAuth": false,
      "error": null
    }
  ]
}
```

The practical seeding pattern is a two-step within the configuration surface: discover the live tool list, then `PUT` a configuration whose `embeddedServer.tool_configs` sets each tool to whatever policy your environment needs (`auto_run_everywhere`, if you want a demo that never stops to ask). Two related admin endpoints are handy here too: `GET /admin/mcp/vetted-tool-seed` returns Mattermost's curated default policies for a vetted server, and `POST /admin/mcp/tools/cache/clear` forces rediscovery.

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

## Surface three: custom prompts

Newer plugin versions add a third surface worth knowing about: **custom prompts** (`/custom-prompts`) — saved prompt templates that users run from the message composer or pin as one-click buttons at the top of the Agents pane. They're a neat way to ship a workflow: rather than teaching everyone to type a paragraph-long instruction, you give them a button labelled "Daily Summary" that posts the rendered template instantly.

The data model differs from agents in two important ways. First, prompts are **owned by their creator** — the authenticated session user — with no team scope; `creator_id` is set server-side and ignored if you send it. Visibility is binary: private to the creator, or shared workspace-wide via `is_shared`. Second, there's **no admin or licence gate** — any authenticated user can create one.

The template itself supports Go `text/template` with a whitelist of context variables (`{{.Username}}`, `{{.Channel}}`, `{{.ChannelName}}`, `{{.Team}}`, `{{.TeamName}}`, `{{.Time}}`, `{{.BotName}}`), so a prompt can adapt to wherever it's run.

```bash
curl -sS -X POST "$MM/plugins/mattermost-ai/custom-prompts" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "name": "Daily Summary",
    "description": "Summarise the last 24h of alerts and post a report",
    "template": "Read the last 24 hours of alerts in the ~alerts channel and post a summary to the ~reports channel",
    "is_shared": true
  }'
```

The surface rounds out predictably: `GET /custom-prompts` lists your own prompts plus shared ones from others; `PUT /custom-prompts/:id` and `DELETE /custom-prompts/:id` are creator-only; and pinning is handled separately by `GET` and `PUT /custom-prompts/pins`. Pinning is **per-user** — pinning a shared prompt pins it only for you — and a pinned prompt becomes a one-click button that posts the rendered template immediately. (There's also a `/render` endpoint that previews a template against the current context, used by the composer; you won't need it for seeding.)

The one gotcha for automation: **there is no upsert**. Re-POSTing the same prompt creates a duplicate rather than replacing it, so a seeder must list first, match by name (and `creator_id`, since the list includes others' shared prompts), and decide between create and update itself. The same defensive pattern the agents surface needs, for a different underlying reason.

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

If you use MCP tools and want them to run without prompting, slot a tool-policy step in between configuration and agents: call `GET /admin/mcp/tools` to enumerate the live tool set, then include those tools in `embeddedServer.tool_configs` (each set to `auto_run_everywhere`, say) in the configuration you `PUT`. And if you're seeding custom prompts, that's an independent final step — list existing prompts, create or update by name, then pin the ones you want surfaced as buttons.

Two behaviours shape any robust implementation of this flow. First, because each agent is a real bot account, usernames are globally unique and a collision returns `409` — which makes **create-or-update by username** the natural idempotent pattern: list the agents, create the absent, `PUT` the present. The same defensive habit applies to custom prompts, which have no upsert at all. Second, `serviceID` is validated at agent-creation time, so configuration must land before agents, always.

For error handling, lean on status codes: `400` for validation failures (create/update validation errors do return a JSON `{"error": "..."}` body), `401` for missing sessions, `403` for admin or licence problems, `404` for missing agents, `409` for username collisions, `413` for oversized bodies or avatars. Several endpoints return empty bodies on failure, so the status code is the reliable signal.

## Closing thoughts

The move from a configuration blob to a proper API is, breaking change aside, a genuine improvement: versioned configuration history, validated writes, agents as first-class objects with real lifecycle semantics, bot account management handled for you, and — as the surface has grown — per-tool MCP approval policies and shareable, pinnable prompt templates besides. The transitional pain is concentrated entirely on those of us who provision from scratch, and the fix is a pleasantly small amount of code against a pleasantly small API.

Until official documentation arrives, treat this guide as a snapshot: verified against plugin v2.1.0 by reading the source, accurate today, and worth re-checking against the [plugin repository](https://github.com/mattermost/mattermost-plugin-agents) whenever you take a plugin upgrade. The API your automation depends on is the same one the admin console depends on — which is about the best stability guarantee an undocumented API can offer.