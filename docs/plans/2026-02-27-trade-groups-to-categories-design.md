# Trade Groups → Categories Rename

**Date:** 2026-02-27
**Status:** Approved
**Approach:** Full public-facing rename (Approach A)

## Context

"Trade groups" is being renamed to "categories" across all public-facing surfaces. The backend keeps its internal `trade_group` naming — Fordje Connect translates at the gateway boundary.

- Pre-launch, no existing API consumers — breaking change is acceptable
- Backend is read-only, not being modified

## Scope

### Public API (`fordje-public-api/`)

| Current | New |
|---------|-----|
| `GET /v1/trade-groups` | `GET /v1/categories` |
| `?trade_group_ids=1,2` (on `/v1/data-points`) | `?category_ids=1,2` |
| `TradeGroupResponse` schema | `CategoryResponse` schema |
| `CollectionResponse.trade_group_ids` | `CollectionResponse.category_ids` |
| `get_trade_groups()` backend client | `get_categories()` (still calls `/internal/trade-groups`) |

#### Files to change

| File | Change |
|------|--------|
| `app/routes/reference_data.py` | Rename route path, function, descriptions |
| `app/routes/data_points.py` | Rename `trade_group_ids` query param to `category_ids` |
| `app/routes/collections.py` | Update field references and descriptions |
| `app/schemas/reference_data.py` | `TradeGroupResponse` → `CategoryResponse` |
| `app/schemas/collections.py` | `trade_group_ids` → `category_ids` |
| `app/services/backend_client.py` | Rename function, keep internal endpoint path |
| `app/config/settings.py` | Update comment |
| `app/main.py` | Update if trade-groups referenced in router tags |
| `PUBLIC_API_SPEC.md` | Update spec |
| `INTERNAL_ENDPOINTS_REQUIRED.md` | Add mapping note |
| `CLAUDE.md` | Update endpoint list |
| `tests/test_reference_data.py` | Rename test class and assertions |
| `tests/test_data_points_scoped.py` | Update param names |
| `tests/test_collections_execution.py` | Update fixtures |

### Docs (`docs/`)

| File | Change |
|------|--------|
| `concepts/trade-groups.mdx` | Rename to `categories.mdx`, rewrite content |
| `docs.json` | Update nav paths |
| `concepts/index.mdx` | Update accordion text |
| `concepts/data-points.mdx` | Update filter references |
| `concepts/collections.mdx` | Update filter references |
| `getting-started/quickstart.mdx` | Update mention |
| `api-reference/introduction.mdx` | Update endpoint listing |
| `api-reference/openapi.json` | Rename paths, params, descriptions |
| `concepts/rate-limiting.mdx` | Update cache mention |
| `AGENTS.md` | Update terminology |

## Architecture Decision

The backend client (`backend_client.py`) is the translation boundary:

```python
async def get_categories(...) -> dict:
    """Fetch categories (mapped from backend trade-groups)."""
    return await self._get("/internal/trade-groups", params=...)
```

Query param mapping: public `category_ids` → backend `group_ids`.

## What Stays the Same

- Backend model, table, routes — untouched
- Redis cache keys — internal implementation detail
- Connect's own database — no trade group tables to change
