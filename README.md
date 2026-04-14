# Flavonomics API v1 Reference.

Production API documentation for the external, key-protected ingredient and flavour API.

## Overview

The v1 API is designed for agent and application integrations that need:

- Ingredient search and forgiving lookup
- Semantic similarity recommendations
- Pairing recommendations with optional explainability
- Pair compatibility checks
- Raw flavour profile access
- Facet-level association insight

All v1 endpoints require an API key in the `x-api-key` header.

## Base URLs

You can call the API using either host style:

- Main site path style: `https://www.flavonomics.com/api/v1/...`
- API subdomain style: `https://api.flavonomics.com/v1/...`

Notes:

- On the API subdomain, `/v1/...` is accepted and rewritten internally to `/api/v1/...`.
- If you call `https://api.flavonomics.com/api/v1/...`, it can 404 depending on path rewrite context.

## Authentication

Send your key as a header on every request:

```bash
curl -H "x-api-key: YOUR_API_KEY" "https://www.flavonomics.com/api/v1/ingredients/search?q=lem"
```

### Obtain a key

- Sign in to the Flavonomics app
- Go to `/developers`
- Create an API key
- Copy and store the value immediately (it is shown once)

## Response Envelope

### Success

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

### Error

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

## Conventions

- `limit` and `offset` are used for pagination where applicable.
- Boolean query flags accept: `true`, `1`, `yes`, `on`.
- Scores are normalized to `0..1` unless explicitly marked as `score_norm` (`0..10`).
- `include_facets=true` enables explainability payloads on pairing-oriented endpoints.

## Endpoints

---

## `GET /api/v1/ingredients/search`

Search ingredients by partial/fuzzy name.

### Query params

- `q` (string, required)
- `limit` (int, optional, default `20`, max `100`)
- `offset` (int, optional, default `0`)

### Example

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/ingredients/search?q=ches&limit=20&offset=0"
```

### Response shape

- `data.query`
- `data.ingredients[]` with `{ ingredient, matchScore, ... }`
- `meta.pagination`

---

## `GET /api/v1/ingredients/{ingredient}/similar`

Find semantically similar ingredients.

### Query params

- `limit` (int, optional, default `10`, max `100`)
- `min_score` (float, optional, default `0`, range `0..1`)

### Example

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/ingredients/rye/similar?limit=10&min_score=0.65"
```

### Response shape

- `data.ingredient` (canonical resolved ingredient name)
- `data.strategy` (`text_embedding` or `flavour_vector`)
- `data.similar[]` with `{ ingredient, score }`
- `meta.pagination`

---

## `GET /api/v1/ingredients/{ingredient}/pairings`

Get ingredient pairings with optional facet-level explainability.

### Query params

- `limit` (int, optional, default `10`, max `200`)
- `offset` (int, optional, default `0`)
- `min_score` (float, optional, default `0`, range `0..1`)
- `include_facets` (bool, optional, default `false`)
- `exclude` (csv string, optional) e.g. `exclude=garlic,onion`

### Example (fast/default)

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/ingredients/lemon/pairings"
```

### Example (deep/explainable)

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/ingredients/lemon/pairings?limit=50&min_score=0.7&include_facets=true"
```

### Response shape

- `data.ingredient`
- `data.pairings[]`:
  - `ingredient`
  - `score` (`0..1`)
  - `score_norm` (`0..10`)
  - optional explainability fields when `include_facets=true`:
    - `relationship` (`shared`, `complementary`, `mixed`)
    - `key_facets` (map)
    - `facet_associations[]` with evidence objects
- `meta.pagination`

### Pairing response example

```json
{
  "data": {
    "ingredient": "Lemon",
    "pairings": [
      {
        "ingredient": "Thyme",
        "score": 0.87,
        "score_norm": 8.7,
        "relationship": "complementary",
        "key_facets": {
          "Citrusy": "Resinous",
          "Bright": "Herbaceous"
        }
      }
    ]
  },
  "meta": {
    "version": "v1",
    "generatedAt": "2026-02-24T12:00:00.000Z",
    "pagination": {
      "limit": 10,
      "offset": 0,
      "returned": 1,
      "total": 1
    }
  }
}
```

---

## `GET /api/v1/ingredients/{ingredient}/profile`

Get raw profile data and top flavour facets.

### Query params

- `top_facets` (int, optional, default `10`, max `40`)
- `include_vector` (bool, optional, default `false`)

### Example

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/ingredients/thyme/profile?top_facets=12&include_vector=true"
```

### Response shape

- `data.ingredient`
- `data.version`
- `data.has_overrides`
- `data.top_facets[]` with `{ facet, value }`
- `data.facets` (full facet map)
- optional `data.vector` when `include_vector=true`

---

## `GET /api/v1/pair`

Compatibility check for two or more ingredients.

### Query params

- `ingredients` (csv string, required) e.g. `ingredients=lemon,thyme`
- `include_facets` (bool, optional, default `false`)

### Example

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/pair?ingredients=lemon,thyme&include_facets=true"
```

### Response shape

- `data.ingredients[]` (canonical resolved names)
- `data.score` (`0..1`)
- `data.relationship` (`high`, `medium`, `low`)
- `data.pairwise[]` pair scores
- optional explainability when `include_facets=true`:
  - `relationship` (`shared`, `complementary`, `mixed`) facet relationship object
  - `key_facets`
  - `facet_associations`

Note: when `include_facets=true`, payload includes additional facet relationship fields to explain why ingredients work together.

---

## `GET /api/v1/facets/{facet}/associations`

Facet-level association lookup.

### Query params

- `limit` (int, optional, default `20`, max `100`)

### Path param behavior

`{facet}` accepts either:

- canonical facet key (`Citrusy`)
- display literal/name (case-insensitive)

### Example

```bash
curl -H "x-api-key: YOUR_API_KEY" \
  "https://www.flavonomics.com/api/v1/facets/Citrusy/associations?limit=15"
```

### Response shape

- `data.facet`
- `data.literal`
- `data.associations[]` with:
  - `sourceFacet`
  - `targetFacet`
  - `sourceLiteral`
  - `targetLiteral`
  - `associationScore`
- `meta.pagination`

---

## Legacy Compatibility Endpoint

## `GET /api/v1/pairings`

This endpoint is kept for backward compatibility and returns a deprecation notice in the response:

- preferred replacement:
  - `GET /api/v1/ingredients/{ingredient}/pairings`

Supported query params:

- `ingredient` (required)
- `limit`, `offset`, `min_score`, `include_facets`, `exclude`, `useDrinks`

## Error Codes and Statuses

- `400 ValidationError` - invalid query/path parameters
- `401 UnauthorizedError` - missing/revoked API key
- `404 NotFoundError` - ingredient or facet not found
- `500 InternalError` - unexpected server error

## Operational Guidance

- **Timeout/retry**: use client retries with exponential backoff for transient `5xx`.
- **Pagination**: use `meta.pagination.nextOffset` when present.
- **Explainability cost**: `include_facets=true` adds extra computation and payload size.
- **Input normalization**: ingredient/facet lookup is forgiving, but canonical names in responses should be used for follow-up calls.
- **Versioning**: integrate against `/api/v1/...`; future versions may evolve scoring and fields.

## Integration Quick Start

1. Create a key in `/developers`.
2. Start with:
   - `GET /api/v1/ingredients/search`
   - `GET /api/v1/ingredients/{ingredient}/pairings`
3. Enable deeper reasoning with `include_facets=true`.
4. Use `/api/v1/pair` for validation before recipe assembly.
5. Use `/api/v1/facets/{facet}/associations` to produce explicit flavour explanations.
