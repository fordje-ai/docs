# Trade Groups → Categories Rename — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Rename "trade groups" to "categories" across all public-facing surfaces (API endpoints, schemas, docs) while keeping backend naming unchanged.

**Architecture:** Fordje Connect is a stateless gateway. The backend keeps `trade_group` naming internally — the public API translates at the boundary. `backend_client.py` is the mapping layer: public `get_categories()` calls backend `/internal/trade-groups`, and public `category_ids` maps to backend `group_ids`.

**Tech Stack:** FastAPI, Pydantic v2, pytest, Mintlify (MDX docs)

---

### Task 1: Create feature branch

**Step 1: Create and checkout branch in fordje-public-api**

```bash
cd /Users/jongamble/repos/public-api-workspace/fordje-public-api
git checkout -b feature/rename-trade-groups-to-categories
```

**Step 2: Create and checkout branch in docs**

```bash
cd /Users/jongamble/repos/public-api-workspace/docs
git checkout -b feature/rename-trade-groups-to-categories
```

---

### Task 2: Rename schema classes

**Files:**
- Modify: `fordje-public-api/app/schemas/reference_data.py`
- Modify: `fordje-public-api/app/schemas/collections.py`

**Step 1: Rename TradeGroupResponse → CategoryResponse**

In `app/schemas/reference_data.py`, replace:

```python
"""Reference data schemas for trade groups and units of measure."""

# ...

class TradeGroupResponse(BaseModel):
    """A trade group classification for data point types."""

    id: int = Field(..., description="Trade group ID", examples=[1])
    name: str = Field(..., description="Trade group name", examples=["Electrical"])
```

With:

```python
"""Reference data schemas for categories and units of measure."""

# ...

class CategoryResponse(BaseModel):
    """A category classification for data point types."""

    id: int = Field(..., description="Category ID", examples=[1])
    name: str = Field(..., description="Category name", examples=["Electrical"])
```

**Step 2: Rename trade_group_ids → category_ids in CollectionResponse**

In `app/schemas/collections.py`, replace:

```python
    trade_group_ids: Optional[list[int]] = Field(
        None, description="Trade group filter IDs"
    )
```

With:

```python
    category_ids: Optional[list[int]] = Field(
        None, description="Category filter IDs", alias="trade_group_ids"
    )
```

Note: The `alias="trade_group_ids"` tells Pydantic to accept `trade_group_ids` from the backend response and map it to the public-facing `category_ids` field. This is the translation boundary.

**Step 3: Run tests to see what breaks**

```bash
cd /Users/jongamble/repos/public-api-workspace/fordje-public-api
uv run pytest tests/ -x --tb=short 2>&1 | head -60
```

Expected: Failures in tests that reference `trade_group_ids` or `get_trade_groups`.

**Step 4: Commit**

```bash
git add app/schemas/reference_data.py app/schemas/collections.py
git commit -m "refactor: rename trade group schemas to categories"
```

---

### Task 3: Rename backend client function

**Files:**
- Modify: `fordje-public-api/app/services/backend_client.py`

**Step 1: Rename get_trade_groups → get_categories**

In `app/services/backend_client.py`, replace the function (lines 374-392):

```python
async def get_trade_groups(
    page: int = 1,
    page_size: int = 100,
) -> dict[str, Any]:
    """Fetch trade groups.

    Calls: GET /internal/trade-groups
    """
    try:
        response = await BackendClient.get(
            "/internal/trade-groups",
            params={"page": page, "page_size": page_size},
        )
        return response
    except BackendAPIError as e:
        raise HTTPException(
            status_code=e.status_code if e.status_code < 500 else 502,
            detail="Failed to fetch trade groups",
        )
```

With:

```python
async def get_categories(
    page: int = 1,
    page_size: int = 100,
) -> dict[str, Any]:
    """Fetch categories (mapped from backend trade-groups).

    Calls: GET /internal/trade-groups
    """
    try:
        response = await BackendClient.get(
            "/internal/trade-groups",
            params={"page": page, "page_size": page_size},
        )
        return response
    except BackendAPIError as e:
        raise HTTPException(
            status_code=e.status_code if e.status_code < 500 else 502,
            detail="Failed to fetch categories",
        )
```

Note: The internal endpoint path `/internal/trade-groups` stays the same — backend owns that naming.

**Step 2: Update the section comment**

Replace `# Reference data` section comment if desired (optional, it's already clear).

**Step 3: Commit**

```bash
git add app/services/backend_client.py
git commit -m "refactor: rename get_trade_groups to get_categories in backend client"
```

---

### Task 4: Rename routes and update descriptions

**Files:**
- Modify: `fordje-public-api/app/routes/reference_data.py`
- Modify: `fordje-public-api/app/routes/data_points.py`
- Modify: `fordje-public-api/app/routes/collections.py`

**Step 1: Update reference_data.py**

Replace the import:

```python
from app.services.backend_client import get_trade_groups, get_units_of_measure
```

With:

```python
from app.services.backend_client import get_categories, get_units_of_measure
```

Replace the module docstring:

```python
"""Reference data routes — trade groups and units of measure.
```

With:

```python
"""Reference data routes — categories and units of measure.
```

Replace the entire trade-groups route (lines 19-68) with:

```python
@router.get(
    "/categories",
    response_model=BaseResponse,
    summary="List categories",
    description="Get a paginated list of categories. "
    "Categories classify data point types into professional disciplines.",
    responses=AUTHENTICATED_RESPONSES,
    operation_id="list_categories",
)
async def list_categories(
    page: int = Query(default=1, ge=1, description="Page number"),
    page_size: int = Query(
        default=100, ge=1, le=500, description="Items per page (max 500)"
    ),
    api_key: APIKeyContext = Depends(verify_api_key),
) -> BaseResponse:
    """Get categories from backend API with caching."""
    cache_key = f"p{page}_s{page_size}"
    cached = await CacheService.get("categories", cache_key)
    if cached is not None:
        return BaseResponse(
            response_type="success",
            description="Retrieved categories (cached)",
            data=cached,
        )

    response = await get_categories(page=page, page_size=page_size)

    backend_data = response.get("data", [])
    items = backend_data if isinstance(backend_data, list) else []
    total = len(items)
    total_pages = (total + page_size - 1) // page_size if total > 0 else 1

    public_items = [{"id": item.get("id"), "name": item.get("name")} for item in items]

    data = {
        "items": public_items,
        "pagination": Pagination(
            page=page, page_size=page_size, total=total, total_pages=total_pages
        ).model_dump(),
    }
    await CacheService.set(
        "categories", cache_key, data, ttl=settings.cache_ttl_reference
    )

    return BaseResponse(
        response_type="success",
        description=f"Retrieved {len(public_items)} category(ies)",
        data=data,
    )
```

**Step 2: Update data_points.py**

Replace the query parameter (lines 51-55):

```python
    trade_group_ids: Optional[str] = Query(
        default=None,
        description="Comma-separated trade group IDs",
        examples=["1,2"],
    ),
```

With:

```python
    category_ids: Optional[str] = Query(
        default=None,
        description="Comma-separated category IDs",
        examples=["1,2"],
    ),
```

Update the parsing (line 66):

```python
    parsed_group_ids = parse_int_list(trade_group_ids)
```

To:

```python
    parsed_group_ids = parse_int_list(category_ids)
```

Note: The variable `parsed_group_ids` and backend param `group_ids` stay the same — they're internal and match the backend's naming.

**Step 3: Update collections.py**

In the `execute_collection` endpoint description (line 117), replace:

```python
        "Uses the collection's data point type and trade group filters "
```

With:

```python
        "Uses the collection's data point type and category filters "
```

In the `execute_collection` function body (line 154), update the field access:

```python
        group_ids=collection.trade_group_ids,
```

To:

```python
        group_ids=collection.category_ids,
```

**Step 4: Commit**

```bash
git add app/routes/reference_data.py app/routes/data_points.py app/routes/collections.py
git commit -m "refactor: rename trade-groups to categories in routes"
```

---

### Task 5: Update main.py OpenAPI tag

**Files:**
- Modify: `fordje-public-api/app/main.py`

**Step 1: Update the Reference Data tag description (line 71)**

Replace:

```python
            "description": "Trade groups and units of measure",
```

With:

```python
            "description": "Categories and units of measure",
```

**Step 2: Commit**

```bash
git add app/main.py
git commit -m "refactor: update OpenAPI tag description for categories"
```

---

### Task 6: Update settings comment

**Files:**
- Modify: `fordje-public-api/app/config/settings.py`

**Step 1: Update the cache TTL comment (line 42)**

Replace:

```python
    cache_ttl_reference: int = 3600  # 1 hour — trade groups, units of measure
```

With:

```python
    cache_ttl_reference: int = 3600  # 1 hour — categories, units of measure
```

**Step 2: Commit**

```bash
git add app/config/settings.py
git commit -m "refactor: update settings comment for categories"
```

---

### Task 7: Update tests

**Files:**
- Modify: `fordje-public-api/tests/test_reference_data.py`
- Modify: `fordje-public-api/tests/test_data_points_scoped.py`
- Modify: `fordje-public-api/tests/test_collections_execution.py`

**Step 1: Update test_reference_data.py**

Rename class and update all references:

- `TestTradeGroups` → `TestCategories`
- Class docstring: `"""Test categories reference data endpoint."""`
- All `@patch("app.routes.reference_data.get_trade_groups")` → `@patch("app.routes.reference_data.get_categories")`
- All method names: `test_get_trade_groups_*` → `test_get_categories_*`
- All `mock_get_trade_groups` params → `mock_get_categories`
- All URL paths: `"/v1/trade-groups` → `"/v1/categories`
- All `"Retrieved 2 trade group(s)"` → `"Retrieved 2 category(ies)"`
- All `"trade_groups"` cache keys → `"categories"` cache keys
- Docstrings: `trade-groups` → `categories`

**Step 2: Update test_data_points_scoped.py**

- Line 402 docstring: replace `trade_group_ids` with `category_ids`
- Line 444: replace `"&trade_group_ids=1,2"` with `"&category_ids=1,2"`

**Step 3: Update test_collections_execution.py**

- In `make_collection_response()` (line 29): replace `"trade_group_ids": [1]` with `"trade_group_ids": [1]` — **keep as-is** because this simulates the backend response which still uses `trade_group_ids`. The Pydantic alias handles mapping.
- Line 361: `collection_resp["data"]["trade_group_ids"] = [5, 10]` — **keep as-is** (backend response format)
- Line 404: `collection_resp["data"]["trade_group_ids"] = None` — **keep as-is** (backend response format)

**Step 4: Run all tests**

```bash
cd /Users/jongamble/repos/public-api-workspace/fordje-public-api
uv run pytest tests/ -v 2>&1 | tail -30
```

Expected: All tests pass.

**Step 5: Commit**

```bash
git add tests/
git commit -m "test: update tests for trade-groups → categories rename"
```

---

### Task 8: Update spec docs (fordje-public-api)

**Files:**
- Modify: `fordje-public-api/PUBLIC_API_SPEC.md`
- Modify: `fordje-public-api/INTERNAL_ENDPOINTS_REQUIRED.md`
- Modify: `fordje-public-api/CLAUDE.md`

**Step 1: Update PUBLIC_API_SPEC.md**

- Replace `GET /v1/trade-groups` → `GET /v1/categories`
- Replace `trade groups` → `categories` in descriptions
- Replace `trade_group_ids` → `category_ids` in query params
- Replace `trade group IDs` → `category IDs` in collection descriptions

**Step 2: Update INTERNAL_ENDPOINTS_REQUIRED.md**

Add a note to the trade-groups section explaining the public-facing name is "categories":

```markdown
> **Note:** Exposed publicly as `GET /v1/categories`. The public API maps "categories" to backend "trade-groups".
```

**Step 3: Update CLAUDE.md**

- Replace `GET /v1/trade-groups` → `GET /v1/categories` in the API endpoints list
- Replace `Trade groups` → `Categories` in descriptions

**Step 4: Commit**

```bash
git add PUBLIC_API_SPEC.md INTERNAL_ENDPOINTS_REQUIRED.md CLAUDE.md
git commit -m "docs: update spec docs for trade-groups → categories rename"
```

---

### Task 9: Rename and rewrite docs concept page

**Files:**
- Delete: `docs/concepts/trade-groups.mdx`
- Create: `docs/concepts/categories.mdx`

**Step 1: Create categories.mdx with updated content**

Create `docs/concepts/categories.mdx` with all content rewritten: title, descriptions, endpoint references, query parameter names, code examples. Change:

- Title: "Trade Groups" → "Categories"
- Description updated
- All `trade groups` → `categories` in prose
- All `GET /v1/trade-groups` → `GET /v1/categories`
- All `trade_group_ids` → `category_ids` in code examples
- All `"Retrieved 5 trade group(s)"` → `"Retrieved 5 category(ies)"`
- Section headings updated

**Step 2: Delete the old file**

```bash
cd /Users/jongamble/repos/public-api-workspace/docs
git rm concepts/trade-groups.mdx
```

**Step 3: Commit**

```bash
git add concepts/categories.mdx
git commit -m "docs: rename trade-groups concept page to categories"
```

---

### Task 10: Update docs.json navigation

**Files:**
- Modify: `docs/docs.json`

**Step 1: Update Core concepts nav (line 31)**

Replace:

```json
"concepts/trade-groups",
```

With:

```json
"concepts/categories",
```

**Step 2: Update API Reference nav (line 67)**

Replace:

```json
"GET /v1/trade-groups",
```

With:

```json
"GET /v1/categories",
```

**Step 3: Commit**

```bash
git add docs.json
git commit -m "docs: update navigation for categories rename"
```

---

### Task 11: Update remaining docs pages

**Files:**
- Modify: `docs/concepts/index.mdx` (concepts overview accordion)
- Modify: `docs/concepts/data-points.mdx` (filter references)
- Modify: `docs/concepts/collections.mdx` (collection filter references)
- Modify: `docs/quickstart.mdx` (filter mention)
- Modify: `docs/api-reference/introduction.mdx` (endpoint listing)
- Modify: `docs/concepts/rate-limiting.mdx` (cache mention)
- Modify: `docs/AGENTS.md` (terminology)

**Step 1: Update each file**

In each file, replace all instances of:
- `trade groups` → `categories` (in prose)
- `trade group` → `category` (singular)
- `Trade Groups` → `Categories` (headings)
- `Trade Group` → `Category` (headings)
- `GET /v1/trade-groups` → `GET /v1/categories`
- `trade_group_ids` → `category_ids`
- `trade group IDs` → `category IDs`

In `AGENTS.md`, update the terminology entry from:
```
**trade group** = "A grouping category for data point types"
```
To:
```
**category** = "A classification for data point types" (e.g., Electrical, Plumbing, Mechanical)
```

**Step 2: Commit**

```bash
git add concepts/ quickstart.mdx api-reference/introduction.mdx AGENTS.md
git commit -m "docs: update all pages for trade-groups → categories rename"
```

---

### Task 12: Update OpenAPI spec

**Files:**
- Modify: `docs/api-reference/openapi.json`

**Step 1: Update the OpenAPI spec**

- Rename path `/v1/trade-groups` → `/v1/categories`
- Update `operationId`: `list_trade_groups` → `list_categories`
- Update `summary`: "List trade groups" → "List categories"
- Update `description` text
- Update `trade_group_ids` query param → `category_ids` on `/v1/data-points`
- Update descriptions mentioning "trade groups"
- Update `TradeGroupResponse` schema name → `CategoryResponse`

**Step 2: Commit**

```bash
git add api-reference/openapi.json
git commit -m "docs: update OpenAPI spec for categories rename"
```

---

### Task 13: Run full test suite and verify

**Step 1: Run all tests**

```bash
cd /Users/jongamble/repos/public-api-workspace/fordje-public-api
uv run pytest tests/ -v --tb=short
```

Expected: All tests pass.

**Step 2: Run linting**

```bash
uv run black --check . && uv run isort --check . && uv run flake8 app/
```

Expected: No formatting issues.

**Step 3: Check docs for broken links**

```bash
cd /Users/jongamble/repos/public-api-workspace/docs
mint broken-links
```

Expected: No broken links.

**Step 4: Commit any fixes if needed**

---

### Task 14: Push branches and open PRs

**Step 1: Push fordje-public-api branch**

```bash
cd /Users/jongamble/repos/public-api-workspace/fordje-public-api
git push -u origin feature/rename-trade-groups-to-categories
```

**Step 2: Open PR for fordje-public-api**

```bash
gh pr create --title "Rename trade groups to categories" --body "$(cat <<'EOF'
## Summary
- Renames all public-facing references from "trade groups" to "categories"
- API endpoint: `/v1/trade-groups` → `/v1/categories`
- Query param: `trade_group_ids` → `category_ids`
- Backend naming unchanged — mapping happens at the gateway boundary

## Test plan
- [ ] All existing tests updated and passing
- [ ] Lint checks pass
- [ ] Manual test: `GET /v1/categories` returns expected response
- [ ] Manual test: `GET /v1/data-points?category_ids=1` filters correctly

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

**Step 3: Push docs branch**

```bash
cd /Users/jongamble/repos/public-api-workspace/docs
git push -u origin feature/rename-trade-groups-to-categories
```

**Step 4: Open PR for docs**

```bash
gh pr create --title "Rename trade groups to categories in docs" --body "$(cat <<'EOF'
## Summary
- Renames all documentation references from "trade groups" to "categories"
- Concept page renamed from `trade-groups.mdx` to `categories.mdx`
- Navigation, API reference, and OpenAPI spec updated
- All cross-references updated across concept pages, quickstart, and guides

## Test plan
- [ ] `mint broken-links` passes
- [ ] `mint dev` previews correctly
- [ ] All concept page links work

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```
