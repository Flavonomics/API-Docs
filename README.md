# Flavonomics Developer Reference

Production reference for both Flavonomics integration surfaces:

- The HTTP API for direct application requests
- The remote MCP server for agent clients such as Claude, Cursor, and other MCP-compatible tools

## Choose The Right Surface

### HTTP API

Use the HTTP API when you want:

- Stable endpoint-by-endpoint integrations
- Predictable request and response envelopes
- Explicit control over pagination, filtering, and retries
- Server-to-server or frontend-to-backend product integrations

### Remote MCP

Use the remote MCP server when you want:

- An agent client to discover Flavonomics as a tool surface
- Tool calls instead of manual endpoint orchestration
- Resolved ingredients, flavour evidence, pairings, and co-occurrence exposed as MCP tools
- A single remote SSE connection instead of multiple HTTP endpoint calls

## Keys And Where To Create Them

Flavonomics uses two different key types.

Both key types are available on the Enterprise plan only. If you are on the Flavour Society membership and need API or MCP access, upgrade to Enterprise and Flavonomics will apply a pro-rated refund for the current month of your Flavour Society subscription.

### HTTP API keys

- Create and revoke them from `https://www.flavonomics.com/developer` with an Enterprise account
- Send them in the `x-api-key` header

### MCP keys

- Create and revoke them from `https://www.flavonomics.com/account` with an Enterprise account
- Send them as `Authorization: Bearer YOUR_MCP_KEY` on the SSE connection
- The server also accepts `?api_key=YOUR_MCP_KEY` on `/sse` as a legacy fallback for clients that cannot send headers

## HTTP API

### Base URLs

You can call the API using either host style:

- Main site path style: `https://www.flavonomics.com/api/v1/...`
- API subdomain style: `https://api.flavonomics.com/v1/...`

Notes:

- On the API subdomain, `/v1/...` is rewritten internally to `/api/v1/...`
- `https://api.flavonomics.com/api/v1/...` can 404 depending on rewrite context, so prefer `/v1/...` there

### Authentication

Send your key on every request:

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/ingredients/search?q=lem"
```

Programmatic HTTP access is Enterprise-only. Valid keys owned by non-Enterprise accounts are rejected.

### Response Envelope

Success:

```json
{
  "data": {},
  "meta": {
    "version": "v1",
    "generatedAt": "2026-02-24T12:00:00.000Z",
    "pagination": {
      "limit": 10,
      "offset": 0,
      "returned": 10,
      "total": 120,
      "nextOffset": 10
    }
  }
}
```

Error:

```json
{
  "error": {
    "message": "Unauthorized. Missing API key.",
    "code": "UnauthorizedError",
    "details": {}
  },
  "meta": {
    "version": "v1",
    "generatedAt": "2026-02-24T12:00:00.000Z"
  }
}
```

### Conventions

- `limit` and `offset` are used for pagination where applicable
- Boolean query flags accept `true`, `1`, `yes`, `on`
- Scores are normalized to `0..1` unless explicitly marked as `score_norm` (`0..10`)
- `include_facets=true` enables explainability payloads on pairing-oriented endpoints

### Endpoints

#### `GET /api/v1/ingredients/search`

Search ingredients by partial or fuzzy name.

Query params:

- `q` (string, required)
- `limit` (int, optional, default `20`, max `100`)
- `offset` (int, optional, default `0`)

Example:

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/ingredients/search?q=ches&limit=20&offset=0"
```

Response shape:

- `data.query`
- `data.ingredients[]` with `{ ingredient, matchScore, ... }`
- `meta.pagination`

#### `GET /api/v1/ingredients/{ingredient}/similar`

Find semantically similar ingredients.

Query params:

- `limit` (int, optional, default `10`, max `100`)
- `min_score` (float, optional, default `0`, range `0..1`)

Example:

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/ingredients/rye/similar?limit=10&min_score=0.65"
```

Response shape:

- `data.ingredient` (canonical resolved ingredient name)
- `data.strategy` (`text_embedding` or `flavour_vector`)
- `data.similar[]` with `{ ingredient, score }`
- `meta.pagination`

#### `GET /api/v1/ingredients/{ingredient}/pairings`

Get ingredient pairings with optional facet-level explainability.

Query params:

- `limit` (int, optional, default `10`, max `200`)
- `offset` (int, optional, default `0`)
- `min_score` (float, optional, default `0`, range `0..1`)
- `include_facets` (bool, optional, default `false`)
- `exclude` (csv string, optional), for example `exclude=garlic,onion`

Example:

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/ingredients/lemon/pairings?limit=50&min_score=0.7&include_facets=true"
```

Response shape:

- `data.ingredient`
- `data.pairings[]`
- `meta.pagination`

Pairing items include:

- `ingredient`
- `score` (`0..1`)
- `score_norm` (`0..10`)
- optional explainability fields when `include_facets=true`:
  - `relationship` (`shared`, `complementary`, `mixed`)
  - `key_facets`
  - `facet_associations[]`

#### `GET /api/v1/ingredients/{ingredient}/profile`

Get raw profile data and top flavour facets.

Query params:

- `top_facets` (int, optional, default `10`, max `40`)
- `include_vector` (bool, optional, default `false`)

Example:

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/ingredients/thyme/profile?top_facets=12&include_vector=true"
```

Response shape:

- `data.ingredient`
- `data.version`
- `data.has_overrides`
- `data.top_facets[]` with `{ facet, value }`
- `data.facets`
- optional `data.vector`

#### `GET /api/v1/pair`

Compatibility check for two or more ingredients.

Query params:

- `ingredients` (csv string, required), for example `ingredients=lemon,thyme`
- `include_facets` (bool, optional, default `false`)

Example:

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/pair?ingredients=lemon,thyme&include_facets=true"
```

Response shape:

- `data.ingredients[]`
- `data.score` (`0..1`)
- `data.relationship` (`high`, `medium`, `low`)
- `data.pairwise[]`
- optional explainability when `include_facets=true`

#### `GET /api/v1/facets/{facet}/associations`

Facet-level association lookup.

Query params:

- `limit` (int, optional, default `20`, max `100`)

Path param behavior:

`{facet}` accepts either:

- canonical facet key such as `Citrusy`
- display literal or display name, case-insensitive

Example:

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/facets/Citrusy/associations?limit=15"
```

Response shape:

- `data.facet`
- `data.literal`
- `data.associations[]`
- `meta.pagination`

#### `GET /api/v1/pairings`

Legacy compatibility endpoint.

This endpoint is kept for backward compatibility and returns a deprecation notice in the response.

Preferred replacement:

- `GET /api/v1/ingredients/{ingredient}/pairings`

Supported query params:

- `ingredient` (required)
- `limit`
- `offset`
- `min_score`
- `include_facets`
- `exclude`
- `useDrinks`

### Error Codes And Statuses

- `400 ValidationError` for invalid query or path parameters
- `401 UnauthorizedError` for missing, revoked, or inactive-plan API keys
- `403 ForbiddenError` for authenticated accounts without Enterprise programmatic access
- `404 NotFoundError` for unknown ingredients or facets
- `500 InternalError` for unexpected server errors

### Operational Guidance

- Use client retries with exponential backoff for transient `5xx`
- Use `meta.pagination.nextOffset` when present
- `include_facets=true` increases compute cost and payload size
- Canonical names returned by the API should be reused for follow-up calls
- Integrate against `/api/v1/...` or `https://api.flavonomics.com/v1/...`

## Remote MCP

### Transport

The Flavonomics MCP server uses remote SSE transport.

- SSE endpoint: `https://www.flavonomics.com/sse`
- Message endpoint: `https://www.flavonomics.com/message`
- Server name: `flavonomics-mcp`

In normal MCP client usage you connect to `/sse`, the server authenticates the session, and the client then posts subsequent messages to `/message?sessionId=...` automatically.

### Authentication

Preferred:

```http
Authorization: Bearer YOUR_MCP_KEY
```

Remote MCP access is Enterprise-only. Non-Enterprise accounts cannot create MCP keys or connect to the remote server.

Legacy fallback on the SSE URL only:

```text
https://www.flavonomics.com/sse?api_key=YOUR_MCP_KEY
```

### Generic Client Configuration Example

```json
{
  "mcpServers": {
    "flavonomics": {
      "transport": "sse",
      "url": "https://www.flavonomics.com/sse",
      "headers": {
        "Authorization": "Bearer YOUR_MCP_KEY"
      }
    }
  }
}
```

If your MCP client does not support custom headers, use the query-string fallback on the SSE URL instead.

### Available MCP Tools

#### `query_database`

Execute a read-only SQL `SELECT` query against the Flavonomics Postgres database.

#### `resolve_ingredients`

Resolve ingredient strings to canonical Flavonomics ingredients using direct lookup, fuzzy search, and embedding fallback.

#### `get_ingredient_profiles`

Return canonical ingredient profiles, dominant flavour notes, and chart-ready flavour vectors.

Inputs:

- `ingredients` (string array, required)
- `topNotesLimit` (int, optional, `1..20`)

#### `get_flavour_associations`

Return dominant flavour facets and the strongest associated flavour notes for canonical ingredients.

Inputs:

- `ingredients` (string array, required)
- `topFlavourLimit` (int, optional, `1..20`)
- `targetLimit` (int, optional, `1..20`)

#### `get_recommended_pairings`

Return ranked ingredient pairing recommendations and chart-ready profile payloads for those recommendations.

Inputs:

- `ingredients` (string array, required)
- `limit` (int, optional, `1..20`)

#### `get_recipe_cooccurrence`

Return recipe co-occurrence evidence and uplift metrics for the canonical ingredients.

Inputs:

- `ingredients` (string array, required)
- `pairingLimit` (int, optional, `1..20`)
- `frequencyLimit` (int, optional, `1..20`)

#### `get_explanation_evidence`

Assemble structured flavour evidence, including profiles, associations, pairings, co-occurrence, and optional seasonal context.

Inputs:

- `ingredients` (string array, required)
- `flavourNotes` (string array, optional)
- `countryCode` (2-letter ISO country code, optional)
- `month` (string, optional)

### MCP Operational Notes

- Authentication happens during the `/sse` handshake
- Per-key rate limits are enforced server-side
- Invalid keys or non-Enterprise plans return `401` or `403` during connection setup
- The remote MCP surface and the HTTP API use the same underlying flavour intelligence, but they expose it differently

## Integration Quick Start

1. Upgrade to the Enterprise plan if you need programmatic access. Flavour Society members who upgrade receive a pro-rated refund for the current month.
2. Create an HTTP API key on `/developer` if you want endpoint-level integration.
3. Create an MCP key on `/account` if you want a remote tool server for agents.
4. Start with `GET /api/v1/ingredients/search` or `resolve_ingredients`.
5. Move to pairings with `GET /api/v1/ingredients/{ingredient}/pairings` or `get_recommended_pairings`.
6. Add deeper explainability with `include_facets=true` in HTTP or `get_explanation_evidence` in MCP.
