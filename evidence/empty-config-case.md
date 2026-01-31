# Empty Config Case Investigation

## Summary

Users with `STORE_MODEL_IN_DB=True` and **empty** `LiteLLM_Config` and `LiteLLM_TeamTable` tables still experience performance regression. This document investigates why.

## Key Finding: Uncached Per-Request DB Queries

The `_get_hierarchical_router_settings` function (introduced in v1.80.13+) makes **uncached database queries on EVERY request**, even when tables are completely empty.

### The Problematic Code Path

**File:** `litellm/proxy/common_request_processing.py` (lines 603-618)

```python
# Apply hierarchical router_settings (Key > Team > Global)
if llm_router is not None and proxy_config is not None:
    from litellm.proxy.proxy_server import prisma_client

    router_settings = await proxy_config._get_hierarchical_router_settings(
        user_api_key_dict=user_api_key_dict,
        prisma_client=prisma_client,
    )

    # If router_settings found (from key, team, or global), apply them
    if router_settings is not None and router_settings:
        model_list = llm_router.get_model_list()
        if model_list is not None:
            user_config = {"model_list": model_list, **router_settings}
            self.data["user_config"] = user_config
```

### The DB Queries Made Per Request

**File:** `litellm/proxy/proxy_server.py` (lines 3382-3476)

```python
async def _get_hierarchical_router_settings(
    self,
    user_api_key_dict: Optional["UserAPIKeyAuth"],
    prisma_client: Optional[PrismaClient],
) -> Optional[dict]:
    """
    Get router_settings in priority order: Key > Team > Global
    """
    if prisma_client is None:
        return None

    # ... key-level check (from already-loaded user_api_key_dict) ...

    # Query 1: Team lookup (if team_id exists)
    if user_api_key_dict is not None and user_api_key_dict.team_id is not None:
        try:
            team_obj = await prisma_client.db.litellm_teamtable.find_unique(
                where={"team_id": user_api_key_dict.team_id}
            )
            # ... process team_router_settings ...
        except Exception:
            pass

    # Query 2: Global router_settings (ALWAYS runs)
    try:
        db_router_settings = await prisma_client.db.litellm_config.find_first(
            where={"param_name": "router_settings"}
        )
        if (
            db_router_settings is not None
            and isinstance(db_router_settings.param_value, dict)
            and db_router_settings.param_value
        ):
            return db_router_settings.param_value
    except Exception:
        pass

    return None
```

## Why Empty Tables Still Cause Slowdown

Even with empty tables:

1. **Query Overhead**: The DB round-trip time for `find_first()` and `find_unique()` still applies
2. **No Caching**: Router settings queries are NOT cached - they hit the database on every single request
3. **Multiple Queries**: Up to 2 queries per request (team lookup + global settings)
4. **Network Latency**: For remote Postgres/Redis, network latency compounds

### Cost Per Request

| Operation | Typical Latency | Cached? |
|-----------|----------------|---------|
| Team lookup (`find_unique`) | 1-5ms | ❌ No |
| Global router_settings (`find_first`) | 1-5ms | ❌ No |
| **Total Added Per Request** | **2-10ms** | |

At 100 req/s, this adds **200-1000ms** of cumulative latency per second.

## When This Was Introduced

- **Commit:** `51759424a6` ("Key and Team Routing Setting")
- **Author:** yuneng-jiang
- **Date:** Wed Jan 7 17:17:30 2026
- **First Appears In:** v1.80.13.rc.1
- **Present In:** v1.81.0-nightly

### Files Changed

```
litellm/proxy/_types.py                            |  1 +
litellm/proxy/common_request_processing.py         | 23 ++++++
litellm/proxy/proxy_server.py                      | 78 ++++++++++++++++++
tests/...                                          | 170 ++++++++++++++++
```

## What STORE_MODEL_IN_DB=True Triggers

When `STORE_MODEL_IN_DB=True` is set:

1. **At Startup** (`proxy_server.py` lines 5024-5028):
   ```python
   store_model_in_db = (
       get_secret_bool("STORE_MODEL_IN_DB", store_model_in_db) or store_model_in_db
   )
   ```

2. **Config Loading** (`proxy_server.py` lines 2137-2141):
   ```python
   if prisma_client is not None and store_model_in_db is True:
       config = await self._update_config_from_db(
           config=config,
           prisma_client=prisma_client,
           store_model_in_db=store_model_in_db,
       )
   ```

3. **Per-Request** (the problematic path):
   - The `_get_hierarchical_router_settings` function runs regardless of table contents
   - It doesn't check if `store_model_in_db` is True - it checks if `prisma_client` exists

## Evidence This Affects Users with Empty Config

User superpoussin22 in issue #19921:
> "on prod everything is empty on my side and I can see the issue"

Their setup:
- `STORE_MODEL_IN_DB=True`
- 90% of models in config files
- Empty `LiteLLM_Config` and `LiteLLM_TeamTable`

## Proposed Fix

1. **Add Caching for Router Settings**: Cache the global router_settings lookup with a reasonable TTL (e.g., 60s)
   ```python
   # Use DualCache for router_settings
   cache_key = "router_settings_global"
   cached = await user_api_key_cache.async_get_cache(key=cache_key)
   if cached is not None:
       return cached
   # ... do DB lookup ...
   await user_api_key_cache.async_set_cache(key=cache_key, value=result, ttl=60)
   ```

2. **Early Exit When Tables Are Empty**: Check a cached flag indicating tables are empty

3. **Configuration Option**: Add `disable_hierarchical_router_settings` flag for users who don't need this feature

## Related Files

- `litellm/proxy/common_request_processing.py` - Hot path entry point
- `litellm/proxy/proxy_server.py` - `_get_hierarchical_router_settings` implementation
- `litellm/proxy/auth/auth_checks.py` - Team table lookups (also no caching for router_settings)

## Comparison with Other DB Lookups

Other lookups in the auth path ARE cached via `DualCache`:
- Team member info (line 1016-1032 in `user_api_key_auth.py`)
- User budget info (line 533-550 in `auth_checks.py`)
- User objects (line 670-709 in `auth_checks.py`)

But router_settings queries are NOT cached.
