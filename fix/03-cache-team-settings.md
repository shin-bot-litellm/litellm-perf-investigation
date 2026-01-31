# Fix #3: Cache Team Router Settings

**Status:** ⚠️ Proposed  
**PR:** Not yet created

## Problem

Every request with `team_id` queries DB for team router_settings, even when:
- Team has no router_settings configured
- Same team was queried milliseconds ago

This adds 5-20ms latency per request.

## Proposed Solution

Add TTL cache for team router_settings:

```python
from cachetools import TTLCache

# Cache team settings for 60 seconds
_team_router_settings_cache: TTLCache = TTLCache(maxsize=1000, ttl=60)

async def _get_hierarchical_router_settings(self, user_api_key_dict, prisma_client):
    if prisma_client is None:
        return None
    
    # 1. Key-level (unchanged)
    key_settings = getattr(user_api_key_dict, "router_settings", None)
    if key_settings and isinstance(key_settings, dict) and key_settings:
        return key_settings
    
    # 2. Team-level WITH CACHING
    if user_api_key_dict and user_api_key_dict.team_id:
        cache_key = f"team:{user_api_key_dict.team_id}"
        
        if cache_key in _team_router_settings_cache:
            cached = _team_router_settings_cache[cache_key]
            return cached if cached else None  # Return None for empty cache
        
        # Cache miss - query DB
        try:
            team_obj = await prisma_client.db.litellm_teamtable.find_unique(
                where={"team_id": user_api_key_dict.team_id}
            )
            settings = None
            if team_obj:
                raw = getattr(team_obj, "router_settings", None)
                if raw and isinstance(raw, dict) and raw:
                    settings = raw
            
            # Cache result (including empty/None)
            _team_router_settings_cache[cache_key] = settings or {}
            return settings
        except Exception:
            pass
    
    # 3. Skip global (per Fix #2)
    return None
```

## Cache Invalidation

Add invalidation when team settings are updated:

```python
# In team update endpoint
async def update_team(...):
    # ... update team ...
    _team_router_settings_cache.pop(f"team:{team_id}", None)
```

## Impact

- Eliminates repeated DB queries for same team
- 60-second TTL balances freshness vs performance
- Empty results cached to prevent query storms
