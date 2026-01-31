# Fix: Cache Router Settings Queries

## Problem

`_get_hierarchical_router_settings` makes uncached DB queries on every request:
- `litellm_teamtable.find_unique()` for team router_settings
- `litellm_config.find_first()` for global router_settings

## Proposed Solution

Add caching to router_settings queries using the existing `DualCache` infrastructure.

### Option 1: Cache Global Router Settings (Recommended)

```python
# In proxy_server.py, _get_hierarchical_router_settings method

async def _get_hierarchical_router_settings(
    self,
    user_api_key_dict: Optional["UserAPIKeyAuth"],
    prisma_client: Optional[PrismaClient],
) -> Optional[dict]:
    if prisma_client is None:
        return None
    
    # Import cache
    from litellm.proxy.proxy_server import user_api_key_cache
    
    # 1. Try key-level router_settings (already loaded, no change needed)
    if user_api_key_dict is not None:
        key_router_settings_value = getattr(user_api_key_dict, "router_settings", None)
        if key_router_settings_value:
            # ... existing parsing logic ...
            if parsed_settings:
                return parsed_settings

    # 2. Try team-level router_settings (ADD CACHING)
    if user_api_key_dict is not None and user_api_key_dict.team_id is not None:
        team_cache_key = f"team_router_settings:{user_api_key_dict.team_id}"
        
        # Check cache first
        cached_team_settings = await user_api_key_cache.async_get_cache(key=team_cache_key)
        if cached_team_settings is not None:
            if cached_team_settings == "__NONE__":
                pass  # Team has no router_settings, skip to global
            else:
                return cached_team_settings
        else:
            # Cache miss - query DB
            try:
                team_obj = await prisma_client.db.litellm_teamtable.find_unique(
                    where={"team_id": user_api_key_dict.team_id}
                )
                if team_obj is not None:
                    team_router_settings = getattr(team_obj, "router_settings", None)
                    if team_router_settings:
                        # Parse and cache
                        parsed = self._parse_router_settings(team_router_settings)
                        if parsed:
                            await user_api_key_cache.async_set_cache(
                                key=team_cache_key, value=parsed, ttl=60
                            )
                            return parsed
                # No team settings - cache the miss
                await user_api_key_cache.async_set_cache(
                    key=team_cache_key, value="__NONE__", ttl=60
                )
            except Exception:
                pass

    # 3. Try global router_settings (ADD CACHING)
    global_cache_key = "router_settings:global"
    
    # Check cache first
    cached_global_settings = await user_api_key_cache.async_get_cache(key=global_cache_key)
    if cached_global_settings is not None:
        if cached_global_settings == "__NONE__":
            return None
        return cached_global_settings
    
    # Cache miss - query DB
    try:
        db_router_settings = await prisma_client.db.litellm_config.find_first(
            where={"param_name": "router_settings"}
        )
        if db_router_settings is not None and db_router_settings.param_value:
            await user_api_key_cache.async_set_cache(
                key=global_cache_key, value=db_router_settings.param_value, ttl=60
            )
            return db_router_settings.param_value
        # No global settings - cache the miss
        await user_api_key_cache.async_set_cache(
            key=global_cache_key, value="__NONE__", ttl=60
        )
    except Exception:
        pass

    return None
```

### Option 2: Short-Circuit When No Settings Exist

Add a startup check to determine if any router_settings exist:

```python
# In proxy_server.py startup

async def _check_router_settings_exist():
    """Check if any router_settings exist in DB at startup"""
    global prisma_client, _router_settings_exist_in_db
    
    if prisma_client is None:
        _router_settings_exist_in_db = False
        return
    
    try:
        # Check for any team with router_settings
        team_with_settings = await prisma_client.db.litellm_teamtable.find_first(
            where={"router_settings": {"not": None}}
        )
        
        # Check for global router_settings
        global_settings = await prisma_client.db.litellm_config.find_first(
            where={"param_name": "router_settings"}
        )
        
        _router_settings_exist_in_db = (
            team_with_settings is not None or 
            global_settings is not None
        )
    except Exception:
        _router_settings_exist_in_db = True  # Fail safe

# In _get_hierarchical_router_settings
async def _get_hierarchical_router_settings(...):
    global _router_settings_exist_in_db
    
    # Quick exit if we know no settings exist
    if not _router_settings_exist_in_db:
        # Still check key-level (no DB hit)
        if user_api_key_dict:
            key_settings = getattr(user_api_key_dict, "router_settings", None)
            if key_settings:
                return self._parse_router_settings(key_settings)
        return None
    
    # ... existing logic with caching ...
```

### Option 3: Configuration Flag to Disable

For users who don't use hierarchical router_settings:

```yaml
# config.yaml
general_settings:
  disable_hierarchical_router_settings: true
```

```python
# In common_request_processing.py
if (
    llm_router is not None 
    and proxy_config is not None
    and not general_settings.get("disable_hierarchical_router_settings", False)
):
    router_settings = await proxy_config._get_hierarchical_router_settings(...)
```

## Expected Impact

| Approach | Cache Hit Latency | Cache Miss Latency | Implementation |
|----------|------------------|-------------------|----------------|
| Current (no cache) | N/A | 2-10ms per request | - |
| Option 1 (DualCache) | <1ms | 2-10ms (first time) | Medium |
| Option 2 (startup check) | 0ms | 2-10ms (first time) | Low |
| Option 3 (config flag) | 0ms | 0ms | Lowest |

## Recommended Approach

Implement Option 1 (caching) as the primary fix, with Option 3 (config flag) as an escape hatch for users who want zero overhead.

## Cache Invalidation

The cache should be invalidated when:
- Team router_settings are updated
- Global router_settings are updated
- A new team is created with router_settings

Add invalidation hooks in the relevant CRUD endpoints:
- `POST /team/update`
- `POST /config/update`
- `PATCH /team/{team_id}`
